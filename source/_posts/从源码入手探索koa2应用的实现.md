---
title: 从源码入手探索koa2应用的实现
date: 2017-12-29
---

## koa2特性
> A Koa application is an object containing an array of middleware functions which are composed and executed in a stack-like manner upon request.

- 只提供封装好http上下文、请求、响应，以及基于async/await的中间件容器
- 基于koa的app是由一系列中间件组成，原来是generator中间件，现在被async/await代替（generator中间件，需要通过中间件koa-convert封装一下才能使用）
- 按照app.use(middleware)顺序依次执行中间件数组中的方法

<!-- more -->

> 1.0 版本是通过组合不同的 generator，可以免除重复繁琐的回调函数嵌套，并极大地提升错误处理的效率。
>
> 2.0版本Koa放弃了generator，采用Async 函数实现组件数组瀑布流式（Cascading）的开发模式。

## 源码文件

```
├── lib
│   ├── application.js
│   ├── context.js
│   ├── request.js
│   └── response.js
└── package.json
```
核心代码就是lib目录下的四个文件

- application.js 是整个koa2 的入口文件，封装了context，request，response，以及最核心的中间件处理流程。
- context.js 处理应用上下文，里面直接封装部分request.js和response.js的方法
- request.js 处理http请求
- response.js 处理http响应

## koa流程

