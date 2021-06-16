# Spa应用

**顺序:**  客户端发送请求，获取到html, 执行vue再获取页面数据

1. 首屏渲染速度慢
2. SEO不友好

# 传统服务端

**顺序:**  客户端发送请求，获取到html渲染到页面上



# SSR

请求后台，后台vue模板解析html,查库等异步操作，返回完整的html，首屏渲染客户端相关代码，转换为spa,

**缺点：**

1. 发条件所限。浏览器特定的代码，只能在某些生命周期钩子函数 (lifecycle hook) 中使用；一些外部扩展库 (external library) 可能需要特殊处理，才能在服务器渲染应用程序中运行。
2. 涉及构建设置和部署的更多要求。与可以部署在任何静态文件服务器上的完全静态单页面应用程序 (SPA) 不同，服务器渲染应用程序，需要处于 Node.js server 运行环境。
3. 更多的服务器端负载。在 Node.js 中渲染完整的应用程序，显然会比仅仅提供静态文件的 server 更加大量占用 CPU 资源 (CPU-intensive - CPU 密集)，因此如果你预料在高流量环境 (high traffic) 下使用，请准备相应的服务器负载，并明智地采用缓存策略。



创建一个项目

```
vue create vue-ssr
```

**安装依赖**

渲染器: vue-server-renderer

node.js服务器express

```bash
npm i vue-server-renderer express -D
```

## server

createBundleRenderer ,基于服务端bundle与客户端bundle创建renderer, 通过webpack从入口entry-server与entry-client打包成服务端bundle与客户端bundle

```js
const express = require('express')
const app = express()
const { createBundleRenderer } = require('vue-server-renderer')
const serverBundle = require('../dist/server/vue-ssr-server-bundle.json');  
const clientManifest = require('../dist/client/vue-ssr-client-manifest.json');
const fs = require('fs')
const path = require('path')
const renderer  = createBundleRenderer(serverBundle, {
    runInNewContext: false,
    template: fs.readFileSync(path.join(__dirname, '../public/index.temp.html'), 'utf-8'),// 宿主模板
    clientManifest
});
// 中间件处理静态文件请求
app.use(express.static(path.join(__dirname, '../dist/client/'), {index: false}))


app.get('*',async (req, res) => {
    // 每次请求都创建一个新的vue实例
    const context = { 
        url: req.url,
        title: 'ssr test'
     }
    try {
        const html = await renderer.renderToString(context);
        // eslint-disable-next-line no-console
        // console.log(html);
        res.send(html);
    } catch (error) {
        res.status(500).end('Internal Server Error')
    }
})

app.listen(8080,() => {
    console.log("http://127.0.0.1:8080")
})
```

clientManifest中描述了首页需要加载的资源，

**clientBunder中的资源需要在服务器配置为静态资源文件，并排除index.html,不然页面不会显示完整的代码**

app.use(express.static(path.join(__dirname, '../dist/client/'), {index: false}))



```
{
  "publicPath": "/",
  "all": [
    "favicon.ico",
    "index.html",
    "index.temp.html",
    "js/chunk-2d0decc6.41dd9f47.js",
    "js/chunk-2d0decc6.41dd9f47.js.map",
    "js/chunk-6612c489.ed0ea55c.js",
    "js/chunk-6612c489.ed0ea55c.js.map",
    "js/chunk-vendors.969dba2b.js",
    "js/chunk-vendors.969dba2b.js.map",
    "js/main.76e68c71.js",
    "js/main.76e68c71.js.map"
  ],
  "initial": [
    "js/chunk-vendors.969dba2b.js",
    "js/main.76e68c71.js"
  ],
  "async": [
    "js/chunk-2d0decc6.41dd9f47.js",
    "js/chunk-6612c489.ed0ea55c.js"
  ],
  "modules": {
    "49826470": [
      7,
      8
    ],
    "72833887": [
      7,
      8
    ],
  	 .......
  }
}
```

使用 vue-server-renderer 将vue实例编译成html,并返回，注意``每次请求都会创建一个新的vue实例`，同时每次只会运行`beforeCreate` 与 `created` 钩子

**注意点：** 

1. 避免在`beforeCreate` `created` 中创建全局副作用代码，如定时器，因为服务端不会执行destroyed钩子，所以会一直存在。
2. 不要使用特定平台的APi，如Window、document等
3. vue自定义指令,使用特定服务端版本



对于路由，状态管理，eventbus实例都需要在 `createApp` 中创建一个新的实例，并从根 Vue 实例注入。

## clientEntry

```js
import { createApp } from './main';


// 这里假定 App.vue 模板中根元素具有 `id="app"`
const { app, router, store } = createApp()

