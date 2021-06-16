createElement：src\core\vdom\create-element.js

createElement 调用 _createElement 函数

_createElement : 

判断alwaysNormalize 参数是否是 true，如果是true 代表是用户手写的render ,需要将children 规范化,最终将children  打平成一个一维数组，每一项都是Vdom( 此时 每一项还是h(tag, data={}, children =[]))的形式

````
  ......
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  ......
````

simpleNormalizeChildren: 

```javascript
// 1。当子组件包含组件时——因为是功能性组件
//可能返回一个数组而不是单个根。在这种情况下，只是一个简单的
//需要规范化——如果任何一个子数组是数组，则将整个数组平化
//使用Array.prototype.concat。它保证只有1级的深度
//因为函数组件已经规范化了它们自己的子组件。
export function simpleNormalizeChildren (children: any) {
  for (let i = 0; i < children.length; i++) {
    if (Array.isArray(children[i])) {
      return Array.prototype.concat.apply([], children)
    }
  }
  return children
}
```

normalizeChildren:如果传入的children为基础类型 ，则直接创建一个文本节点， 如果是一个数组则调用normalizeArrayChildren

normalizeArrayChildren主要逻辑，如果存在深层嵌套则递归调用，最后返回一个一维数组，

```javas
export function normalizeChildren (children: any): ?Array<VNode> {
  return isPrimitive(children)
    ? [createTextVNode(children)]
    : Array.isArray(children)
      ? normalizeArrayChildren(children)
      : undefined
}
```

_createElement 根据当前传入的 tag 判断当前是 组件还是普通的标签

如果是普通的标签则 生成 new VNode(tag，data， children， undefined, undefined, context ）

````javascript
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
		........
     data vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
        
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
````

如果是 不是字符串则是组件：