![koa总体流程图](https://camo.githubusercontent.com/707ab384df1cffd60ba1c19f188fd6edbce34d3d/687474703a2f2f62657277696e2e6769746875622e696f2f707074732f6b6f612f696d672f6b6f612d666c6f772e6a7067)

koa的流程分为三个部分：**初始化 -> 启动Server -> 请求响应**

- 初始化

    - 初始化koa对象之前我们称为初始化

- 启动server

    - 初始化中间件(中间件建立联系)
    - 启动服务,监听特定端口,并生成一个新的上下文对象

- 请求响应

    - 接受请求,初始化上下文对象
    - 执行中间件
    - 将body返回给客户端

> ### 初始化

定义了三个对象，`context`, `response`, `request`

- `request` 定义了一些set/get访问器，用于设置和获取请求报文和url信息，例如获取query数据，获取请求的url（详细API参见[Koa-request文档](http://koajs.com/#request)）
- `response` 定义了一些set/get操作和获取响应报文的方法（详细API参见[Koa-response 文档](http://koajs.com/#response)）
- `context` 通过第三方模块 delegate 将 koa 在 Response 模块和 Request 模块中定义的方法委托到了 context 对象上，所以以下的一些写法是等价的：

    ```js
    //在每次请求中，this 用于指代此次请求创建的上下文 context(ctx)
    this.body ==> this.response.body
    this.status ==> this.response.status
    this.href ==> this.request.href
    this.host ==> this.request.host
    ......
    ```
    为了方便使用，许多上下文属性和方法都被委托代理到他们的 `ctx.request` 或 `ctx.response`，比如访问 `ctx.type` 和 `ctx.length` 将被代理到 `response` 对象，`ctx.path` 和 `ctx.method` 将被代理到 `request` 对象。

    每一个请求都会创建一段上下文，在控制业务逻辑的中间件中，`ctx`被寄存在`this`中（详细API参见 [Koa-context 文档](http://koajs.com/#context)）

> ### 启动Server

> 1. 初始化一个koa对象实例
> 2. 监听端口

```js
var koa = require('koa');
var app = koa()

app.listen(9000)
```

> #### 解析启动流程，分析源码

#### `application.js`是koa的入口文件

```js
// 暴露出来class，`class Application extends Emitter`，用new新建一个koa应用。
module.exports = class Application extends Emitter {

    constructor() {
        super();

        this.proxy = false; // 是否信任proxy header，默认false // TODO
        this.middleware = [];   // 保存通过app.use(middleware)注册的中间件
        this.subdomainOffset = 2;
        this.env = process.env.NODE_ENV || 'development';   // 环境参数，默认为 NODE_ENV 或 ‘development’
        this.context = Object.create(context);  // context模块，通过context.js创建
        this.request = Object.create(request);  // request模块，通过request.js创建
        this.response = Object.create(response);    // response模块，通过response.js创建
    }
    ...
```

`Application.js` 除了上面的的构造函数外，还暴露了一些公用的api，比如常用的 `listen`和`use`（use放在后面讲）。

#### listen
> 作用： **启动koa server**

> 语法糖

```js
// 用koa启动server
const Koa = require('koa');
const app = new Koa();
app.listen(3000);

// 等价于

// node原生启动server
const http = require('http');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
https.createServer(app.callback()).listen(3001); // on mutilple address
```

```js
// listen
listen(...args) {
    const server = http.createServer(this.callback());
    return server.listen(...args);
}
```
封装了nodejs的创建http server，在监听端口之前会先执行`this.callback()`

```js
// callback

callback() {
    // 使用koa-compose(后面会讲) 串联中间件堆栈中的middleware，返回一个函数
    // fn接受两个参数 (context, next)
    const fn = compose(this.middleware);

    if (!this.listeners('error').length) this.on('error', this.onerror);

    // this.callback()返回一个函数handleReqwuest，请求过来的时候，回调这个函数
    // handleReqwuest接受参数 (req, res)
    const handleRequest = (req, res) => {
        // 为每一个请求创建ctx，挂载请求相关信息
        const ctx = this.createContext(req, res);
        // handleRequest的解析在【请求响应】部分
        return this.handleRequest(ctx, fn);
    };

    return handleRequest;
}

```

`const ctx = this.createContext(req, res);`创建一个最终可用版的`context`

![](https://camo.githubusercontent.com/270554efef8e3112443c22e695b401ba7e8847bd/687474703a2f2f62657277696e2e6769746875622e696f2f707074732f6b6f612f696d672f636f6e746578742e706e67)

ctx上包含5个属性，分别是request，response，req，res，app

request和response也分别有5个箭头指向它们，所以也是同样的逻辑

> 补充了解 各对象之间的关系

![](https://sfault-image.b0.upaiyun.com/135/226/1352261008-594a32c87b693_articlex)
> 最左边一列表示每个文件的导出对象
>
> 中间一列表示每个Koa应用及其维护的属性
>
> 右边两列表示对应每个请求所维护的一些列对象
>
> 黑色的线表示实例化
>
> 红色的线表示原型链
>
> 蓝色的线表示属性

> ### 请求响应

回顾一下，koa启动server的代码

```js
app.listen = function() {
    var server = http.createServer(this.callback());
    return server.listen.apply(server, arguments);
};
```

```js
// callback
callback() {
    const fn = compose(this.middleware);
    ...
    const handleRequest = (req, res) => {
        const ctx = this.createContext(req, res);
        return this.handleRequest(ctx, fn);
    };
    return handleRequest;
}
```
`callback()`返回了一个请求处理函数`this.handleRequest(ctx, fn)`

```js
// handleRequest

handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;

    // 请求走到这里标明成功了，http respond code设为默认的404 TODO 为什么？
    res.statusCode = 404;

    // koa默认的错误处理函数，它处理的是错误导致的异常结束
    const onerror = err => ctx.onerror(err);

    // respond函数里面主要是一些收尾工作，例如判断http code为空如何输出，http method是head如何输出，body返回是流或json时如何输出
    const handleResponse = () => respond(ctx);

    // 第三方函数，用于监听 http response 的结束事件，执行回调
    // 如果response有错误，会执行ctx.onerror中的逻辑，设置response类型，状态码和错误信息等
    onFinished(res, onerror);

    // 执行中间件，监听中间件执行结果
    // 成功：执行response
    // 失败，捕捉错误信息，执行对应处理
    // 返回Promise对象
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
}
```
##### Koa处理请求的过程：当请求到来的时候，会通过 req 和 res 来创建一个 context (ctx) ，然后执行中间件


#### koa中另一个常用API - use

> 作用：  **将函数推入middleware数组**

```js
use(fn) {
    // 首先判断传进来的参数，传进来的不是一个函数，报错
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    // 判断这个函数是不是 generator
    // koa 后续的版本推荐使用 await/async 的方式处理异步
    // 所以会慢慢不支持 koa1 中的 generator，不再推荐大家使用 generator
    if (isGeneratorFunction(fn)) {
        deprecate('Support for generators will be removed in v3. ' +
        'See the documentation for examples of how to convert old middleware ' +
        'https://github.com/koajs/koa/blob/master/docs/migration.md');
        // 如果是 generator，控制台警告，然后将函数进行包装
        fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    // 将函数推入 middleware 这个数组，后面要依次调用里面的每一个中间件
    this.middleware.push(fn);
    // 保证链式调用
    return this;
}
```

#### koa-compose

`const fn = compose(this.middleware)`

app.use([MW])仅仅是将函数推入middleware数组，真正让这一系列函数组合成为中间件的，是koa-compose，koa-compose是Koa框架中间件执行的发动机

```js
'use strict'

module.exports = compose

function compose (middleware) {
    // 传入的 middleware 必须是一个数组, 否则报错
    if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
    // 循环遍历传入的 middleware， 每一个元素都必须是函数，否则报错
    for (const fn of middleware) {
        if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
    }

    return function (context, next) {
        // last called middleware #
        let index = -1
        return dispatch(0)
        function dispatch (i) {
            if (i <= index) return Promise.reject(new Error('next() called multiple times'))
            index = i
            let fn = middleware[i]
            if (i === middleware.length) fn = next
            // 如果中间件中没有 await next ，那么函数直接就退出了，不会继续递归调用
            if (!fn) return Promise.resolve()
            try {
                return Promise.resolve(fn(context, function next () {
                    return dispatch(i + 1)
                }))
            } catch (err) {
                return Promise.reject(err)
            }
        }
    }
}
```

Koa2.x的compose方法虽然从纯generator函数执行修改成了基于Promise.all，但是中间件加载的中心思想没有发生改变，依旧是从第一个中间件开始，遇到await/yield next，就中断本中间件的代码执行，跳转到对应的下一个中间件执行期内的代码…一直到最后一个中间件，然后逆序回退到倒数第二个中间件await/yield next下部分的代码执行，完成后继续会退…一直会退到第一个中间件await/yield next下部分的代码执行完成，中间件全部执行结束

> 级联的流程，V型加载机制

![洋葱结构](https://camo.githubusercontent.com/d80cf3b511ef4898bcde9a464de491fa15a50d06/68747470733a2f2f7261772e6769746875622e636f6d2f66656e676d6b322f6b6f612d67756964652f6d61737465722f6f6e696f6e2e706e67)




## koa2常用中间件

> ### koa-router 路由

对其实现机制有兴趣的可以戳看看 -> [Koa-router路由中间件API详解](http://www.nodepeixun.com/a/nodekuangjia/20170114/126.html)

```
const Koa = require('koa')
const fs = require('fs')
const app = new Koa()

const Router = require('koa-router')

// 子路由1
let home = new Router()
home.get('/', async ( ctx )=>{
  let html = `
    <ul>
      <li><a href="/page/helloworld">/page/helloworld</a></li>
      <li><a href="/page/404">/page/404</a></li>
    </ul>
  `
  ctx.body = html
})

// 子路由2
let page = new Router()
page.get('hello', async (ctx) => {
    ctx.body = 'Hello World Page!'
})

// 装载所有子路由的中间件router
let router = new Router()
router.use('/', home.routes(), home.allowedMethods())
router.use('/page', page.routes(), page.allowedMethods())

// 加载router
app.use(router.routes()).use(router.allowedMethods())

app.listen(3000, () => {
  console.log('[demo] route-use-middleware is starting at port 3000')
})
```

> ### koa-bodyparser 请求数据获取

#### GET请求数据获取
获取GET请求数据有两个途径

1. 是从上下文中直接获取

    - 请求对象ctx.query，返回如 { a:1, b:2 }
    - 请求字符串 ctx.querystring，返回如 a=1&b=2

2. 是从上下文的request对象中获取

    - 请求对象ctx.request.query，返回如 { a:1, b:2 }
    - 请求字符串 ctx.request.querystring，返回如 a=1&b=2

#### POST请求数据获取
> 对于POST请求的处理，koa2没有封装获取参数的方法需要通过解析上下文context中的原生node.js请求对象req，将POST表单数据解析成query string（例如：a=1&b=2&c=3），再将query string 解析成JSON格式（例如：{"a":"1", "b":"2", "c":"3"}）

对于POST请求的处理，koa-bodyparser中间件可以把koa2上下文的formData数据解析到ctx.request.body中

```
...
const bodyParser = require('koa-bodyparser')

app.use(bodyParser())

app.use( async ( ctx ) => {

  if ( ctx.url === '/' && ctx.method === 'POST' ) {
    // 当POST请求的时候，中间件koa-bodyparser解析POST表单里的数据，并显示出来
    let postData = ctx.request.body
    ctx.body = postData
  } else {
    ...
  }
})

app.listen(3000, () => {
  console.log('[demo] request post is starting at port 3000')
})

```
> ### koa-static 静态资源加载

为静态资源访问创建一个服务器，根据url访问对应的文件夹、文件

```
...
const static = require('koa-static')
const app = new Koa()

// 静态资源目录对于相对入口文件index.js的路径
const staticPath = './static'

app.use(static(
  path.join( __dirname,  staticPath)
))


app.use( async ( ctx ) => {
  ctx.body = 'hello world'
})

app.listen(3000, () => {
  console.log('[demo] static-use-middleware is starting at port 3000')
})
```


#### 参考
- [koa文档](http://koajs.com/#)
- [深入浅出koa #2](https://github.com/snailTJ/Blog/issues/2)
- [深入浅出koa2 #11](https://github.com/snailTJ/Blog/issues/11)
- [Node.js Koa 之Async中间件](https://www.jianshu.com/p/7c795c28e8ec)
- [koa中文文档](https://github.com/guo-yu/koa-guide)
- [koa2 源码分析 (一)](https://dongliang1993.github.io/2017/12/06/koa2-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E4%B8%80/)
- [Koa2源码阅读笔记](https://mrsunny123.github.io/2017/06/21/Koa2-Code/)
- [Koa2进阶学习笔记](https://chenshenhai.github.io/koa2-note/)
- [Koa-router路由中间件API详解](http://www.nodepeixun.com/a/nodekuangjia/20170114/126.html)
- [跨入Koa2.0，从Compose开始](https://cnodejs.org/topic/5780e12e69d72f545483ca69)

