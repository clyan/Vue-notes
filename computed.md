# 初始化过程

>  在initState时执行initComputed，遍历computed的属性，为每个属性实例化一个Watcher，并且watcher的dirty与lazy为true， getter为属性对应的函数或者带get的对象。
>
> 判断vm.data上是否已存在该属性，如果存在则抛出错误，不存在则执行defineComputed。

**initComputed：**

```javascript

const computedWatcherOptions = { lazy: true }

function initComputed (vm: Component, computed: Object) {

  const watchers = vm._computedWatchers = Object.create(null)

  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
     
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```

**defineComputed:**

>  通过createComputedGetter(key)   获取get函数 ，并通过Object.defineProperty为vm定义该属性的get

```javascript
export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    .........
  }
   
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

**createComputedGetter：**

>  注意get执行是在调用该计算属性时触发（即render过程中），判断当前watcher的ditry如果为true则执行 watcher.evaluate()重新计算值。

```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

**watcher.evaluate:**

> 如：getName对应的函数。
>
> ```javascript
> computed:{
>     getName(){
>         return this.name + '456'
>     }
> }
> ```

> 进行求值。this.get最终调用的就是计算属性对应的函数，并把当前属性的this.dirty  设置为false，只有当this.dirty = true时，才会进行求值，那么记下来分析什么时候会再次求值。

```javascript
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }
```

**阶段总结：**

此时只是对computed对应的属性进行重新定义get和对应一个Watcher,并没有触发，接下来看首次渲染与更新时的逻辑

# 首次渲染

在执行完初始化后，执行beforeMounted钩子, 此时 new Watcher最后一个参数为true代表渲染watcher,在watcher中执行updateComponent。

```
  callHook(vm, 'beforeMount')

  let updateComponent
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        .....	
  } else {
    updateComponent = () => {
      // debugger;
      // vm._render() 返回vnode
      vm._update(vm._render(), hydrating)
    }
  }

  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
```

> 首先明确： vue维护了一个Watcher栈，每次执行Watcher.get时将该Watcher赋值给Dep.target
>
> 记住 pushTarget(this)，是将watcher放到栈中即可，此时栈中为` [渲染watcher]` 
>
> 调用 this.getter.call(vm, vm) 执行 以下函数，
>
>   明确vm.\_render()是将render函数的形式转换为Vnode,vm.\_update()是将Vnode转换成真实dom。
>
> ```javascript
>     updateComponent = () => {
>       // debugger;
>       // vm._render() 返回vnode
>       vm._update(vm._render(), hydrating)
>     }
> ```

```javascript
# watcher

get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
```

在执行render时，会调用所有依赖的响应式数据，

当执行this.getName时，this.getName对应的计算watcher此时dirty为true

所以执行watcher.evaluate, 调用watcher.get（）

首先将当前`计算watcher`放入到watcher栈中，此时栈中为`[渲染watcher, 计算watcher]` 

然后执行this.getName对应的函数内容，发现依赖了this.name,触发this.name的get,

将当前计算watcher加入到this.name对应的dep.subs中，  `计算watcher ` 将该dep加入到  `计算watcher ` 的deps中   (这一步关键)

然后将计算watcher从watcher栈中移除，此时栈中为`[渲染watcher]` 

然后计算出 this.getName对应的函数的值赋值给`计算watcher.value`,

然后将dirty设置为false。

接着判断当前Dep.target是否存在，此时dep.target 为`[渲染watcher]`

存在则执行 watcher.depend() ，此时watcher为当前计算Watcher	

```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

此时计算Watcher中的deps中存在了this.name对应的dep,执行该dep.depend()

```javascript
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }
```

可以看到，就是把this.name对应的dep添加到渲染Watcher中，把渲染Watcher添加到this.name的dep中。

````javascript
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
````

```javascript
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
```

```javascript
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }
```

最后执行 return watcher.value，将值返回给页面。 完成第一次计算。

总结：每个计算属性都是一个内部属性为dirty=true与lazy=true的Watcher，并重写属性的get方法，set方法（set为一个空函数，因为计算属性一般不会更改。）

在调用此计算属性时，触发get, 执行计算属性对应的函数，发现有`依赖的响应式数据`，将该`依赖的响应式数据`的dep添加到watcher中，并把该watcher添加到`依赖的响应式数据`的dep.subs中，然后将dirty设置为false,

然后将渲染Watcher添加到该`依赖的响应式数据`  dep.subs中， `依赖的响应式数据`的dep添加到`渲染watcher`的newDeps中。

此时，`依赖的响应式数据`dep 中有了`【计算Watcher，渲染watcher】`,  计算Watcher中有了`响应式数据的Dep` 渲染watcher也有了`响应式数据的Dep` 。

最后将计算属性函数的值返回。首次触发计算完成。





# 第二次调用

页面上有可能会触发多次this.getName(), 第二次调用时，发现dirty为false, 所以不会再调用watcher.evaluate()，即不会再执行函数计算，其次，再次调用watcher.depend()时最终会执行Dep.target.addDep(this)。

```javascript

function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      debugger;
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

此时该计算属性所依赖的`响应式数据dep`已经存在于渲染watcher中了，所以渲染watcher不会重复添加。

```javascript
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
```

总结： 第二次调用时，只会返回计算属性的值。



# 依赖的数据改变

此时当this.name改变时，会触发this.name的set函数，

判断新值与旧值是否一致，如果一直没直接返回。  dep.notify()通知this.name dep所有的watcher进行更新。

此时dep中有两个watcher，由首次渲染得知顺序为, `[计算watcher，渲染watcher]`

```javascript
set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
```

Dep： subs[i].update()就是执行watcher的update，

