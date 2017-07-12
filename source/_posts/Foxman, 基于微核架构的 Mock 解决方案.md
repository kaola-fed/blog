---
title:  Foxman, 基于微核架构的 Mock 解决方案
date: 2017-05-26
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
[Foxman ⇗](https://github.com/kaola-fed/foxman) 是一个使用 Node.js 开发的命令行工具，定位是一个可扩展的 Mock Server，帮助前端开发者轻松、独立、高效地完成前端开发和联调工作。

他不是一款静态文件响应工具， 假如你只需要一款轻量的 Node.js 开发服务器，推荐你使用 [puer ⇗](https://github.com/leeluolee/puer) 或 [webpack-dev-server ⇗](https://github.com/webpack/webpack-dev-server)。

github 地址:  [https://github.com/kaola-fed/foxman](https://github.com/kaola-fed/foxman)

![](https://haitao.nos.netease.com/056a58c4-0241-4947-a609-9a4bd657b52c.png)

<!-- more -->

## 背景

作为前端开发的我们，在实际的开发场景中会遇到以下问题：

1. 环境：进行本地开发，需要起后端环境(Tomcat)，对于新人来说，需要大量时间熟悉；熟练的人遇到某些确实存在的问题，也要花时间去解决，耗费大量前端开发的时间；
2. 流程：前端开发先开发 html，再将 html 改写成指定的模板语法，影响开发效率；
3. 接口：
    * 接口定义使用聊天工具发送，前端开发时不好理解接口字段，影响开发效率；
    * 接口变更需要重新编写文档，并重新发送，影响开发效率；
    * 文档散落，影响接口维护；
4. 联调：
    * 联调过程很复杂，尤其是没有做热部署的Java工程，改视图还需要重启Tomcat，影响前端联调效率；
5. 效益：
    * 前后端对接的方式，期望纯粹的 JSON 交换。不过现实情况，是依赖后端的模板引擎，导致前端理解接口存在一定的障碍；

以上问题的存在，才产生了 Foxman 这个项目。

## 影响
从 *[考拉前端](https://blog.kaolafed.com/)* 使用情况来看，在接入 Foxman 后，拥有了更优雅的开发体验和更高的开发效率：
1. 前端开发者不再需要在本地起 Tomcat 服务，新人也无需熟悉本地启动环境；而启动一个 Foxman 所需的时间，在 2s 以内；
2. 接口定义，前端开发者更加有意识地去与后端定义接口，因为接口定义会落实到具体的 Mock 数据上；
3. Mock 功能，使得前端开发者在开发阶段几乎可以是自治、无打扰的情况（产品不改需求的前提下）；
4. Living Reload 的功能， 页面开发过程中，修改 html 和 js 会通知浏览器 reload 页面；修改 css 会通知浏览器只 reload 样式，提升了开发体验，节省了人肉刷新耗费的时间。
5. Processors 的功能， 即时编译的设定，更好地结合无 webpack 构建的场景；
6. Proxy 功能，更优雅的联调方式。前端开发者，可以在本地调试模板和 javascript，避免了修改提交，再重新部署服务器的时间耗费。


## 核心概念
**容器** - Foxman 核心提供了一个挂载插件的容器，并且提供方法供插件提供或调用的服务。实现上，使用了IoC（依赖查找）、插件化等架构设计的思想。

**插件** - Foxman 所有具体的功能都使用插件实现。插件的作用是实现本身需求，并提供服务供其他插件使用。

**服务** - 服务是架设于 容器 与 插件 之上的概念，容器 提供方法供 插件 注册或调用服务。

在这样的体系下，你可以轻松地编写 Foxman 的插件，并调用已有插件的服务。所以，完全不需要担心，Foxman 会不适合你的项目，因为你完全可以根据自己的需求来定制你所需要的Foxman。

## 安装
NPM
```bash
$ npm i -g foxman@lastest # 无梯子用户，推荐使用 cnpm
```

⚠️  Foxman 采用 es6 语法的大部分特性编写，要求使用 Node.js 版本不低于 v6.4.0

## 编写配置文件
```javascript
module.exports = {
    port: 9000,
    secure: false,
    statics: [
        './src/'
    ],
    routes: [
        {
            method: 'GET',
            url: '/ajax/index.html',
            sync: false,
            filePath: 'foo.bar'
        }
    ],
    engineConfig: {},
    viewRoot: './views/',
    extension: 'ftl',
    syncData: './syncData/',
    asyncData: './ajax/',
    plugins: [],
    processors: [
        { match: '/src/css/*.css', pipeline: [], locate( reqUrl ) {} }
    ],
    proxy:  [
        { name: 'pre', host: 'm.kaola.com', ip: '1.1.1.1', protocol: 'http' }
    ]
}
```

这是一份基础的 Foxman 的配置文件，可以发现大部分字段都给 Server 用的，比如：

* port - Server 监听的端口
* secure - 是否启用 https
* statics - 静态资源配置
* routes - 路由列表
* engineConfig - 模板引擎配置项
* viewRoot - 模板根目录
* extension - 模板扩展名
* syncData - 同步数据根目录
* asyncData - 异步数据根目录

以及一些特殊的字段，后面我们会重点介绍：
* proxy - 联调阶段，同步数据与异步数据的转发至后端主机或测试服务器
* processors - Runtime Compiler 的配置
* plugins - 插件配置

更详细的 Foxman 配置，[点击此处 ⇗ ](https://github.com/kaola-fed/foxman/blob/master/example/foxman.config.js)

## 启动

在编写完 foxman.config 的目录下，执行命令即可启动 Foxman ：

```bash
$ foxman
```

## 设计理念
### 插件体系
Foxman 的外置插件可以在配置文件中灵活载入:
```
...
plugins: [
        new RouteDisplay(),
        new MockControl({}),
        new Automount({}),
        new WebpackDevServer({}),
]     
...
```
而所有的内置功能，其实也是依托于插件展开。每个 Foxman 插件，需要实现一些方法，用于装载入 Foxman 容器时，做一些登记工作：

```javascript
class Plugin {
    constructor() {
        // 初始化自身需要的属性
    }
    
    name() { // 定义插件的名字，如果没有该字段，会使用 constructor.name
        return 'name';
    }
    
    service() { // 提供给其他插件的服务
        return {
            foo() {
                return 'bar';
            }
        }
    }

    init({getter, service}) {
        const use = service('service.use'); 
    }
}
```
[LivereloadPlugin ⇗ ](https://github.com/kaola-fed/foxman/blob/master/packages/foxman-plugin-livereload/lib/index.js)

### 容器与依赖查找
容器的设定，离不开 IoC（控制反转）的概念。

实现 IoC，惯用的一种方案是依赖注入 (Dependency Injection) ，用于运行时被动地接收依赖的对象，早期的 Foxman 是根据 DI 的方式实现插件化的；

另一种方案是依赖查找 (Dependency Lookup) - 与 DI 相比更加主动，主动得调用框架提供的方法来获取依赖，获取时提供相关的配置文件路径 或 keypath 等信息。

Foxman 核心提供了 use 和 start 两个方法：
* use - 注册 Plugin 及 service
* start - 执行 Plugin 的 init 方法，传入 service/getter 方法，供其依赖查找

```javascript
// core.js
class Core {
    use() {
        // 1. 注册 Plugin 进入容器;
        // 2. 在容器中登记 Plugin 提供的 service 
    } 

    start() {
        // 1. 循环 Plugin 执行 init 方法， 注入 getters, service 等方法，用于获取其他插件的配置或是服务
        // 2. 如果插件执行了 this.pending 方法，则等待异步操作完成。
    } 
}
```

```javascript
// app.js
const core = new Core();

core.use(new Plugin({})); 
// 1. 执行 Plugin constructor
// 2. 注册 Plugin 进入容器
// 3. 在容器中登记 Plugin 提供的 service 

core.start(); 
// 执行 Plugin 的 init 方法，会在参数中注入的 getters 和 service 方法，用于插件依赖查找，
```

具体的实现细节，感兴趣的同学可以 [查看源码 ⇗ ](https://github.com/kaola-fed/foxman/tree/master/packages/foxman-core)

## 功能模块
### Server模块 
> 基于 Node.js Server 框架 koa@1.x 构建，Server 的职责便是渲染模板、响应异步数据，以及在页面插入一些特定的脚本。

整个 Server 的启动分为三个阶段：
1. 初始化 - 设置配置，设置路由，以及初始化 Koa 对象；
2. 装载中间件 - 初始化中间件队列；
3. 启动服务 - 启动 Server，并建立 websocket 服务器，用于与浏览器的通信。

在 Server 启动后，请求进入 Server 时，会经历中间件的处理，这个过程又能分为 3 个阶段：
1. 请求分析，及确定响应方式，在请求的 context 上，生成 dispatcher 对象，用于在步骤 3 中确定以何种方式进行响应（同步 or 异步，模板路径 or mock 数据路径）；
2. 由插件装载的中间件对请求进行处理（取决于具体使用的插件），这个阶段可以对 dispatcher 对象进行修改，以完成插件所期望的渲染方式；
3. 请求响应，根据请求的类型，分为以下几种方式
    * 同步请求 - 交给 Foxman-Engine 进行渲染，（注入一些 script 脚本，并且在页面上追加同步数据，使得浏览器 console 中输入 window.__FOXMAN_SYNC_DATA__ 即可获得 ）
    * 异步请求 - 默认 json 方式展示，如需要 jsonp 响应，或是要自由控制响应方式，请使用插件 [@foxman/plugin-mockcontrol ⇗ ](https://github.com/kaola-fed/foxman/tree/master/packages/foxman-plugin-mockcontrol)
    * 文件夹请求 - 展示文件夹内的文件列表
    * 静态资源 - 响应静态资源

Server模块 提供其他插件一些关于 Server 相关的服务，可以供其他插件调用，比如：
* injectScript - 允许其他插件在同步接口中插入 javascript 脚本
* eval - 允许其他插件执行 js 代码
* livereload - 允许其他插件通知浏览器 reload 
* use - 允许其他插件给 server 加入中间件
* registerRouterNamespace - 允许其他插件新增路由，使用命名空间可以保证不同的插件的路由不会相互干扰

Foxman 的内置的 Mock Data 编写方式使用最原始的 JSON 字符串。  
有特殊需求可以使用插件 [@foxman/plugin-mockcontrol ⇗ ](https://github.com/kaola-fed/foxman/tree/master/packages/foxman-plugin-mockcontrol) 对响应进行额外控制，也可以在此基础上使用 MockJS，对响应数据进行控制。

### Template 渲染引擎
> 模板解析模块，具有特定接口，完成模板渲染需求。

```javascript
var engine = require('@foxman/engine-arttemplate');

...
engine: engine,
engineConfig: { // 取决于具体的模板引擎
    bail: true,
    compileDebug: true,
    imports: renderImports,
    debug: false,
    cache: false,
}
...
```

目前支持的模板引擎有：
* Freemaker - [@foxman/engine-freemarker ⇗ ](https://github.com/kaola-fed/foxman/blob/master/packages/foxman-engine-freemarker/)
* Art-template - [@foxman/engine-arttemplate ⇗ ](https://github.com/kaola-fed/foxman/blob/master/packages/foxman-engine-arttemplate/)
* Handlebars - [@foxman/engine-handlebars ⇗ ](https://github.com/kaola-fed/foxman/blob/master/packages/foxman-engine-handlebars/)

假如没有你需要的，你也可以自行开发一款 Foxman 的模板引擎解析器，只需要实现一个特定的接口，基本结构如下。

```javascript
const template = require('xxx-template');
class TemplateEngin {
    constructor(viewRoot, engineConfig) {
        // 初始化配置
    }
    
    parse(path, mockData) {
        // 返回一个 Promise，Promise 的返回是处理后的接口
        return Promise.resolve(template(path, mockData));
    }
}
```
具体实现，参考 [@foxman/engine-arttemplate ⇗ ](https://github.com/kaola-fed/foxman/blob/master/packages/foxman-engine-arttemplate/lib/index.js)

### Proxy
> 使用本地的模板，结合远程端的数据来拼装页面。

代理的原理：
1. Foxman 接收到用户的代理需求时，将请求转发给后台服务器，并带上特殊的请求头（X-Special-Proxy-Header: foxman）;
2. 后端接收到 Foxman 的代理请求后，要求以 JSON 的方式将页面的同步数据返回；
3. Foxman 接收到服务端的响应数据后，结合本地的模板来实现模板渲染的需求，并响应给用户。

代理的设定，使得我们可以在本地的环境下调试测试环境的场景，发现存在前端的 bug 也能轻松修复，不再需要重复的部署测试服务器。

来接触下 Foxman Proxy 的实际配置:
```javascript
...
proxy:  [{ 
    name: 'pre', 
    host: 'm.kaola.com',  // 用于 nginx 转发到制定应用
    ip: '1.1.1.1',        // 目标的 IP 地址
    protocol: 'http'      // 协议
}]
...
```

完成上述配置后，使用者输入以下命令启动 Foxman，即可代理至远程服务器
```bash
$ foxman -P pre # pre 为 配置的 proxy name
```

### Processors
> Processors 是 Runtime Compiler 的设定，在接收到静态资源请求时，才去即时地编译前端资源（sass/less/mcss/autoprefixer），主要目的是兼容无 webpack 构建的开发场景。如已使用 webpack，则推荐使用插件 [@foxman/plugin-webapck-dev-server](https://github.com/kaola-fed/foxman/tree/master/packages/foxman-plugin-webpackdevserver)

举例介绍 mcss 的即时编译配置
```javascript
const Mcss = require('@foxman/processor-mcss');
const AutoPrefixer = require('@foxman/processor-autoprefixer');

...
processors: [
    {
        match: '/src/css/**.css', // 拦截该请求
        pipeline: [ // pipe 式的处理
            new Mcss({
                paths: []
            }),
            new AutoPrefixer({
                cascade: false,
                browsers: '> 5%'
            })
        ],
        locate(reqPath) { // 根据请求路径，定位到在系统中具体路径
            return path.join(__dirname + reqPath.replace(/css/g, 'mcss'));
        }
    }
],
...
```

假如没有你需要的，你也可以自行开发一款 Foxman 的 Processor ，只需要实现一个特定的接口，基本结构如下：
```javascript
const mcss = require('mcss');

class Processor {
    constructor(options) {
        // 初始化解析器的参数
    }

    locate(reqPath) { // 根据请求路径找到文件在系统中的位置
        return reqPath.replace(/\.css$/g, '\.mcss');
    }

    *handler({ raw, filename }) {
        return yield new Promise((resolve, reject) => {
            // 在这里 进行 parse 操作，如 sass | less 的解析操作
            return {
                dependencies, content 
                // 该文件依赖，及内容
            }
        })
    }
}
```
具体实现，参考 [@foxman/processor-mcss ⇗ ](https://github.com/kaola-fed/foxman/blob/master/packages/foxman-processor-mcss/index.js)

## 关于贡献者

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
| [<img src="https://avatars3.githubusercontent.com/u/10825163?v=3" width="80px;"/><br /><sub>君羽</sub>](https://github.com/imhype)<br />[💻](https://github.com/kaola-fed/foxman/commits?author=ImHype "Code") [🔌](#plugin-ImHype "Plugin/utility libraries") [🚇](#infra-ImHype "Infrastructure (Hosting, Build-Tools, etc)") [📖](https://github.com/kaola-fed/foxman/commits?author=ImHype "Documentation") [⚠️](https://github.com/kaola-fed/foxman/commits?author=ImHype "Tests") [🐛](https://github.com/kaola-fed/foxman/issues?q=author%3AImHype "Bug reports") [💡](#example-ImHype "Examples") | [<img src="https://avatars3.githubusercontent.com/u/9125255?v=3" width="80px;"/><br /><sub>MO</sub>](https://github.com/fengzilong)<br />[💻](https://github.com/kaola-fed/foxman/commits?author=fengzilong "Code") [🔌](#plugin-fengzilong "Plugin/utility libraries") [🚇](#infra-fengzilong "Infrastructure (Hosting, Build-Tools, etc)") [📖](https://github.com/kaola-fed/foxman/commits?author=fengzilong "Documentation") [⚠️](https://github.com/kaola-fed/foxman/commits?author=fengzilong "Tests") [🐛](https://github.com/kaola-fed/foxman/issues?q=author%3Afengzilong "Bug reports") | [<img src="https://avatars0.githubusercontent.com/u/4298621?v=3" width="80px;"/><br /><sub>froguardoge</sub>](https://github.com/Froguard)<br />[💻](https://github.com/kaola-fed/foxman/commits?author=Froguard "Code") [🔌](#plugin-Froguard "Plugin/utility libraries") [📖](https://github.com/kaola-fed/foxman/commits?author=Froguard "Documentation") |
| :---: | :---: | :---: |
<!-- ALL-CONTRIBUTORS-LIST:END -->

## 结尾
最后，如果你对 Foxman 的设计有那么点好感，或者是感兴趣，欢迎一起参与到开发当中。

感谢阅读！