首先看下watcher的使用方法：

1. a: 监听对象的某个属性，使用`.` 或者`[]`取值的形式，需使用字符串，
2. b: 回调函数如需使用methods中定义过的函数，则使用字符串形式的方式包裹函数，内部会去vm上找这个函数。
3. c: 如果是一个对象，则需提供handler函数,并且提供`deep=true` c.cc改变了也会触发。
4. d:` immediate=true` 初始值的时候就会触发handler, 没有设置则再再次更新时触发。
5. e: 可以提供数组形式，传递多个handler, 依次执行。

```js
data() {
    return {
        a: {
            aa: 1
        },
        b: 2,
        c: {
            cc: 123
        },
        d: 4,
        e: {
            f: {
                g: 5
            }
        }
    }
},
watch: {
            'a.aa': function(newVal, oldVal) {
                console.log('AnewVal', newVal)
                console.log('AoldVal', oldVal)
            },
            b:'BMethod',
            c: {
                handler:function(newVal, oldVal) {
                    console.log('CnewVal', newVal)
                    console.log('ColdVal', oldVal)
                },
                deep:true
            },
            // immediate立即执行。其他的都需要在修改时执行。
            d:{
                handler: 'DMethod',
                immediate: true
            },
            e: [
                'Ehandle1',
                function handle2 (newVal, oldVal) { 
                    console.log('EnewVal2', newVal)
                    console.log('EoldVal2', oldVal)
                },
                {
                    handler: function handle3 (newVal, oldVal) { 
                        console.log('EnewVal3', newVal)
                        console.log('EoldVal3', oldVal)
                    },
                }
            ],
        }
```



遍历watch对象，取出watch属性对应的handler，一个key可以对应多个handler，所以遍历创建createWatcher

```js
function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}


```

首先对 `hanlder` 的类型做判断，拿到它最终的回调函数，最后调用 `vm.$watch(keyOrFn, handler, options)` 函数

```js
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

为定义的属性实例化一个Watcher，并且`user=true` ， 判断immediate 是否为true, 则立即执行，返回一个unwatchFn函数，销毁此watch.

```js
 Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      try {
        cb.call(vm, watcher.value)
      } catch (error) {
        handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
      }
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }
```



watcher的类型

```js
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    }
```

**deep原理：**

> 遍历对象，触发对象属性的get对当前watcher进行收集，更新时就会通知此watcher执行。

**user：**

> 使用watch或者vm.$watch定义的watch user=true, 作用是在执行handler时捕获异常，因为用户输入的函数可能存在错误

```js
if (this.user) {
    try {
      this.cb.call(this.vm, value, oldValue)
    } catch (e) {
      handleError(e, this.vm, `callback for watcher "${this.expression}"`)
    }
  } else {
    this.cb.call(this.vm, value, oldValue)
  }
```

**lazy:**

> computed watcher

**sync:**

> 当响应式数据发送变化后，触发了 `watcher.update()`，只是把这个 `watcher` 推送到一个队列中，在 `nextTick` 后才会真正执行 `watcher` 的回调函数。而一旦我们设置了 `sync`，就可以在当前 `Tick` 中同步执行 `watcher` 的回调函数。



**总结：**

计算属性本质上是 `computed watcher`，而侦听属性本质上是 `user watcher`。就应用场景而言，计算属性适合用在模板渲染中，某个值是依赖了其它的响应式对象甚至是计算属性计算而来；而侦听属性适用于观测某个值的变化去完成一段复杂的业务逻辑。

同时我们又了解了 `watcher` 的 4 个 `options`，通常我们会在创建 `user watcher` 的时候配置 `deep` 和 `sync`，可以根据不同的场景做相应的配置。