if (window.__INITIAL_STATE__) {
    store.replaceState(window.__INITIAL_STATE__)
}
// 需要在挂载 app 之前调用 router.onReady，因为路由器必须要提前解析路由配置中的异步组件，才能正确地调用组件中可能存在的路由钩子
    router.onReady(() => {
    // 添加路由钩子函数，用于处理 asyncData.
    // 在初始路由 resolve 后执行，
    // 以便我们不会二次预取(double-fetch)已有的数据。
    // 使用 `router.beforeResolve()`，以便确保所有异步组件都 resolve。
    router.beforeResolve((to, from, next) => {
        const matched = router.getMatchedComponents(to)
        const prevMatched = router.getMatchedComponents(from)
    
        // 我们只关心非预渲染的组件
        // 所以我们对比它们，找出两个匹配列表的差异组件
        let diffed = false
        const activated = matched.filter((c, i) => {
          return diffed || (diffed = (prevMatched[i] !== c))
        })
    
        if (!activated.length) {
          return next()
        }
    
        // 这里如果有加载指示器 (loading indicator)，就触发
        Promise.all(activated.map(c => {
          if (c.asyncData) {
            return c.asyncData({ store, route: to })
          }
        })).then(() => {
          // 停止加载指示器(loading indicator)
          next()
        }).catch(next)
    })

    // 客户端代码，将内容挂载到dom上
  app.$mount('#app')
})
```



## entry-server

```js
import { createApp } from './main';

// 服务端，创建与返回应用实例，每个用户请求创建一个新的应用程序实例
export default context => {
    // 因为有可能会是异步路由钩子函数或组件，所以我们将返回一个 Promise，
    // 以便服务器能够等待所有的内容在渲染前，
    // 就已经准备就绪。
    return new Promise((resolve, reject)=> {
        const { app, router, store } = createApp();
        // 设置服务器端 router 的位置
        router.push(context.url)

        // 等到 router 将可能的异步组件和钩子函数解析完
        router.onReady(() => {
            const matchedComponents = router.getMatchedComponents()
            // 匹配不到的路由，执行 reject 函数，并返回 404
            if (!matchedComponents.length) {
             return reject({ code: 404 })
            }
             // 对所有匹配的路由组件调用 `asyncData()`
            Promise.all(matchedComponents.map(Component => {
                if (Component.asyncData) {
                return Component.asyncData({
                    store,
                    route: router.currentRoute
                })
                }
            })).then(() => {
                // 在所有预取钩子(preFetch hook) resolve 后，
                // 我们的 store 现在已经填充入渲染应用程序所需的状态。
                // 当我们将状态附加到上下文，
                // 并且 `template` 选项用于 renderer 时，
                // 状态将自动序列化为 `window.__INITIAL_STATE__`，并注入 HTML。
                context.state = store.state

                resolve(app)
            }).catch(reject)
            // Promise 应该 resolve 应用程序实例，以便它可以渲染
            resolve(app)
        }, reject)

    })
}
```

## createApp

将router与store挂载到vue实例上，每次访问都返回不同的实例及路由与状态

```js
import Vue from 'vue';
import App from './App.vue';
import { createRouter } from './router';
import { createStore } from './store/index';
import { sync } from 'vuex-router-sync'
Vue.config.productionTip = false
export function createApp(){
  const router = createRouter();
  const store = createStore();
    // 同步路由状态(route state)到 store
  sync(store, router)
  const app =new Vue({
    router,
    store,
    beforeCreate(){
        // 页面请求时会触发
        console.log('beforeCreate')
    },
    created(){
        console.log('created')
    },
    beforeMount(){
        // 不运行
        console.log('beforeMount')
    },
    render: h => h(App),
  })
  return  { app, router, store  }
}

```

## 路由

安装 vue-router

```bash
 npm install vue-router

```

创建createRouter，工厂函数。

```js
import Vue from 'vue';
import Router from 'vue-router';
Vue.use(Router);
export function createRouter() {
    return new Router({
        mode: 'history',
        routes:[
            {
                path: '/',
                component: ()=> import('./components/Index.vue')
            },
            {
                path: '/hello',
                component: ()=> import('./components/HelloWorld.vue')
            },
        ]
    })
} 
```



## createStore

vuex全局状态

```js
// store.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

// 假定我们有一个可以返回 Promise 的
// 通用 API（请忽略此 API 具体实现细节）
function fetchItem() {
    return new Promise((resolve) => {
        setTimeout(()=> (resolve('data ewsolve')), 1000)
    })
}

export function createStore () {
  return new Vuex.Store({
    state: {
      items: {}
    },
    actions: {
      fetchItem ({ commit }, id) {
        // `store.dispatch()` 会返回 Promise，
        // 以便我们能够知道数据在何时更新
        return fetchItem(id).then(item => {
          commit('setItem', { id, item })
        })
      }
    },
    mutations: {
      setItem (state, { id, item }) {
        Vue.set(state.items, id, item)
      }
    }
  })
}
```

组件中提供asyncData，在entryServer中router.onReady时调用

```js
export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  asyncData ({ store }) {
    store.registerModule('foo', fooStoreModule)
    return store.dispatch('foo/inc')
  },
  // 重要信息：当多次访问路由时，
  // 避免在客户端重复注册模块。
  destroyed () {
    this.$store.unregisterModule('foo')
  },
  computed: {
    fooCount () {
      return this.$store.state.foo.count
    }
  }
}
```



# 预渲染

如果你调研服务器端渲染 (SSR) 只是用来改善少数营销页面（例如 `/`, `/about`, `/contact` 等）的 SEO，那么你可能需要**预渲染**。无需使用 web 服务器实时动态编译 HTML，而是使用预渲染方式，在构建时 (build time) 简单地生成针对特定路由的静态 HTML 文件。优点是设置预渲染更简单，并可以将你的前端作为一个完全静态的站点。

如果你使用 webpack，你可以使用 [prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin) 轻松地添加预渲染。