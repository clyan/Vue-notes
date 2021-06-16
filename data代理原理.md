数据代理原理：将data返回的对象赋值给vm.\_data，通过proxy代理，  Object.defineProperty(target, key, sharedPropertyDefinition) 定义vm的key返回的是vm.\_data.key的值。

```js
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
 proxy(vm, `_data`, key)


const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}

export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
 
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {

    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

