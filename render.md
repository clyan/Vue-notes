入口：src\core\instance\lifecycle.js

```javascript
 updateComponent = () => {
      // vm._render() 返回vnode
      debugger;
      vm._update(vm._render(), hydrating)
 }
```

执行 vm._render()

_render定义文件： src\core\instance\render.js   

执行vm.$options.render.call(vm._renderProxy, vm.$createElement)获取vnode

定义入口： src\core\instance\init.js

Vue.prototype._init:中

```javascript

/* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
```

生产环境 会判断当前环境是否否有Proxy对象，如果有则使用Proxy代理vm   

```javascript
  initProxy = function initProxy (vm) {
    if (hasProxy) {
      // determine which proxy handler to use
      const options = vm.$options
      const handlers = options.render && options.render._withStripped
        ? getHandler
        : hasHandler
      vm._renderProxy = new Proxy(vm, handlers)
    } else {
      vm._renderProxy = vm
    }
  }
```

handlers:

hasHandler 判断vm取值时，值是否在vm上，如果没有则抛出警告， getter一样

场景: 页面上使用了vm没有的值, 因为init时   data,props， method中的键都会代理到vm实例上，所以能直接获取vm获取data，props,method等中的值。

```javascript
  const hasHandler = {
    has (target, key) {
      const has = key in target
      const isAllowed = allowedGlobals(key) ||
        (typeof key === 'string' && key.charAt(0) === '_' && !(key in target.$data))
      if (!has && !isAllowed) {
        if (key in target.$data) warnReservedPrefix(target, key)
        else warnNonPresent(target, key)
      }
      return has || !isAllowed
    }
  }

  const getHandler = {
    get (target, key) {
      if (typeof key === 'string' && !(key in target)) {
        if (key in target.$data) warnReservedPrefix(target, key)
        else warnNonPresent(target, key)
      }
      return target[key]
    }
  }
```



开发环境直接为 vm._renderProxy = vm