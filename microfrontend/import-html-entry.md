# import-html-entry

import-html-entry流程分析

## Table of contents

- [importEntry](#importentry)
- [importHTML](#importhtml)
- [processTpl](#processtpl)
- [getEmbedHTML](#getembedhtml)
  - [getEmbedHTML函数](#getembedhtml函数)
  - [getExternalStyleSheets](#getexternalstylesheets)
  - [getEmbedHTML返回值](#getembedhtml返回值)
- [execScripts](#execScripts)
  - [evalCode](#evalcode)
  - [getExcutableScript](#getexcutablescript)
  - [execScripts函数](#execscripts函数)
  - [exec](#exec)

## importEntry

entry为string时进入importHTML函数

```javascript
export function importEntry(entry, opts = {}) {
	const { fetch = defaultFetch, getTemplate = defaultGetTemplate, postProcessTemplate } = opts;
	const getPublicPath = opts.getPublicPath || opts.getDomain || defaultGetPublicPath;
    
	// html entry
	if (typeof entry === 'string') {
		return importHTML(entry, {
			fetch,
			getPublicPath,
			getTemplate,
			postProcessTemplate,
		});
	}
	// ...
}

```

## importHTML

下载html，处理模板

```javascript
export default function importHTML(url, opts = {}) {
	// ...
	return embedHTMLCache[url] || (embedHTMLCache[url] = fetch(url)
		.then(response => readResAsString(response, autoDecodeResponse))
		.then(html => {
			const assetPublicPath = getPublicPath(url);
			const { template, scripts, entry, styles } = processTpl(getTemplate(html), assetPublicPath, postProcessTemplate);

			return getEmbedHTML(...);
}
```

## processTpl

分析html模板，将styles和scripts的内容提取成数据，被提取的styles和scripts会被替换成注释

- styles处理方式
  - link或style标签上有ignore属性会忽略内容
  - link有href属性，将会将href的值存放在styles数组
  - style标签会将style元素存放在styles数据
- scripts处理方式
  - script标签上有ignore属性会忽略内容
  - script有href属性，判断是否有async属性
    - 有async属性，scripts数组存放{ async: true, src: '...' }
    - 没有async属性，scripts数组存放href属性
  - 内联script会将script元素存放在scripts数组中
- 设置entry的值
  - 如果script标签上有entry属性，则会将改script设置为entry
  - 如果script标签上没有entry属性，则会将scripts最后一个元素作为entry

## getEmbedHTML

主要将template中被注释替换的样式表，替换回模板。

- 外链link内容下载下来，替换模板中的注释；
- 内联样式直接将样式内容替换

### getEmbedHTML函数

```javascript
/**
 * convert external css link to inline style for performance optimization
 */
function getEmbedHTML(template, styles, opts = {}) {
	const { fetch = defaultFetch } = opts;
	let embedHTML = template;

	return getExternalStyleSheets(styles, fetch)
		.then(styleSheets => {
			embedHTML = styles.reduce((html, styleSrc, i) => {
				html = html.replace(genLinkReplaceSymbol(styleSrc), isInlineCode(styleSrc) ? `${styleSrc}` : `<style>/* ${styleSrc} */${styleSheets[i]}</style>`);
				return html;
			}, embedHTML);
			return embedHTML;
		});
}
```

### getExternalStyleSheets

```javascript
export function getExternalStyleSheets(styles, fetch = defaultFetch) {
	return Promise.all(styles.map(styleLink => {
			if (isInlineCode(styleLink)) {
				// if it is inline style
				return getInlineCode(styleLink);
			} else {
				// external styles
				return styleCache[styleLink] ||
					(styleCache[styleLink] = fetch(styleLink).then(response => response.text()));
			}
		},
	));
}
```

### getEmbedHTML返回值

向外暴露得到的信息。

- 注入内联样式后的template模板
- assetPublicPath
- getExternalScripts，返回一个获取全部脚本的Promise
- getExternalStyleSheets，返回一个获取全部样式的Promise
- 返回一个execScripts函数，用来执行脚本

```javascript
export default function importHTML(url, opts = {}) {
	// ...

	return embedHTMLCache[url] || (embedHTMLCache[url] = fetch(url)
		.then(response => readResAsString(response, autoDecodeResponse))
		.then(html => {
			// ...
			return getEmbedHTML(template, styles, { fetch }).then(embedHTML => ({
				template: embedHTML,
				assetPublicPath,
				getExternalScripts: () => getExternalScripts(scripts, fetch),
				getExternalStyleSheets: () => getExternalStyleSheets(styles, fetch),
				execScripts: (proxy, strictGlobal, opts = {}) => {
					if (!scripts.length) {
						return Promise.resolve();
					}
					return execScripts(entry, scripts, proxy, {
						fetch,
						strictGlobal,
						...opts,
					});
				},
			}));
		}));
}
```

## execScripts

执行template中的脚本，最后将entry的执行结果返回。

### evalCode

执行string代码

```javascript
export function evalCode(scriptSrc, code) {
	const key = scriptSrc;
	if (!evalCache[key]) {
		const functionWrappedCode = `(function(){${code}})`;
        // indirect invoke eval, eval will invoke in global scope.
		evalCache[key] = (0, eval)(functionWrappedCode);
	}
	const evalFunc = evalCache[key];
	evalFunc.call(window);
}
```

### getExcutableScript

将执行的代码的this引用绑定在proxy对象上。

```javascript
function getExecutableScript(scriptSrc, scriptText, opts = {}) {
	const { proxy, strictGlobal, scopedGlobalVariables = [] } = opts;

	const sourceUrl = isInlineCode(scriptSrc) ? '' : `//# sourceURL=${scriptSrc}\n`;

	// 将 scopedGlobalVariables 拼接成函数声明，用于缓存全局变量，避免每次使用时都走一遍代理
	const scopedGlobalVariableFnParameters = scopedGlobalVariables.length ? scopedGlobalVariables.join(',') : '';

	// 通过这种方式获取全局 window，因为 script 也是在全局作用域下运行的，所以我们通过 window.proxy 绑定时也必须确保绑定到全局 window 上
	// 否则在嵌套场景下， window.proxy 设置的是内层应用的 window，而代码其实是在全局作用域运行的，会导致闭包里的 window.proxy 取的是最外层的微应用的 proxy
	const globalWindow = (0, eval)('window');
	globalWindow.proxy = proxy;
	// TODO 通过 strictGlobal 方式切换 with 闭包，待 with 方式坑趟平后再合并
	return strictGlobal
		? (
			scopedGlobalVariableFnParameters
				? `;with(window.proxy){(function(${scopedGlobalVariableFnParameters}){;${scriptText}\n${sourceUrl}}).bind(window)(${scopedGlobalVariableFnParameters})};`
				: `;(function(window, self, globalThis){with(window){;${scriptText}\n${sourceUrl}}}).bind(window.proxy)(window.proxy, window.proxy, window.proxy);`
		)
		: `;(function(window, self, globalThis){;${scriptText}\n${sourceUrl}}).bind(window.proxy)(window.proxy, window.proxy, window.proxy);`;
}
```

### execScripts函数


```javascript
export function execScripts(entry, scripts, proxy = window, opts = {}) {
	// ...

	return getExternalScripts(scripts, fetch, error)
		.then(scriptsText => {

			const geval = (scriptSrc, inlineScript) => {
				    const rawCode = beforeExec(inlineScript, scriptSrc) || inlineScript;
                    const code = getExecutableScript(scriptSrc, rawCode, { proxy, strictGlobal, scopedGlobalVariables });

                    evalCode(scriptSrc, code);
                    afterExec(inlineScript, scriptSrc);
			};

			function exec(scriptSrc, inlineScript, resolve) {
				// ...
			}

			function schedule(i, resolvePromise) {
                if (i < scripts.length) {
                    const scriptSrc = scripts[i];
                    const inlineScript = scriptsText[i];

                    exec(scriptSrc, inlineScript, resolvePromise);
                    // resolve the promise while the last script executed and entry not provided
                    if (!entry && i === scripts.length - 1) {
                        resolvePromise();
                    } else {
                        schedule(i + 1, resolvePromise);
                    }
                }
			}

			return new Promise(resolve => schedule(0, success || resolve));
		});
}
```

### exec

执行代码，执行到entry代码时，返回exports

```javascript
function exec(scriptSrc, inlineScript, resolve) {
    if (scriptSrc === entry) {
        noteGlobalProps(strictGlobal ? proxy : window);

        try {
            // bind window.proxy to change `this` reference in script
            geval(scriptSrc, inlineScript);
            const exports = proxy[getGlobalProp(strictGlobal ? proxy : window)] || {};
            resolve(exports);
        } catch (e) {
            // ...
        }
    } else {
        if (typeof inlineScript === 'string') {
            try {
                // bind window.proxy to change `this` reference in script
                geval(scriptSrc, inlineScript);
            } catch (e) {
                // consistent with browser behavior, any independent script evaluation error should not block the others
                throwNonBlockingError(e, `[import-html-entry]: error occurs while executing normal script ${scriptSrc}`);
            }
        } else {
            // external script marked with async
            inlineScript.async && inlineScript?.content
                .then(downloadedScriptText => geval(inlineScript.src, downloadedScriptText))
                .catch(e => {
                throwNonBlockingError(e, `[import-html-entry]: error occurs while executing async script ${inlineScript.src}`);
            });
        }
    }
}
```