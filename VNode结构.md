虚拟dom的作用，首先是方便组织结构，其次才说比原生dom节点小，快。

考虑以下结构

```
   new Vue({
        el: "#app",
        render(h){
            return h('div', {
                style: {
                     color: 'red'
                 }
            }, [
                h('div',{
                    style: {
                        color: 'red'
                    }
                }, 'hello'),
                h('div',{
                        
                }, 'world')
            ])
        },
    })
```

types\vnode.d.ts ： 节点参考目录

```
/* @flow */

export default class VNode {
  tag: string | void;	// 标签 div
  data: VNodeData | void;	//数据 { style: {color: 'red'} }
  children: ?Array<VNode>;	// 子节点 [h(div,...),h('div', ...)]
  text: string | void;	// 当前节点的文本，一般文本节点或注释节点会有该属性
  elm: Node | void;	// 当前虚拟节点对应的真实的dom节点
  ns: string | void;	// 节点的namespace
  context: Component | void; //  编译作用域
  key: string | number | void;	//  函数化组件的作用域
  componentOptions: VNodeComponentOptions | void;	// 节点的key属性，用于作为节点的标识，有利于patch的优化
  componentInstance: Component | void; // : 创建组件实例时会用到的选项信息
  parent: VNode | void; // 组件的占位节点

  // strictly internal
  raw: boolean; // raw html
  isStatic: boolean; // 静态节点的标识
  isRootInsert: boolean; // 是否作为根节点插入
  isComment: boolean; // 当前节点是否是注释节点
  isCloned: boolean; // 当前节点是否为克隆节点
  isOnce: boolean; // i当前节点是否有v-once指令
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  devtoolsMeta: ?Object; // used to store functional render context for devtools
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}

export const createEmptyVNode = (text: string = '') => {
  const node = new VNode()
  node.text = text
  node.isComment = true
  return node
}

export function createTextVNode (val: string | number) {
  return new VNode(undefined, undefined, undefined, String(val))
}

// optimized shallow clone
// used for static nodes and slot nodes because they may be reused across
// multiple renders, cloning them avoids errors when DOM manipulations rely
// on their elm reference.
export function cloneVNode (vnode: VNode): VNode {
  const cloned = new VNode(
    vnode.tag,
    vnode.data,
    // #7975
    // clone children array to avoid mutating original in case of cloning
    // a child.
    vnode.children && vnode.children.slice(),
    vnode.text,
    vnode.elm,
    vnode.context,
    vnode.componentOptions,
    vnode.asyncFactory
  )
  cloned.ns = vnode.ns
  cloned.isStatic = vnode.isStatic
  cloned.key = vnode.key
  cloned.isComment = vnode.isComment
  cloned.fnContext = vnode.fnContext
  cloned.fnOptions = vnode.fnOptions
  cloned.fnScopeId = vnode.fnScopeId
  cloned.asyncMeta = vnode.asyncMeta
  cloned.isCloned = true
  return cloned
}

```

