文件入口： src\platforms\web\entry-runtime-with-compiler.js

缓存$mount  *const* mount = *Vue*.prototype.$mount

重写 *Vue*.prototype.$mount（ *el*?: *string* | *Element*,*hydrating*?: *boolean*） 进行挂载；

el 的类型可以是字符串dom或者*Element*

流程：

1. 处理el: 如果是字符串则 使用原生  document.querySelector(*el*) 并返回Element, 如果不存在则报错，并document.createElement('div')一个，如果是*Element* 则直接返回。

   并且返回dom不能是body 与 document（如果是直接return）

   执行完成后此时返回的el 是 *Element* 类型

2. 获取this.$options 

   - 判断options.render 如果为空 

     - 判断options.template 

       - 若存在
         - 如果 template  的类型为string 并且 第一个字符为 # ，则调用原生document.querySelector(template) 跟el一样的逻辑
         - 如果 template  的类型为html类型则返回 template.innerHTML
         - 否则报错并直接return this;

       - 不存在	
         - template  = getOuterHTML(*el*) 如果el.outerHTML 存在则直接返回(string 类型)
         - 不存在则创建一个外层容器container（div）包裹它，并返回container.innerHTML (string 类型)

     - 执行完template  存在与不存在的逻辑后，再一次判断，template  是否存在，
       - 若存在
         - 执行compileToFunctions 执行编译逻辑，返回render, staticRenderFns 并赋值给 
         -  this.$options.render = render， this.$options.staticRenderFns = staticRenderFns

   - options.render不为空时 

   3. 执行缓存的mount , 传入 `mount.call(this, el, hydrating)`  `hydrating`服务端渲染相关先不管

- 进入src\platforms\web\runtime\index.js 

- 判断当前是否是浏览器环境，再一次判断el是否存在，不存在则赋值underfined给el

- 执行 mountComponent ， 进入src\core\instance\lifecycle.js

  - 将el 赋值给`vm.$el`
  
  - 判断`vm.$options.render` 是否存在
    - 不存在： 赋值一个createEmptyVNode函数给*vm*.$options.render ，此函数返回一个VNode实例，并且实例 isComment = true（render，不存在有两种情况，一个是用户没有手写render,第二种不是**runtime-compiler**版本，因为第2步中会给this.$options.render 赋值）
    - 如果当前不是是production环境
      - 判断`vm.$options.template` 存在并且第一个字符不为#  或者  `vm.$options.el` 存在， 提醒 `You are using the runtime-only build of Vue where the template`
    - 报错提醒当前   `template or render function not defined`
  
- 执行`callHook(vm, 'beforeMount')` 生命周期钩子
  
- 定义updateComponent函数，函数中调用`vm._update(vm._render(), hydrating)，vm._render()`返回的是`vdom`,  `vm. _update `调用的是 `vm.__patch__`函数 定义在`src\platforms\web\runtime\index.js` 因为 patch与平台相关
  
    ```javascript
      import { patch } from './patch'
    Vue.prototype.__patch__ = inBrowser ? patch : noop
      ```

      

  - 创建一个渲染`Watcher`, `new Watcher(*vm*, updateComponent, noop, {

      before () {if (*vm*._isMounted && !*vm*._isDestroyed) {
  
    ​	 callHook(*vm*, 'beforeUpdate')
  
       }, true
  
    + Watcher 中会执行 updateComponent 
  
    
    
    总结：如果option没有传入render, 判断有没有template ,如果template 为# 选择器，则获取document.querySelector(template) 获取对应的html, 如果template 为element节点，则获取  template.innerHTML，如果没有template， 则 获取$mount(el) 传入的el的outerHTML,
    
    最后通过 这个html 生成 render并挂载到options.render