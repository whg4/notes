# Single-spa

single-spa主要作用

- 劫持路由事件，通过路由变化加载/卸载微应用，调用微应用的生命周期函数
- 没有提供沙箱机制，仅仅只是一个微应用加载工具库

## 路由事件劫持

single-spa会劫持hashchange和popstate事件。

```javascript
export const routingEventsListeningTo = ["hashchange", "popstate"];
```

### addEventListener猴子补丁

监听到hashchange或popstate事件后，触发**urlReroute**函数。

```javascript
// We will trigger an app change for any routing events.
window.addEventListener("hashchange", urlReroute);
window.addEventListener("popstate", urlReroute);

// Monkeypatch addEventListener so that we can ensure correct timing
const originalAddEventListener = window.addEventListener;
const originalRemoveEventListener = window.removeEventListener;
window.addEventListener = function (eventName, fn) {
if (typeof fn === "function") {
  if (
    routingEventsListeningTo.indexOf(eventName) >= 0 &&
    !find(capturedEventListeners[eventName], (listener) => listener === fn)
  ) {
    capturedEventListeners[eventName].push(fn);
    return;
  }
}

return originalAddEventListener.apply(this, arguments);
};

window.removeEventListener = function (eventName, listenerFn) {
if (typeof listenerFn === "function") {
  if (routingEventsListeningTo.indexOf(eventName) >= 0) {
    capturedEventListeners[eventName] = capturedEventListeners[
      eventName
    ].filter((fn) => fn !== listenerFn);
    return;
  }
}

return originalRemoveEventListener.apply(this, arguments);
};

```

## urlReroute

urlReroute会调用reroute函数。

```javascript
function urlReroute() {
  reroute([], arguments);
}
```

### reroute

reroute函数主要用来加载微应用，调用微应用的生命周期函数，然后在调用微应用生命周期钩子成功后，

调用所有劫持的hashchange，popstate函数。

```javascript
export function reroute(pendingPromises = [], eventArguments) {
  // ...
  if (isStarted()) {
    appChangeUnderway = true;
    appsThatChanged = appsToUnload.concat(
      appsToLoad,
      appsToUnmount,
      appsToMount
    );
    return performAppChanges();
  } else {
    // ...
  }
  // ...
  function performAppChanges() {
    return Promise.resolve().then(() => {
      // ...
      return unmountAllPromise
        .catch((err) => {
          callAllEventListeners();
          throw err;
        })
        .then(() => {
          /* Now that the apps that needed to be unmounted are unmounted, their DOM navigation
           * events (like hashchange or popstate) should have been cleaned up. So it's safe
           * to let the remaining captured event listeners to handle about the DOM event.
           */
          callAllEventListeners();

          return Promise.all(loadThenMountPromises.concat(mountPromises))
            .catch((err) => {
              pendingPromises.forEach((promise) => promise.reject(err));
              throw err;
            })
            .then(finishUpAndReturn);
        });
    });
  }

  /* We need to call all event listeners that have been delayed because they were
   * waiting on single-spa. This includes haschange and popstate events for both
   * the current run of performAppChanges(), but also all of the queued event listeners.
   * We want to call the listeners in the same order as if they had not been delayed by
   * single-spa, which means queued ones first and then the most recent one.
   */
  function callAllEventListeners() {
    pendingPromises.forEach((pendingPromise) => {
      callCapturedEventListeners(pendingPromise.eventArguments);
    });

    callCapturedEventListeners(eventArguments);
  }
}
```

### callCapturedEventListeners

```javascript
/* We need to call all event listeners that have been delayed because they were
* waiting on single-spa. This includes haschange and popstate events for both
* the current run of performAppChanges(), but also all of the queued event listeners.
* We want to call the listeners in the same order as if they had not been delayed by
* single-spa, which means queued ones first and then the most recent one.
*/
function callAllEventListeners() {
    pendingPromises.forEach((pendingPromise) => {
      callCapturedEventListeners(pendingPromise.eventArguments);
    });

    callCapturedEventListeners(eventArguments);
}
```

可以发现**callCaturedEventListeners**只有当eventArguments不为undefined时才会被调用。即只有single-spa中**popstate，hashchange**事件监听器调用才会触发reroute事件，然后才会调用子应用注册的事件监听器。

```javascript
export function callCapturedEventListeners(eventArguments) {
  if (eventArguments) {
    const eventType = eventArguments[0].type;
    if (routingEventsListeningTo.indexOf(eventType) >= 0) {
      capturedEventListeners[eventType].forEach((listener) => {
        try {
          // The error thrown by application event listener should not break single-spa down.
          // Just like https://github.com/single-spa/single-spa/blob/85f5042dff960e40936f3a5069d56fc9477fac04/src/navigation/reroute.js#L140-L146 did
          listener.apply(this, eventArguments);
        } catch (e) {
          setTimeout(() => {
            throw e;
          });
        }
      });
    }
  }
}
```

## pushState和replaceState函数劫持

single-spa还将pushState和replaceState函数进行了猴子补丁，

```javascript
window.history.pushState = patchedUpdateState(
    window.history.pushState,
    "pushState"
);
window.history.replaceState = patchedUpdateState(
    window.history.replaceState,
    "replaceState"
);
```

可以看到，如果urlRerouteOnly不为true，且pushState或replaceState使得url变化，会触发一次popstate事件，让**reroute**函数被调用。

```javascript
function patchedUpdateState(updateState, methodName) {
  return function () {
    const urlBefore = window.location.href;
    const result = updateState.apply(this, arguments);
    const urlAfter = window.location.href;

    if (!urlRerouteOnly || urlBefore !== urlAfter) {
      if (isStarted()) {
        // fire an artificial popstate event once single-spa is started,
        // so that single-spa applications know about routing that
        // occurs in a different application
        window.dispatchEvent(
          createPopStateEvent(window.history.state, methodName)
        );
      } else {
        // do not fire an artificial popstate event before single-spa is started,
        // since no single-spa applications need to know about routing events
        // outside of their own router.
        reroute([]);
      }
    }

    return result;
  };
}
```