```
  notify () {

    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
```

先执行计算watcher , lazy为true，则把该watcher的dirty设置为true。

```
  update () {
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
```

再执行渲染watcher,渲染watcher中又会调用this.getName这个计算属性，此时发现dirty为true了，则再次计算值并返回。

总结： 

1. 响应式数据更新，执行计算watcher更新将dirty设置为true
2. 执行渲染watcher，调用计算属性函数，重新求值，返回给页面。



# 依赖的数据改变，最终值不变

如果依赖的数据改变，最终值不变，页面会不会重新渲染？



如下例子：依赖的数据改变，最终值不变

getMsg依赖a,b,初始a=1,b=2点击按钮修改a=2,b=1。最终值都为3，页面是没有发生改变的。这是为什么？

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title> </title>
</head>
<body>
    <div id = "app">
        <div>
            <button @click="changeMsg()">改变值</button>
        </div>
        <div>
            <p>{{getMsg}}</p>
        </div>
    </div>
</body>
<script src="../dist/vue.js"></script>
<script>
    new Vue({
        el: "#app",
        data() {
            return {
                a: 1,
                b: 2,
            }
        },
        methods:{
            changeMsg(){
               this.a = 2;
               this.b = 1;
            }
        },
        computed:{
            getMsg(){
                return this.a + this.b;
            }
        }
    })

</script>
</html>
```

首先明确，getMsg对应一个计算watcher, 该watcher中deps含有a.dep与b.dep,

`a.dep.subs`与`b.dep.subs`中都含有`该计算watcher`, 并且`a.dep.subs`与`b.dep.subs`都含有两个watcher`[计算watcher，渲染watcher]`那么接下来分析，此更新流程。

当点击按钮时执行changeMsg， 调用this.a = 2 ，触发this.a 的set函数。

可以看到先判断新旧值是否一致，一致直接返回，不一致则通知dep.subs所有的watcher进行update,

此时dep中watcher的顺序为`[计算watcher，渲染watcher]`，即先执行计算watcher,再执行渲染watcher,

```
  //  **只保留了关键逻辑。**
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
  
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      .....
      dep.notify()
    }
```

```js
  notify () {
   
    const subs = this.subs.slice()
    ......
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

计算watcher的lazy为true，所以将计算watcher的dirty改为true,

然后执行渲染watcher，会执行queueWatcher。

```js
  update () {
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
```

维护一个队列，将更新的逻辑统一交由nextTick异步渲染flushSchedulerQueue，使用has保证，同一个watcher只添加一次，` 此时将watcher添加进队列`，

判断`flushing(初始为false)`,是否已执行flushSchedulerQueue逻辑， 通过waiting保证nextTick只执行一次。

接着将flushSchedulerQueue 传入nextTick中，并且waiting设置为true.

```js
const queue: Array<Watcher> = []
const activatedChildren: Array<Component> = []
let has: { [key: number]: ?true } = {}
let circular: { [key: number]: number } = {}
let waiting = false
let flushing = false
let index = 0

export function queueWatcher (watcher: Watcher) {

  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}

```

注意nextTick，并没有立即执行flushSchedulerQueue，而是将flushSchedulerQueue添加到了callbacks队列中，判断pending 为false则执行timerFunc。

```js
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

timerFunc：降级处理   Promise -> MutationObserver -> setImmediate  ->  setTimeout

以promise为例，这里我们发现，此时虽然立即执行了timerFunc，但是timerFunc执行了一个异步函数p.then,但是此时回到最开始。this.b是同步操作，还没有执行，这是会又会触发this.b的set函数。

**说明flushSchedulerQueue此时并没有执行也就是页面还不会渲染。**

```
let timerFunc

if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {

  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {

  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {

  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

接着触发b的set函数通知所有的watcher，依旧是先执行计算Watcher，虽然此时该计算watcher的dirty已经为true,但依旧会再次赋值为true.

接着触发`渲染watcher` 发现watcher中已经存在该渲染watcher了，则不再添加到`Watcher队列`中，也就不会向nextTick的callback中添加flushSchedulerQueue了。

此时同步代码就执行完成了。

最后执行nextTick的`flushCallbacks`  此时flushCallbacks中只有一个`flushSchedulerQueue` 

```javascript
const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

接着执行`flushSchedulerQueue`  此时queue即watcher队列中只有一个渲染watcher。对队列进行排序，遍历队列，执行wathcher的before,即beforeUpdate钩子， 再执行watcher.run()；

```js
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. 组件的更新由父到子；因为父组件的创建过程是先于子的，所以 watcher 的创建也是先父后子，执行顺序也应该保持先父后子。
  // 2.用户的自定义 watcher 要优先于渲染 watcher 执行；因为用户自定义 watcher 是在渲染 watcher 之前创建的。
  // 3.如果一个组件在父组件的 watcher 执行期间被销毁，那么它对应的 watcher 执行都可以被跳过，所以父组件的 watcher 应该先执行
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
      // 从has中删除该watcher
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
	...更多...
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

run中调用 this.get最终触发this.getter.

```js
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }
```

此时watcher为`渲染watcher`, this.getter为updateComponent，即会重新调用render与update,在render过程中，会在次触发this.getMsg(),

```js
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
```

this.getMsg()触发 get,执行watcher.evaluate()重新根据a与b计算值，最后返回值。

```js
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    debugger;
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

render执行完后，执行update，两个节点并未发生改变，所以dom就不会更新啦。

最后总结，计算属性，只要依赖的响应式数据发生改变，则会重新计算，最后dom更新取决于，两次Vnode是否一致，值未改变，两次vnode自然是一致的，所以就不会更新啦。

