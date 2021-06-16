# 虚拟dom

> 深度优先遍历算法，先遍历子节点再遍历兄弟节点。

简介： 对真实dom的一个抽象。

优点:  虚拟DOM不仅可以变成DOM，还可以变成小程序、ios应用、安卓应用、因为**虚拟DOM本质上只是一个JS对象** ,

**某些情况下快**

- 虚拟DOM可以将多次操作合并，并为一次操作 比如：你添加1000个节点，确实一个个操作的（减少频率）
- 虚拟DOM借助DOM diff可以把多余的操作省掉 比如你添加1000个节点，其实只要10个是新的 （减少范围）

缺点：在需要极致的性能时，需要针对性优化，使用vdom不太合适。

[vdom真的高效吗](https://www.h5w3.com/39698.html)



**对比oldVnode与Vnode**

1. 根据key与tag 是否是注释节点, 节点的属性，input的type，判断是否为同一节点
2. 如果不是同一节点，根据新的Vnode渲染页面，然后删除老节点
3. 如果是同一节点
   - 判断oldVnode与Vnode引用一致，有则直接返回。
   - 判断新的Vnode是否有text，如果有并且与oldVnode不同，则更新文本内容
   - 判断oldVnode.children与Vnode.children是否存在
      - 都存在,且不一致则进行children对比，即diff算法
      - 只有Vnode.children存在，则直接添加子节点
      - 只有oldVnode.children存在，则直接删除直接子节点
   - 在对开始和结束节点比较的时候，总共有四种情况
      - `oldStartVnode / newStartVnode` (旧开始节点 / 新开始节点)
      - `oldEndVnode / newEndVnode` (旧结束节点 / 新结束节点)
      - `oldStartVnode / oldEndVnode` (旧开始节点 / 新结束节点)
      - `oldEndVnode / newStartVnode` (旧结束节点 / 新开始节点)
      -  首先比较`oldStartVnod`与`newStartVnode` 接着递归执行第三步 ，头指针+1
      - 再比较 `oldEndVnode`, `newEndVnode` 接着递归执行第三步 , 尾针 - 1
      - 再比较 `oldStartVnode`, `newEndVnode` 接着递归执行第三步 （**说明oldStartVnode已经跑到了后面，那么就将oldStartVnode.el移到oldEndVnode.el的后边。oldStartIdx+1，newEndIdx-1）**
      - 再比较`oldEndVnode`, `newStartVnode` 接着递归执行第三步 **（说明oldEndVnode已经跑到了前面，那么就将oldEndVnode.el移到oldStartVnode.el的前边。oldEndIdx-1，newStartIdx+1）**
      - **如果都不符合，通过newStartVnode.key 看是否能在oldVnode中找到，如果没有则新建节点，如果有则对比新旧节点中相同key的Node，newStartIdx+1** 

- 最后如果oldStartInx > newStartInx 说明 newStartInx  到 newStartInx 