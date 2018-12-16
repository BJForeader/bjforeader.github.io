---
title: vue预渲染
date: 2018-12-12 23:03:40
categories:

- 前端

tags:

- vue

photos:
- https://ws1.sinaimg.cn/large/006tNbRwly1fy652lkaksj308006t0sk.jpg
---
## 前言
 Ajax 技术的出现，让我们的 Web 应用能够在不刷新的状态下显示不同页面的内容。在一个单页应用中，往往只有一个 html 文件，然后根据访问的 url 来匹配对应的路由脚本，动态地渲染页面内容。单页应用在优化了用户体验的同时，也给我们带来了许多问题，例如 SEO 不友好、首屏可见时间过长等。服务端渲染（SSR）和预渲染（Prerender）技术正是为解决这些问题而生的。
 
## 服务器端渲染 vs 预渲染
客户端渲染：用户访问 url，请求 html 文件，前端根据路由动态渲染页面内容。关键链路较长，有一定的白屏时间；

服务端渲染：用户访问 url，服务端根据访问路径请求所需数据，拼接成 html 字符串，返回给前端。前端接收到 html 时已有部分内容；[服务端渲染介绍](https://ssr.vuejs.org/zh/)

预渲染：构建阶段生成匹配预渲染路径的 html 文件（注意：每个需要预渲染的路由都有一个对应的 html）。构建出来的 html 文件已有部分内容。

针对单页应用，服务端渲染和预渲染共同解决的问题：
1. SEO：单页应用的网站内容是根据当前路径动态渲染的，html 文件中往往没有内容，网络爬虫不会等到页面脚本执行完再抓取；
2. 弱网环境：当用户在一个弱环境中访问你的站点时，你会想要尽可能快的将内容呈现给他们。甚至是在 js 脚本被加载和解析前；
3. 低版本浏览器：用户的浏览器可能不支持你使用的 js 特性，预渲染或服务端渲染能够让用户至少能够看到首屏的内容，而不是一个空白的网页。

## Prerender SPA Plugin 
由于我们的项目是使用webpack + vue-cli构建的，也就是说这是个用webpack进行打包的单页应用。
在路由方面选择的是vue-router，mode是hash模式。使用这个[prerender-spa-plugin](https://www.npmjs.com/package/prerender-spa-plugin)配置起来比较简单

##### 配置Prerender SPA Plugin 
1. 安装需要将代理设置为全局 因为关联Puppeteer 需要下载Chromium否则会一直处于node install.js 安装失败
  ```angular2html
    npm install prerender-spa-plugin --save-dev
```
2. 配置webpack
   在webpack中配置 prerender-spa-plugin
   配置先弄懂要配置在哪个文件里，配置是否生效。 vue-cli2 的配置文件很多，对这些文件不了解的话，很容易配置错地方。
   这个配置只需要在 build 的时候可以生成预渲染好的html，所以应该配置在 build/webpack.prod.conf.js 这个文件里。
   [vue-cli配置文件详解](https://blog.csdn.net/Mr_YanYan/article/details/79233188)
```
    //引入prerender-spa-plugin插件
    const PrerenderSPAPlugin = require('prerender-spa-plugin')
    const Renderer = PrerenderSPAPlugin.PuppeteerRenderer
    //在文件找到plugins 属性 放到最后
    plugins:[
            new PrerenderSPAPlugin({
                // 生成文件的路径，也可以与webpakc打包的一致。
                // 这个目录只能有一级，如果目录层次大于一级，在生成的时候不会有任何错误提示，在预渲染的时候只会卡着不动。
                staticDir: path.join(__dirname, '../dist'),
               
                // 对应自己的路由文件，比如index有参数，就需要写成 /index/param1。
                routes: ['/home', ],
                
                // 这个很重要，如果没有配置这段，也不会进行预编译
                renderer: new Renderer({
                    inject: {
                      foo: 'bar'
                    },
                    headless: false,
                    // 在 main.js 中 document.dispatchEvent(new Event('render-event'))，两者的事件名称要对应上。
                    renderAfterDocumentEvent: 'render-event'
                })
            })
    ]
```
3. 在main.js中加入 document.dispatchEvent(new Event('render-event'))
```
new Vue({
  el: '#app',
  created() {
    //解决页面刷新数据丢失的问题
    //在页面加载时读取sessionStorage里的状态信息
    if (sessionStorage.getItem("store")) {
      this.$store.replaceState(Object.assign({}, this.$store.state, JSON.parse(sessionStorage.getItem("store"))))
      sessionStorage.removeItem("store")
    }
    //在页面刷新时将vuex里的信息保存到sessionStorage里
    window.addEventListener("beforeunload", () => {
      sessionStorage.setItem("store", JSON.stringify(this.$store.state))
      sessionStorage.setItem("CHANNL_ID", CHANNL_ID)
    })
  },
  mounted(){
    document.dispatchEvent(new Event('render-event'))
  },
  components: {App},
  template: '<App/>',
  router,
  store
})
```
4. 将路由改成history模式
```
export default new Router({
  mode:'history',
  routes,})
```
5. 执行 npm run build
 一切顺利的话将在 dist 目录 下生成 在webpack.prod.conf.js配置了路由的文件夹 例如我配置的 /home
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fy4ephqmaxj30y80auq35.jpg)
home文件下存在index.html
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fy4eqjv7caj311e0c274g.jpg)
 查看源码已经预渲染ok了
 ![](https://ws2.sinaimg.cn/large/006tNbRwgy1fy4esdgtuoj325l0u0aj0.jpg)
 没有预渲染的页面
 ![](https://ws1.sinaimg.cn/large/006tNbRwgy1fy4euu1kuij326s0js42r.jpg)
 对比我们未预渲染的页面，我们预渲染页面中生成了dom结构让我们访问更快
6. 改变服务器访问的地址，判断访问的路由是否预渲染了，如果是 直接访问预渲染生成的index.html 否则保持原样访问根html
## 爬坑 
 * 页面重新加载<br>
 查看文档需要在根节点上也就是 id=app 添加  data-server-rendered="true"属性 告诉vue 这个页面不需要再重新渲染了
 * 只加载一屏幕图片<br>
 是由于我们的页面是使用了图片懒加载的问题，在预渲染的时候只生成了一屏幕的图片，其他图片的地址放到了data-src中。我们预渲染的dom已经生成了 不会再请求图片地址了
 去掉图片属性的 v-lazy 改成 :src 直接引用地址  
 * 生成的页面在本地服务器调试没有问题，上测试服务器后页面不能点击(未解决)
 [页面点击问题，参考文档](https://github.com/chrisvfritz/prerender-spa-plugin/issues/24)
 
## 参考
[How to Pre-render Vue.js Powered Websites With webpack](https://markus.oberlehner.net/blog/how-to-pre-render-vue-powered-websites-with-webpack/)<br>

[prerender-spa-plugin github](https://github.com/chrisvfritz/prerender-spa-plugin) 
 
  
