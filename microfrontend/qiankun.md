# qiankun

分析qiankun加载微应用的过程

## Table of content

- [加载微应用的流程](#加载微应用的流程)
- [loadMicroApp](#loadmicroapp)
- [loadApp](#loadapp)
  - [createSandboxContainer](#createsandboxcontainer)
  - [patchAtBootstrapping/patchAtMounting](#patchatbootstrapping或patchatmounting)
    - [patchHTMLDynamicAppendPrototypeFunctions](#patchhtmldynamicappendprototypefunctions)
    - [getOverwrittenAppendChildOrInsertBefore](#getoverwrittenappendchildorinsertbefore)

## 加载微应用的流程

- boostrap阶段，只会执行一次
- mount阶段，可能会执行多次，例如微应用多次加载/卸载

## loadMicroApp

loadMicroApp函数是用来手动加载微应用的一个函数，主要作用为

- 缓存loadApp函数的执行结果，可以理解为Bootstrap阶段，执行微应用的入口代码
- 调用single-spa中的start函数，让single-spa开始劫持路由事件

```typescript
export function loadMicroApp<T extends ObjectType>(
  app: LoadableApp<T>,
  configuration?: FrameworkConfiguration & { autoStart?: boolean },
  lifeCycles?: FrameworkLifeCycles<T>,
): MicroApp {
  const { props, name } = app;

  const container = 'container' in app ? app.container : undefined;
  // Must compute the container xpath at beginning to keep it consist around app running
  // If we compute it every time, the container dom structure most probably been changed and result in a different xpath value
  const containerXPath = getContainerXPath(container);
  const appContainerXPathKey = `${name}-${containerXPath}`;

  let microApp: MicroApp;
  // 缓存loadApp函数的执行结果
  const memorizedLoadingFn = async (): Promise<ParcelConfigObject> => {
	// ...
    const parcelConfigObjectGetterPromise = loadApp(app, userConfiguration, lifeCycles);

    if (container) {
      if ($$cacheLifecycleByAppName) {
        appConfigPromiseGetterMap.set(name, parcelConfigObjectGetterPromise);
      } else if (containerXPath) appConfigPromiseGetterMap.set(appContainerXPathKey, parcelConfigObjectGetterPromise);
    }
    
    return (await parcelConfigObjectGetterPromise)(container);
  };

  if (!started && configuration?.autoStart !== false) {
    startSingleSpa({ urlRerouteOnly: frameworkConfiguration.urlRerouteOnly ?? defaultUrlRerouteOnly });
  }

  // 调用single-spa的mountRootParcel函数，执行微应用的生命周期函数
  microApp = mountRootParcel(memorizedLoadingFn, { domElement: document.createElement('div'), ...props });
 // ...

  return microApp;
}
```

## loadApp

loadApp函数主要做了一下事情，

- 通过import-html-entry分析微应用的脚本及样式表信息
- 创建微应用全局对象沙箱
- 通过import-html-entry提供的execScripts函数传入全局对象沙箱获取微应用的生命周期函数
- 返回创建parcelConfig的函数

```typescript
export async function loadApp<T extends ObjectType>(
  app: LoadableApp<T>,
  configuration: FrameworkConfiguration = {},
  lifeCycles?: FrameworkLifeCycles<T>,
): Promise<ParcelConfigObjectGetter> {
  // get the entry html content and script executor
  const { template, execScripts, assetPublicPath, getExternalScripts } = await importEntry(entry, importEntryOpts);
  // trigger external scripts loading to make sure all assets are ready before execScripts calling
  await getExternalScripts();
  
  // ...
  let sandboxContainer;
  if (sandbox) {
    sandboxContainer = createSandboxContainer(
      appInstanceId,
      // FIXME should use a strict sandbox logic while remount, see https://github.com/umijs/qiankun/issues/518
      initialAppWrapperGetter,
      scopedCSS,
      useLooseSandbox,
      excludeAssetFilter,
      global,
      speedySandbox,
    );
    // 用沙箱的代理对象作为接下来使用的全局对象
    global = sandboxContainer.instance.proxy as typeof window;
    mountSandbox = sandboxContainer.mount;
    unmountSandbox = sandboxContainer.unmount;
  }
    
  //...
    
  // get the lifecycle hooks from module exports
  const scriptExports: any = await execScripts(global, sandbox && !useLooseSandbox, {
    scopedGlobalVariables: speedySandbox ? cachedGlobals : [],
  });
  const { bootstrap, mount, unmount, update } = getLifecyclesFromExports(
    scriptExports,
    appName,
    global,
    sandboxContainer?.instance?.latestSetProp,
  );
    
   const parcelConfigGetter: ParcelConfigObjectGetter = (remountContainer = initialContainer) => {
    let appWrapperElement: HTMLElement | null;
    let appWrapperGetter: ReturnType<typeof getAppWrapperGetter>;

    const parcelConfig: ParcelConfigObject = {
      name: appInstanceId,
      bootstrap,
      mount: [
       	// mount钩子
      ],
      unmount: [
       // unmount钩子
      ],
    };
    // ...
    return parcelConfig;
  };

  return parcelConfigGetter;
}
```

### createSandboxContainer

createSandboxContainer函数主要做了以下事情，

- 创建沙箱对象
- 在boostrap阶段猴子补丁DOM注入方法，获取boostrap阶段的副作用，方便在mount阶段时rebuild初始化阶段的副作用
- 在mount阶段也同理

```typescript
export function createSandboxContainer(
  appName: string,
  elementGetter: () => HTMLElement | ShadowRoot,
  scopedCSS: boolean,
  useLooseSandbox?: boolean,
  excludeAssetFilter?: (url: string) => boolean,
  globalContext?: typeof window,
  speedySandBox?: boolean,
) {
  let sandbox: SandBox;
  if (window.Proxy) {
    sandbox = useLooseSandbox ? new LegacySandbox(appName, globalContext) : new ProxySandbox(appName, globalContext);
  } else {
    sandbox = new SnapshotSandbox(appName);
  }

  // some side effect could be be invoked while bootstrapping, such as dynamic stylesheet injection with style-loader, especially during the development phase
  const bootstrappingFreers = patchAtBootstrapping(
    appName,
    elementGetter,
    sandbox,
    scopedCSS,
    excludeAssetFilter,
    speedySandBox,
  );
  // mounting freers are one-off and should be re-init at every mounting time
  let mountingFreers: Freer[] = [];

  let sideEffectsRebuilders: Rebuilder[] = [];

  return {
    instance: sandbox,

    /**
     * 沙箱被 mount
     * 可能是从 bootstrap 状态进入的 mount
     * 也可能是从 unmount 之后再次唤醒进入 mount
     */
    async mount() {
      /* ------------------------------------------ 因为有上下文依赖（window），以下代码执行顺序不能变 ------------------------------------------ */

      /* ------------------------------------------ 1. 启动/恢复 沙箱------------------------------------------ */
      sandbox.active();

      const sideEffectsRebuildersAtBootstrapping = sideEffectsRebuilders.slice(0, bootstrappingFreers.length);
      const sideEffectsRebuildersAtMounting = sideEffectsRebuilders.slice(bootstrappingFreers.length);

      // must rebuild the side effects which added at bootstrapping firstly to recovery to nature state
      if (sideEffectsRebuildersAtBootstrapping.length) {
        sideEffectsRebuildersAtBootstrapping.forEach((rebuild) => rebuild());
      }

      /* ------------------------------------------ 2. 开启全局变量补丁 ------------------------------------------*/
      // render 沙箱启动时开始劫持各类全局监听，尽量不要在应用初始化阶段有 事件监听/定时器 等副作用
      mountingFreers = patchAtMounting(appName, elementGetter, sandbox, scopedCSS, excludeAssetFilter, speedySandBox);

      /* ------------------------------------------ 3. 重置一些初始化时的副作用 ------------------------------------------*/
      // 存在 rebuilder 则表明有些副作用需要重建
      if (sideEffectsRebuildersAtMounting.length) {
        sideEffectsRebuildersAtMounting.forEach((rebuild) => rebuild());
      }

      // clean up rebuilders
      sideEffectsRebuilders = [];
    },

    /**
     * 恢复 global 状态，使其能回到应用加载之前的状态
     */
    async unmount() {
      // record the rebuilders of window side effects (event listeners or timers)
      // note that the frees of mounting phase are one-off as it will be re-init at next mounting
      sideEffectsRebuilders = [...bootstrappingFreers, ...mountingFreers].map((free) => free());

      sandbox.inactive();
    },
  };
}
```

### patchAtBootstrapping或patchAtMounting

在boostrap/mount阶段猴子补丁DOM注入方法，做了以下事情

- 劫持动态注入的style样式表，注入到qiankun自定义的head元素中
- 劫持动态注入的script脚本，将脚本运行在沙箱对象环境中
- 返回需要清理函数以及需要重构的操作

这里看StrictSandbox的实现

```typescript
export function patchStrictSandbox(
  appName: string,
  appWrapperGetter: () => HTMLElement | ShadowRoot,
  proxy: Window,
  mounting = true,
  scopedCSS = false,
  excludeAssetFilter?: CallableFunction,
  speedySandbox = false,
): Freer {
  // ...
  // all dynamic style sheets are stored in proxy container
  const { dynamicStyleSheetElements } = containerConfig;

  const unpatchDocumentCreate = patchDocumentCreateElement();

  const unpatchDynamicAppendPrototypeFunctions = patchHTMLDynamicAppendPrototypeFunctions(
    (element) => elementAttachContainerConfigMap.has(element),
    (element) => elementAttachContainerConfigMap.get(element)!,
  );
    
  // ...

  return function free() {
    // ...
    // release the overwritten prototype after all the micro apps unmounted
    if (isAllAppsUnmounted()) {
      unpatchDynamicAppendPrototypeFunctions();
      unpatchDocumentCreate();
    }

    recordStyledComponentsCSSRules(dynamicStyleSheetElements);

    // As now the sub app content all wrapped with a special id container,
    // the dynamic style sheet would be removed automatically while unmoutting

    return function rebuild() {
      rebuildCSSRules(dynamicStyleSheetElements, (stylesheetElement) => {
        const appWrapper = appWrapperGetter();
        if (!appWrapper.contains(stylesheetElement)) {
          const mountDom =
            stylesheetElement[styleElementTargetSymbol] === 'head' ? getAppWrapperHeadElement(appWrapper) : appWrapper;
          rawHeadAppendChild.call(mountDom, stylesheetElement);
          return true;
        }

        return false;
      });
    };
  };
}

```

#### patchHTMLDynamicAppendPrototypeFunctions

猴子补丁DOM注入方法

```typescript
export function patchHTMLDynamicAppendPrototypeFunctions(
  isInvokedByMicroApp: (element: HTMLElement) => boolean,
  containerConfigGetter: (element: HTMLElement) => ContainerConfig,
) {
  // Just overwrite it while it have not been overwrite
  if (
    HTMLHeadElement.prototype.appendChild === rawHeadAppendChild &&
    HTMLBodyElement.prototype.appendChild === rawBodyAppendChild &&
    HTMLHeadElement.prototype.insertBefore === rawHeadInsertBefore
  ) {
    HTMLHeadElement.prototype.appendChild = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawHeadAppendChild,
      containerConfigGetter,
      isInvokedByMicroApp,
      target: 'head',
    }) as typeof rawHeadAppendChild;
    HTMLBodyElement.prototype.appendChild = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawBodyAppendChild,
      containerConfigGetter,
      isInvokedByMicroApp,
      target: 'body',
    }) as typeof rawBodyAppendChild;

    HTMLHeadElement.prototype.insertBefore = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawHeadInsertBefore as any,
      containerConfigGetter,
      isInvokedByMicroApp,
      target: 'head',
    }) as typeof rawHeadInsertBefore;
  }

  // Just overwrite it while it have not been overwrite
  if (
    HTMLHeadElement.prototype.removeChild === rawHeadRemoveChild &&
    HTMLBodyElement.prototype.removeChild === rawBodyRemoveChild
  ) {
    HTMLHeadElement.prototype.removeChild = getNewRemoveChild(rawHeadRemoveChild, containerConfigGetter, 'head');
    HTMLBodyElement.prototype.removeChild = getNewRemoveChild(rawBodyRemoveChild, containerConfigGetter, 'body');
  }

  return function unpatch() {
	// ...
  };
}
```

#### getOverwrittenAppendChildOrInsertBefore

通过判断element的类型，处理不同操作

```typescript
function getOverwrittenAppendChildOrInsertBefore(opts: {
  rawDOMAppendOrInsertBefore: <T extends Node>(newChild: T, refChild?: Node | null) => T;
  isInvokedByMicroApp: (element: HTMLElement) => boolean;
  containerConfigGetter: (element: HTMLElement) => ContainerConfig;
  target: DynamicDomMutationTarget;
}) {
  return function appendChildOrInsertBefore<T extends Node>(
    this: HTMLHeadElement | HTMLBodyElement,
    newChild: T,
    refChild: Node | null = null,
  ) {
    let element = newChild as any;
    const { rawDOMAppendOrInsertBefore, isInvokedByMicroApp, containerConfigGetter, target = 'body' } = opts;
    if (!isHijackingTag(element.tagName) || !isInvokedByMicroApp(element)) {
      return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
    }

    if (element.tagName) {
      const containerConfig = containerConfigGetter(element);
      const {
        appName,
        appWrapperGetter,
        proxy,
        strictGlobal,
        speedySandbox,
        dynamicStyleSheetElements,
        scopedCSS,
        excludeAssetFilter,
      } = containerConfig;

      switch (element.tagName) {
        case LINK_TAG_NAME:
        case STYLE_TAG_NAME: {
          let stylesheetElement: HTMLLinkElement | HTMLStyleElement = newChild as any;
          const { href } = stylesheetElement as HTMLLinkElement;
          if (excludeAssetFilter && href && excludeAssetFilter(href)) {
            return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
          }

          Object.defineProperty(stylesheetElement, styleElementTargetSymbol, {
            value: target,
            writable: true,
            configurable: true,
          });

          const appWrapper = appWrapperGetter();

          if (scopedCSS) {
            // exclude link elements like <link rel="icon" href="favicon.ico">
            const linkElementUsingStylesheet =
              element.tagName?.toUpperCase() === LINK_TAG_NAME &&
              (element as HTMLLinkElement).rel === 'stylesheet' &&
              (element as HTMLLinkElement).href;
            if (linkElementUsingStylesheet) {
              const fetch =
                typeof frameworkConfiguration.fetch === 'function'
                  ? frameworkConfiguration.fetch
                  : frameworkConfiguration.fetch?.fn;
              stylesheetElement = convertLinkAsStyle(
                element,
                (styleElement) => css.process(appWrapper, styleElement, appName),
                fetch,
              );
              dynamicLinkAttachedInlineStyleMap.set(element, stylesheetElement);
            } else {
              css.process(appWrapper, stylesheetElement, appName);
            }
          }

          const mountDOM = target === 'head' ? getAppWrapperHeadElement(appWrapper) : appWrapper;

          dynamicStyleSheetElements.push(stylesheetElement);
          const referenceNode = mountDOM.contains(refChild) ? refChild : null;
          return rawDOMAppendOrInsertBefore.call(mountDOM, stylesheetElement, referenceNode);
        }

        case SCRIPT_TAG_NAME: {
          const { src, text } = element as HTMLScriptElement;
          // some script like jsonp maybe not support cors which should't use execScripts
          if ((excludeAssetFilter && src && excludeAssetFilter(src)) || !isExecutableScriptType(element)) {
            return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
          }

          const appWrapper = appWrapperGetter();
          const mountDOM = target === 'head' ? getAppWrapperHeadElement(appWrapper) : appWrapper;

          const { fetch } = frameworkConfiguration;
          const referenceNode = mountDOM.contains(refChild) ? refChild : null;

          const scopedGlobalVariables = speedySandbox ? cachedGlobals : [];

          if (src) {
            let isRedfinedCurrentScript = false;
            execScripts(null, [src], proxy, {
              fetch,
              strictGlobal,
              scopedGlobalVariables,
			  // ...
            });

            const dynamicScriptCommentElement = document.createComment(`dynamic script ${src} replaced by qiankun`);
            dynamicScriptAttachedCommentMap.set(element, dynamicScriptCommentElement);
            return rawDOMAppendOrInsertBefore.call(mountDOM, dynamicScriptCommentElement, referenceNode);
          }

          // inline script never trigger the onload and onerror event
          execScripts(null, [`<script>${text}</script>`], proxy, { strictGlobal, scopedGlobalVariables });
          const dynamicInlineScriptCommentElement = document.createComment('dynamic inline script replaced by qiankun');
          dynamicScriptAttachedCommentMap.set(element, dynamicInlineScriptCommentElement);
          return rawDOMAppendOrInsertBefore.call(mountDOM, dynamicInlineScriptCommentElement, referenceNode);
        }

        default:
          break;
      }
    }

    return rawDOMAppendOrInsertBefore.call(this, element, refChild);
  };
}
```

