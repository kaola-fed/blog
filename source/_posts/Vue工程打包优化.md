<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

# Vue工程打包优化

## 一.前言
本文以考拉社区后台工程（[kaola-communityms-web](https://g.hz.netease.com/haitao/kaola-communityms-web)）为例进行分析。
首先介绍一个可视化打包分析工具——[webpack-bundle-analyzer](https://www.npmjs.com/package/webpack-bundle-analyzer)。
kaola-communityms-web已经做了如下配置：
```
    // webapp/config/index.js
    module.exports = {
      build: {
        //...
        bundleAnalyzerReport: process.env.npm_config_report
      },
    }
    
    // webapp/build/webpack.prod.conf.js
    if (config.build.bundleAnalyzerReport) {
      var BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin
      webpackConfig.plugins.push(new BundleAnalyzerPlugin())
    }
```
所以添加指令：
```
    // package.json
    {
      // ...
      "scripts":{
        // ...
        "analyz": "NODE_ENV=production npm_config_report=true npm run build"
      }
    }
```
执行指令`npm run analyz`即可启动webpack-bundle-analyzer。

其他工具如 [webpack analyse](http://webpack.github.io/analyse/)、[webpack chart](http://alexkuz.github.io/webpack-chart/)等不再做详细介绍。

## 二.为什么优化、有什么优化点
webpack打包可以总结为：
- 对于单入口文件，每个入口文件把自己所依赖的资源全部打包到一起，即使一个资源循环加载，也只会打包一份；
- 对于多入口文件，分别独立执行单个入口的情况，每个入口文件各不相干。

所以对于kaola-communityms-web（vue-cli构建，单入口），在默认情况下，执行`npm run build`会把所有js代码打包成3个文件
- static/js/vendor.[hash].js //公用模块，该工程配置为/node_modules下的文件
- static/js/app.[hash].js //所有引用到的组件js代码
- static/js/manifest.[hash].js //chunks清单

其中app.[hash].js文件在页面很多时会很大、加载缓慢（特别是在加载一些只在特定环境下才会使用到的阻塞的代码的时候），所以需要把大的文件拆分成多个小块。另外并不是每个页面都需要所有的资源，如果能只加载当前页面所需资源，那页面加载速度将得到有效提升。

对kaola-communityms-web进行本地打包，打包情况如下所示：
![](https://haitao.nos.netease.com/3e9d13d0-3a2f-4e3a-af0b-d7bee91a70e3.jpg)
![](https://haitao.nos.netease.com/58671d3f-cba4-40bc-a510-e5aa3a8a27a0.jpg)
![](https://haitao.nos.netease.com/5979f660-9d76-4412-a332-7e9c66b8f3ac.jpg)
可以看到vue-highcharts.js过大，考虑到只有一个页面用到了vue-highcharts，可以把它单独打个包；element-ui中的方法并不是所有页面都有用到，可以考虑按需加载，或单独打包；在static/js/vendor.[hash].js和static/js/app.[hash].js中均打包了element-ui，需要移除重复打包；一些没有用到的库可以移除掉。

所以可以优化点的有：

- 大文件拆分
- 按需加载
- 个别大而使用率不高的资源剥离处理
- 处理重复加载的模块

## 三.如何优化

- Vue懒加载——路由懒加载
  把不同路由对应的组件分割成不同的代码块，然后当路由被访问时才加载对应组件。结合Vue的[异步组件](https://cn.vuejs.org/v2/guide/components.html#)和Webpack的[代码分割功能（Code splitting）](https://doc.webpack-china.org/guides/code-splitting#-dynamic-imports-)，可以轻松实现路由组件的懒加载；另外，在Webpack 2中，我们可以使用[动态import](https://github.com/tc39/proposal-dynamic-import)语法来定义代码分块点。结合以上两点，定义一个能够被Webpack自动代码分割的异步组件即：
  ```
      const Foo = () => import('./Foo.vue')
  ```
  在路由配置中：
  ```
      const router = new VueRouter({
        routes: [
          { path: '/foo', component: Foo }
        ]
      })
  ```
  如果想把某个路由下的所有组件都打包在同个异步块（chunk）中，只需要使用命名chunk（一个特殊的注释语法）来提供chunk name：
  ```
      const Foo = () => import(/* webpackChunkName: "group-foo" */ './Foo.vue')
      const Bar = () => import(/* webpackChunkName: "group-foo" */ './Bar.vue')
      const Baz = () => import(/* webpackChunkName: "group-foo" */ './Baz.vue')
  ```
  Webpack会将任何一个异步模块与相同的块名称组合到相同的异步块中，即上述三个组件Foo、Bar、Baz将被打包到同一个chunk中。
- Vue懒加载——组件懒加载：只下载当前页面需要执行的代码、下载之后缓存起来以供再次渲染，这里用到了vue的[异步组件](https://cn.vuejs.org/v2/guide/components.html#)。
  ```
      new Vue({
        // ...
        components: {
          'AsyncCmp': () => import('./AsyncCmp')
        }
      })
  ```
  注：按需加载（一个模块打包一个文件），需要配置webpack的output：
  ```
      output: {
          path: config.build.assetsRoot,
          filename: utils.assetsPath('js/[name].[chunkhash].js'),
          chunkFilename: utils.assetsPath('js/[id].[chunkhash].js') //每个模块都会在打包目录下生成相应的js/[id].[chunkhash].js文件
        }
  ```
  这里有两个问题需要额外提下：
  1. 如果在两个异步加载的页面中分别同步与异步加载同一个组件，会造成资源重复加载。解决方案：团队约定都使用异步加载组件。
  2. 在异步加载页面中嵌入异步加载组件对页面渲染会产生影响。解决方案：对页面结构进行合理的设计，尽量避免首次加载闪屏现象。
- 以外源性脚本引入某些不需要改动的模块（webpack的externals）：webpack [externals](https://doc.webpack-china.org/configuration/externals/)配置选项提供了从输出的bundle中排除依赖的方法，它可以防止把某些`import`的包打包到bundle中，在运行时（runtime）才去从外部获取这些扩展依赖，比如从CND引入jQuery，而不是把它打包：
  ```
      //index.html
      <script
        src="https://code.jquery.com/jquery-3.1.0.js"
        integrity="sha256-slogkvB1K3VOkzAI8QITxV3VzpOnkeNVsKvtkYLMjfk="
        crossorigin="anonymous">
      </script>
      
      //webpack.config.js
      externals: {
        jquery: 'jQuery'
      }
      
      // my-component.vue
      import $ from 'jquery';
      $('.my-element').animate(...);//可以正常运行
  ```
  又比如项目开发中常用到的 `moment`等比较大的模块，如果必须引入的话，可以考虑外部引入，再借助 `externals` 予以指定， webpack可以处理使之不参与打包，而依旧可以在代码中通过CMD、AMD或者window/global全局的方式访问。
- webpack插件：webpack提供了一个插件 `DllPlugin` ，用于将不常变动的js单独打包。DllPlugin 和 DllReferencePlugin 提供了以大幅度提高构建时间性能的方式拆分软件包的方法。其中原理是，将特定的第三方NPM包模块提前构建，然后通过页面引入。这不仅能够使得 vendor 文件大幅度减小，同时，也极大地提高了构建速度。对于kaola-communityms-web，先创建webpack.dll.conf.js、配置需要单独打包的模块，在webpack.prod.conf.js中添加DllReferencePlugin插件配置，在webpack.base.conf.js中添加externals，删掉代码中对这些模块的多余的引用，执行指令即可。
  ```
      // webpack.dll.conf.js
      // ...
      module.exports = {
          // 你想要打包的模块的数组
        entry: {
          vuecharts:['vue2-highcharts'],
          vendor: ['vue', 'lodash', 'vuex', 'axios', 'vue-router', 'element-ui','moment']
        },
        plugins: [
          new webpack.DllPlugin({
            path: path.join(__dirname, '.', '[name]-manifest.json'),
            name: '[name]_library',
            context: __dirname
          })
          // ...
        ]
        // ...
      }
      
      // webpack.prod.conf.js
      // ...
      plugins: [
        new webpack.DllReferencePlugin({
          context: __dirname,
          manifest: merge(require('./vuecharts-manifest.json'),require('./vendor-manifest.json'))
        })
        // ...
      ]
      
      // webpack.base.conf.js
      // ...
      externals: {
        'vue': 'Vue',
        'vue-router': 'VueRouter',
        'elemenct-ui': 'ELEMENT',
        'lodash':'lodash',
        'vuex':'vuex',
        'axios':'axios',
        'vue-highcharts':'vue2-highcharts',
        'moment':'moment'
      }
      
      // 添加指令
      // package.json
      "dll": "webpack --config ./build/webpack.dll.conf.js"
  ```
  执行`npm run dll`后生成两个manifest配置文件（vendor-manifest.json、vuecharts-manifest.json）和两个打包文件：
  ![](https://haitao.nos.netease.com/f2631b68-222b-4a86-916a-7cef744dec6c.jpg)
  鉴于篇幅，具体用法可见：[webpack.dll.conf.js](https://github.com/nicejade/vue-boilerplate-template/blob/master/build/webpack.dll.conf.js)、[externals&Dll Reference](https://rawidn.com/posts/webpack-in-vue-development.html)。
  [happypack](http://taobaofed.org/blog/2016/12/08/happypack-source-code-analysis/)可以加速代码构建。
  webpack提供的UglifyJS插件由于采用单线程压缩，速度很慢，[webpack-parallel-uglify-plugin](https://www.npmjs.com/package/webpack-parallel-uglify-plugin/tutorial)插件可以并行运行UglifyJS插件，可以有效减少构建时间(UglifyJsPlugin 不仅可以将未使用的 exports 清除，还能去掉很多不必要的代码，如无用的条件代码、未使用的变量、不可达代码等)。
- 移除组件中没有用到的引入；
- 不生成.map文件
  ```
      // config/index.js
      module.exports = {
        build: {
          // ...
          productionSourceMap: false
        }
  ```

- 另外：webpack3新功能Scope Hoisting（作用域提升），只需在配置文件中添加一个新的插件，就可以让 Webpack 打包出来的代码文件更小、运行的更快。参见 Webpack 3 的新功能：[Scope Hoisting](https://zhuanlan.zhihu.com/p/27980441)。

## 四.效果

优化后的打包情况：
![](https://haitao.nos.netease.com/f9743bad-48a9-45d1-8d58-844c1ea51e15.jpg)
![](https://haitao.nos.netease.com/23f0f324-aee3-4f1c-9cf2-074ea7332b92.jpg)
![](https://haitao.nos.netease.com/b09763ba-3983-4ed9-a617-815269db2817.jpg)
![](https://haitao.nos.netease.com/401a5aac-bd5a-47d8-ab86-90e5cc4cce7e.jpg)

优化后的首页资源加载情况：
![](https://haitao.nos.netease.com/ed7aa167-c3e5-4f88-a599-eca54ab925d3.jpg)

可以看到优化后打包时间从29418ms减少到了11998ms，打包资源大小从1.53MB减少到了367.87KB，并且实现了按需加载（没有加载所有资源）。

## 五.结束语

性能优化范围广、方法多，还有很多可以提升打包速度、打包文件大小、首屏加载等性能问题的方法，本文只是对部分方法进行学习总结，还需要不断学习、研究更多方案。

## 六.参考文献

1. https://segmentfault.com/q/1010000012211052/a-1020000012211401
2. https://segmentfault.com/a/1190000008377195
3. http://blog.csdn.net/qq_16559905/article/details/78551719
4. https://github.com/hehongwei44/my-blog/issues/203
5. https://router.vuejs.org/zh-cn/advanced/lazy-loading.html
6. https://router.vuejs.org/en/advanced/lazy-loading.html
7. https://segmentfault.com/a/1190000011519350
8. https://alexjoverm.github.io/2017/07/16/Lazy-load-in-Vue-using-Webpack-s-code-splitting/
9. https://zhuanlan.zhihu.com/p/29433875
10. https://www.zhihu.com/question/41147233
11. https://cn.vuejs.org/v2/guide/components.html#%E5%BC%82%E6%AD%A5%E7%BB%84%E4%BB%B6
12. http://blog.csdn.net/qq_27626333/article/details/76228578
13. http://zakwu.me/2016/09/18/vue-routerpei-zhi/
14. https://kinm.github.io/2017/08/24/VUE%E7%BB%84%E4%BB%B6%E4%BC%98%E5%8C%96%E7%AD%96%E7%95%A5%EF%BC%88%E6%8C%89%E9%9C%80%E5%8A%A0%E8%BD%BD%E3%80%81%E8%B7%AF%E7%94%B1%E6%87%92%E5%8A%A0%E8%BD%BD%EF%BC%89/
15. https://github.com/eyasliu/blog/issues/8
16. https://www.cnblogs.com/zhanyishu/p/6587571.html
17. https://www.jianshu.com/p/171e8e529f35
18. https://jeffjade.com/2017/08/06/124-webpack-packge-optimization-for-volume/
19. http://imweb.io/topic/597f47c790ccc00402bb1820
20. https://segmentfault.com/a/1190000012220132
21. https://doc.webpack-china.org/configuration/externals/
22. https://juejin.im/post/5a337a1f6fb9a0452b4949e0



by Frida




