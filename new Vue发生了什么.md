入口： src\core\instance\index.js

```
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```

执行this._init , this._init定义在原型上，通过initMixin 进行拓展

initMixin 入口：  src\core\instance\init.js

```
  ......
  if (options && options._isComponent) {
   ......
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
 	......
    // expose real self
    vm._self = vm
    initLifecycle(vm)   // 初始化组件间的父子关系
    initEvents(vm)      //更新组件的事件
    initRender(vm)      // 初始化_c方法
    callHook(vm, 'beforeCreate')  //beforeCreate钩子函数
    initInjections(vm) // 初始化inject
    initState(vm)         // 初始化状态
    initProvide(vm)   // 初始化provide
    callHook(vm, 'created') //created钩子函数
    .......
   	if (vm.$options.el) {
        vm.$mount(vm.$options.el)
    }
```

​	1.  合并 options 赋值给 vm.$options

 2.  initLifecycle初始化组件间的父子关系 , 初始化当前实例的$parent， $children，$refs 等

     ```
       ...
           vm.$parent = parent
           vm.$root = parent ? parent.$root : vm
     
           vm.$children = []
           vm.$refs = {}
     
           vm._watcher = null
           vm._inactive = null
           vm._directInactive = false
           vm._isMounted = false
           vm._isDestroyed = false
           vm._isBeingDestroyed = false
       ...
     ```

3. initEvents 初始化更新组件的事件， 创建了一个空对象赋值给vm._events

   ```javascript
   export function initEvents (vm: Component) {
     vm._events = Object.create(null)
     vm._hasHookEvent = false
     // init parent attached events
     const listeners = vm.$options._parentListeners
     if (listeners) {
       updateComponentListeners(vm, listeners)
     }
   }
   ```

4. initRender 初始化_c方法 也就是 $createElement ，创建Element  ，其他的vm.$slots， vm.$scopedSlots,  vm. $attrs , vm.$listeners 是响应式的，  此函数还导出了renderMixin

   

   ```javascript
   const options = vm.$options
     const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
     const renderContext = parentVnode && parentVnode.context
     vm.$slots = resolveSlots(options._renderChildren, renderContext)
     vm.$scopedSlots = emptyObject
     vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
    
     vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
   
     const parentData = parentVnode && parentVnode.data
   
     if (process.env.NODE_ENV !== 'production') {
   	.....
     } else {
       defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
       defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
     }
   ```

5. initInjections 父组件使用 Provide注入的值， 与Provide配合使用，使用时 `inject` 选项应该是   一个字符串数组，或一个对象，对象的 key 是本地的绑定名，value 是在可用的注入内容中搜索用的 key (字符串或 Symbol)，或一个对象，该对象的`from` property 是在可用的注入内容中搜索用的 key (字符串或 Symbol)`default` property 是降级情况下使用的 value

6. initState 初始化 props, methods, data, computed, initWatch，对

   ```
   export function initState (vm: Component) {
     vm._watchers = []
     const opts = vm.$options
     if (opts.props) initProps(vm, opts.props)
     if (opts.methods) initMethods(vm, opts.methods)
     if (opts.data) {
       initData(vm)
     } else {
       observe(vm._data = {}, true /* asRootData */)
     }
     if (opts.computed) initComputed(vm, opts.computed)
     if (opts.watch && opts.watch !== nativeWatch) {
       initWatch(vm, opts.watch)
     }
   }
   ```

7.  initProvide  初始化provide, provide使用时 选项应该是一个对象或返回一个对象的函数

   ```javascript
   export function initProvide (vm: Component) {
     const provide = vm.$options.provide
     if (provide) {
       vm._provided = typeof provide === 'function'
         ? provide.call(vm)
         : provide
     }
   }
   ```

8. callHook(vm, 'created')  执行 created 钩子函数
9. 执行vm.$mount(vm.$options.el)