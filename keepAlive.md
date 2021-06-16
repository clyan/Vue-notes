# keepAlive源码解析

> Vue.options.components保存者所有内置组件，new 实例或者使用Vue.extends创建实例时会合并该配置到实例的componets上，所以每个组件都能使用。

注意： 只能缓存组件节点不能缓存普通节点

## 定义

keep-alive就是一个带有render函数的一个组件对象，接受include,exlcude,max三个参数，（ `abstract` 为 true，是一个抽象组件，它在组件实例建立父子关系的时候会被忽略）

`props` 定义了 `include`，`exclude`，它们可以字符串或者表达式

`include` 表示只有匹配的组件会被缓存，而 `exclude` 表示任何匹配的组件都不会被缓存

`props` 还定义了 `max`，它表示缓存的大小，因为我们是缓存的 `vnode` 对象，它也会持有 DOM，当我们缓存很多的时候，会比较占用内存，所以该配置允许我们指定缓存大小。

原理： 

1.  created 钩子时创建一个空对象，cache和keys保存对象的键
2. destroyed钩子时，执行pruneCacheEntry 删除cache中的键值
3. mounted 钩子时,监听exclude和include ，将匹配的组件名称或者tag才能缓存。
4. render 获取第一个子组件，（如果没有则返回默认的第一个元素），并且使用最近最少使用的思想，每次访问组件都将该组件对应的key放到keys的最后面，当max满了则先进行移除第一个,还要判断如果要删除的缓存并的组件 `tag` 不是当前渲染组件 `tag`，也执行删除缓存的组件实例的 `$destroy` 方法。。

```js
export default {
  name: 'keep-alive',
  abstract: true,

  props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number]
  },

  created () {
    this.cache = Object.create(null)
    this.keys = []
  },

  destroyed () {
    for (const key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys)
    }
  },

  mounted () {
    this.$watch('include', val => {
      pruneCache(this, name => matches(val, name))
    })
    this.$watch('exclude', val => {
      pruneCache(this, name => !matches(val, name))
    })
  },

  render () {
    const slot = this.$slots.default
    const vnode: VNode = getFirstComponentChild(slot)
    const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
    if (componentOptions) {
      // check pattern
      const name: ?string = getComponentName(componentOptions)
      const { include, exclude } = this
      if (
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }

      const { cache, keys } = this
      const key: ?string = vnode.key == null

        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
      if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance

        remove(keys, key)
        keys.push(key)
      } else {
        cache[key] = vnode

        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
      }

      vnode.data.keepAlive = true
    }
    return vnode || (slot && slot[0])
  }
}

```

## patch

第一次patch，跟普通组件没有区别，但是第二次，会直接去除缓存的vnode,并且如果vnode.data.keepAlive为true，不会执行$mounted, 执行KeepALive钩子。

总结： `<keep-alive>` 的实现原理通过分析我们知道了 `<keep-alive>` 组件是一个抽象组件，它的实现通过自定义 `render` 函数并且利用了插槽，并且知道了 `<keep-alive>` 缓存 `vnode`，了解组件包裹的子元素——也就是插槽是如何做更新的。且在 `patch` 过程中对于已缓存的组件不会执行 `mounted`，所以不会有一般的组件的生命周期函数但是又提供了 `activated` 和 `deactivated` 钩子函数。

根据vnode的elm直接将内容插入到页面上，而不用去走children patch的过程