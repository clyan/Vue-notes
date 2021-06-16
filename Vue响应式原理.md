首先执行initData , 将 传入的data函数或者对象（如果是函数则执行函数返回对象），先将对象所有的data代理到vm实例上， 然后调用observer传入 data对象， 然后为判断data是否已将有`  __ob__ `属性和`  __ob__ `是否是属于Observer 对象，如果不属于， 则new Observer (obj);并传入此对象，Observer 会new一个Dep，继续遍历对象的Kyes执行 defineReactive 传入对象与对象的key,  首先会new 一个 Dep,  然后又获取val = obj[key]的内容， 递归调用observe(*val*) , 然后就执行Object.defineProperty ，所以首先是响应化子对象的属性，再响应化父对象的属性。

```javascript
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
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
  })
```

在组件挂载的时候，会new watcher() 在watcher中会将当前这个watcher实例赋值给Dep.target,  再调用render, 在render过程中，会触发一次依赖的属性的  get， 接着执行 属性的 get 函数， 调用  dep.depend()