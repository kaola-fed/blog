---
title: webpack热更新流程
date: 2018-03-27
---
## What？
webpack热更新，即模块热替换(HMR - Hot Module Replacement)，用于在开发过程中，实时预览修改后的页面，无需重新加载整个页面。其主要通过一下几种方式来加快开发速度：
> 保留在完全重新加载页面时丢失的应用程序状态。  
> 只更新变更内容，以节省宝贵的开发时间。  
> 调整样式更加快速 - 几乎相当于在浏览器调试器中更改样式。  
## Why？
在热更新出现之前我们对于刷新页面一般为强制刷新，或者使用live reload工具，例如浏览器的扩展工具、gulp-livereload、[Live-server](http://tapiov.net/live-server/)等，这些都需要浏览器进行整个页面的刷新，而热更新可以在不刷新页面的前提下进行更新，可以保持当前页面的一些状态和数据。
总的来说，**可以更好的提高开发效率**
## How？
### HMR开启方法
#### webpack-dev-server
* config文件配置dev-server，具体参数见[开发中 Server(DevServer)](https://doc.webpack-china.org/configuration/dev-server/)
* plugins配置NamedModulesPlugin及HotModuleReplacementPlugin。*

NamedModulesPlugin在热加载时直接返回更新文件名，而不是文件的id。
使用NamedModulesPlugin效果：
```javascript
[HMR] Updated modules:
[HMR]  - ./example.js
[HMR]  - ./hmr.js
[HMR] Update applied.
```
不使用NamedModulesPlugin效果：
```javascript
[HMR] Updated modules:
[HMR]  - 39
[HMR]  - 40
[HMR] Update applied.
```
	* HotModuleReplacementPlugin启用 HMR
* 入口文件增加热更新处理。
```javascript
if (module.hot) {
   module.hot.accept('xxx', function() {
     	console.log('Accepting the updated printMe module!');
     	// do something
   })
}
```
#### nodejs API启动devserver
```javascript
const webpackDevServer = require('webpack-dev-server');
const webpack = require('webpack');

const config = require('./webpack.config.js');
const options = {
  contentBase: './dist',
  hot: true,
  host: 'localhost'
};

webpackDevServer.addDevServerEntrypoints(config, options);
const compiler = webpack(config);
const server = new webpackDevServer(compiler, options);

server.listen(5000, 'localhost', () => {
  console.log('dev server listening on port 5000');
});
```
#### webpack-dev-middleware
webpack-dev-middleware是一个容器，它的作用是将webpack处理后的文件传递给server（webpack-dev-middleware 依赖于[memory-fs](https://github.com/webpack/memory-fs)，它将 webpack 原本的 outputFileSystem 替换成了MemoryFileSystem 实例，这样webpack编译的结果是放置在内存中而不是直接生成文件）。webpack-dev-server也是通过webpack-dev-middleware实现，同时，webpack-dev-middleware本身可以作为一个单独的包来使用。
* 在监视模式(watch mode)下如果文件发生改变，middleware 會馬上停止提供bundle 並且延迟请求的回应直至编译完成，如此一來我们就不需要去观察编译是否结束了

使用时，需要两个参数：
* compiler：可以通过 webpack(webpackConfig) 得到
* options：补充 webpack-dev-middleware 需要的特定选项，其中 publicPath 是必须的。

同时，实现热更新必须使用webpack-hot-middleware插件，该插件通过webpack的HMR API，浏览器和服务器之间建立连接并接收更新。它只专注于webpack和浏览器之间的通信机制。
以下为webpack-dev-middleware结合koa的热更新配置示例：
server.js
```javascript
const Koa = require('koa');
const webpack = require('webpack');
const webpackConfig = require('webpack.config.js');
const compiler = webpack(webpackConfig);
// 引入webpack-dev-middleware，
app.use(require('koa-webpack-dev-middleware')(compiler, {
	// 「启动时和每次保存之后，那些显示的 webpack 包(bundle)信息」的消息将被隐藏。错误和警告仍然会显示。
	noInfo: true,
	// publicPath表示对应的处理文件路径
  publicPath: webpackConfig.output.publicPath
}));
app.use(require('koa-webpack-hot-middleware')(compiler));  
app.listen(3000, function () {
  console.log('Example app listening on port 3000!\n');
}); 
```
webpack.config.js
```javascript
{
  entry: [
    // 增加该入口文件，用于处理热更新，其中relaod表示没有找到对应热更新时，是否需要刷新页面
    'webpack-hot-middleware/client?reload=true'
  ],
  plugins: [
    new webpack.NamedModulesPlugin(),
    new webpack.HotModuleReplacementPlugin(),
  ],
}
```

### HMR的工作原理
#### 先简单了解下webpack的工作原理
##### 核心概念
* entry： 入口文件
* module：模块，webpack里一切皆模块，一个模块对应一个文件
* chunk：代码块，对应多个module。
* loader：模块转换器，用于模块内容的转换。
* plugin：插件，在构建流程中监听特定的事件来做一些处理。
##### 流程
[细说 webpack 之流程篇](http://taobaofed.org/blog/2016/09/09/webpack-flow/)这篇文章对于webpack的流程说的比较好，文中的[webpack整体流程图](https://img.alicdn.com/tps/TB1GVGFNXXXXXaTapXXXXXXXXXX-4436-4244.jpg)较为详细的阐述了整个流程。
![webpack整体流程图（引用自七珏）](https://img.alicdn.com/tps/TB1GVGFNXXXXXaTapXXXXXXXXXX-4436-4244.jpg)
* 初始化参数，通过shell脚步和config解析options
* 编译，通过options初始化compiler对象，加载所有配置的插件，执行run()方法开始编译。
* 调用addEntry方法，找到入口文件。
* 编译模块，从入口文件出发，调用所有配置的loader对模块进行处理，同时处理依赖。
* 得到编译结果，包含处理后的最终内容和依赖关系。
* 打包输出，监听seal事件调用各插件对构建后的结果进行封装，根据入口和模块之间的依赖关系，合并拆分组成chunk，每一个chunk对应一个入口文件。（这部分是我们在开发时进行代码优化和功能添加的关键环节）
* 输出，按照output中的配置将文件输出到对应path。
#### 简单工作流
说完webpack的流程，来了解下HMR的工作流程。
开启HMR后，webpack实际上在我们的bundle中加入了一段小型的HMR执行环境，在编译过程中，这个runtime会在我们的页面中运行。
当编译完成时，webpack也不会结束，而是继续监控整个文件是否修改，一旦有修改，就会去编译那些有修改的模块（不会全部重建），然后HMR找到对应修改的模块，尝试在运行状态下进行更新。
更新时，首先检查更新的模块是否能self-accept，即是否支持热替换。如果没有办法确认自己能否直接更新，那么就往上传，通知那些require这个模块的模块进行更新，这样一层层往上，知道有模块可以accept或者寻找结束，表示热更新失败（可刷新整个页面）。
##### 从应用程序的角度
![HMR流程](https://haitao.nos.netease.com/238b6b5c-0fa4-42b4-8258-677833b82239.png)
1. 应用程序代码要求 HMR runtime 检查更新。
2. HMR runtime（异步）下载更新，然后通知应用程序代码。
3. 应用程序代码要求 HMR runtime 应用更新。
4. HMR runtime（异步）应用更新。
##### 从编译器的角度
Webpack通过 Manifest 来解析和加载模块，通过使用manifest 中的数据，runtime 将能够查询模块标识符，检索出背后对应的模块。
那么，对编译器来说，发生HMR时，需要发出更新的请求，以运行之前的版本到新的版本。
生成文件结果：

![生成的文件结果](https://haitao.nos.netease.com/941f5fb6-dc43-4fe9-a593-4ffae83fbdfb.png)

文件hash值为webpack上次编译后生成的hash值，即未发生修改前的值。
这个过程主要两部分完成：
1. 更新后的mainifest，对应json文件
```javascript
{"h":"6bed12d84b2b685b9a2d","c":{"3":true}}
```
 “h“为新的编译hash值（下一次编译生成文件的hash值）
“C”为待更新的chunk目录，有ID和是否更新组成。
2. 一个或多个更新后的chunk片段，对应js文件
```javascript
webpackHotUpdate(3,{

// 文件名称及其对应的修改内容
/***/ "./es6/src/javascript/components/deregulation/appeal/appealList.html":
/***/ (function(module, exports) {

module.exports = "...";

/***/ })

})
```
每个更新chunk包含对应于此chunk的全部更新模块
##### 从模块的角度
> HMR 是可选功能，只会影响包含 HMR 代码的模块。举个例子，通过 style-loader 为 style 样式追加补丁。 为了运行追加补丁，style-loader 实现了 HMR 接口；当它通过 HMR 接收到更新，它会使用新的样式替换旧的样式。  

所以，在每一个模块中，都需要实现当该模块更新后，发生了什么，这个过程也就是无刷新更新替换更新页面的过程。如果一个模块没有hmr处理函数，那么就会冒泡，so，只要整个页面顶端有一个处理函数，那么整个模块也就会被更新。如果一个模块发生更新，整个依赖的模块都会被重新加载。

##### 从HMR Runtime的角度
主要的核心方法为check和apply。
> Check发送http请求来更新manifest，如果请求失败，说明没有可用更新。如果请求成功，待更新的chunk会和当前加载过的chunk进行比较。对每个加载过的chunk，会下载对应的待更新chunk。当所有待更新chunk完成下载，会切换到ready状态。 

> Apply方法将所有被更新模块标记为无效。对于每个无效模块，都需要在模块中有一个更新处理函数，或者在他的父级模块门中有更新处理函数。否则，无效标记冒泡，并也使父级冒泡。每个冒泡直到到达应用程序入口七点，或者到达带有更新处理函数的模块。如果它从入口起点开始冒泡，则此过程失败。  

> 之后，所有的无效模块都被处理和接触加载，然后更新当前hash，并且调用所有accept处理函数，runtime变为闲置状态，一切照常继续。 

示意图：
![](https://haitao.nos.netease.com/03b7a9d3-65bc-4351-8d1b-87768f30c7f3.png) 

### HMR在webpack中的更新流程
从之前的工作原理了解到，实现热更新需要服务端和client的配置，简单查看webpack-dev-server和webpack-hot-middleware的源码，发现两边都有对应的服务端和client的源码。例如webpack-dev-server包含client-src和server.js，webpack-hot-middleware包含client.js和middleware.js。那么他们怎么实现的呢？又具体做了什么呢？
对应源码：

[webpack-dev-middleware](https://github.com/webpack/webpack-dev-middleware)

[webpack-dev-server](https://github.com/webpack/webpack-dev-server)

[webpack-hot-middleware](https://github.com/glenjamin/webpack-hot-middleware)

#### 总的流程图
![总流程图](https://haitao.nos.netease.com/b52b6b00-2ccf-4493-b719-90f658b181f9.png)
1. webpack 监听到文件的修改
2. 根据配置信息，打包编译，且依赖webpack-dev-middleware实现打包结果在内存中
3. webpack-dev-server初始化sockjs，监听”webpack-dev-server”;webpack-hot-middleware初始化eventSource，监听”webpack-hot-middleware”及“done”
4. 监听到修改后，发送消息给客户端
5. 对应的client监听到修改后执行modul.hot.check
6. HotModuleReplacement.runtime执行check事件，请求manifest文件，获取需要更新的模块。
7. 执行module.hot.apply来进行更新

以下对流程进行具体的分析

#### webpack打包初始化
webpack在初次编译打包时，会先根据是否热更新的配置来编译不同的打包结果。别忘了，在配置webpack-dev-server时，我们需要加hot的配置才能使热更新生效；在配置webpack-hot-middleware时，也需要添加client入口文件和热更新插件来使其生效。
##### webpack-dev-server
webpack-dev-server开启热更新的配置是增加--hot的配置，开启后，会自动引入HotModuleReplacementPlugin，并将webpack-dev-server/client文件加入打包入口中。这样就将对应的client代码注入到了最后生成的bundle.js中，并且同时将HMR实现的核心部分（HotModuleReplacement.runtime）也注入到打包后的文件中。

具体源码见webpack-dev-server_lib_util/addDevServerEntrypoints.js中。
```javascript
// 大致代码，加入client，并且根据配置参数决定加入的dev-server内容。
 const domain = createDomain(devServerOptions, app);
    const devClient = [`${require.resolve('../../client/')}?${domain}`];

    if (devServerOptions.hotOnly) { devClient.push('webpack/hot/only-dev-server'); } else if (devServerOptions.hot) { devClient.push('webpack/hot/dev-server'); }

```
##### webpack-hot-middleware
同webpack-dev-server，在手动配置了webpack-hot-middleware/client作为入口文件，以及HotModuleReplacementPlugin后，webpack会将这些热更新需要的代码打包。

#### 监听编译并作出响应
在服务初始化和编译后执行打包文件时，会分别初始化服务端和client建立连接的代码。webpack监听到文件改变，会对文件进行重新编译和打包，然后保存在内存中，等待热更新的调用。webpack-dev-server和webpack-hot-middleware通过监听编译事件，来对修改后的文件及时作出响应。
##### webpack-dev-server
webpack-dev-server在webpack-dev-middleware的基础上，使用websocket（依赖于sockjs实现）进行服务端和浏览器之间的通信。

SockJS是一个浏览器JavaScript库，它提供了一个类似于网络的对象。SockJS提供了一个连贯的、跨浏览器的Javascript API，它在浏览器和web服务器之间创建了一个低延迟、全双工、跨域通信通道。SockJS的一大好处在于提供了浏览器兼容性。优先使用原生WebSocket，如果在不支持websocket的浏览器中，会自动降为轮询的方式。

webpack-dev-server监听webpack编译事件，当编译完成后，通过_sendStatus方法将新的hash值或者对应的错误信息等发送给浏览器。
```javascript
// 监听
const addCompilerHooks = (comp) => {
    comp.hooks.compile.tap('webpack-dev-server', invalidPlugin);
    comp.hooks.invalid.tap('webpack-dev-server', invalidPlugin);
    comp.hooks.done.tap('webpack-dev-server', (stats) => {
      this._sendStats(this.sockets, stats.toJson(clientStats));
      this._stats = stats;
    });
  };
// _sendStats负责发送消息状态给浏览器
// send stats to a socket or multiple sockets
Server.prototype._sendStats = function (sockets, stats, force) {
  if (!force &&
  stats &&
  (!stats.errors || stats.errors.length === 0) &&
  stats.assets &&
  stats.assets.every(asset => !asset.emitted)
  ) { return this.sockWrite(sockets, 'still-ok'); }
  this.sockWrite(sockets, 'hash', stats.hash);
  if (stats.errors.length > 0) { this.sockWrite(sockets, 'errors', stats.errors); } else if (stats.warnings.length > 0) { this.sockWrite(sockets, 'warnings', stats.warnings); } else { this.sockWrite(sockets, 'ok'); }
};
```
webpack-dev-server中的client依赖sockjs-client接受到消息后，更新对应的hash值，并执行reloadApp()进行页面的更新。
```javascript
function reloadApp() {
  if (isUnloading || !hotReload) {
    return;
  }
// 根据hot配置来判断是否需要热更新（依赖hotEmitter执行更新），不需要则刷新页面。
  if (hot) {
    log.info('[WDS] App hot update...');
    // eslint-disable-next-line global-require
    const hotEmitter = require('webpack/hot/emitter');
// 如果配置了模块热更新，就调用 webpack/hot/emitter中初始化的events来 将最新 hash 值发送给 webpack，然后将控制权交给 webpack 客户端代码
    hotEmitter.emit('webpackHotUpdate', currentHash);
    if (typeof self !== 'undefined' && self.window) {
// 如果在浏览器中，则使用window.postMessage API广播事件
      // broadcast update to window
      self.postMessage(`webpackHotUpdate${currentHash}`, '*');
    }
  } else {
// 如果没有配置模块热更新，就直接调用 location.reload 方法刷新页面。
    let rootWindow = self;
    // use parent window for reload (in case we're in an iframe with no valid src)
    const intervalId = self.setInterval(() => {
      if (rootWindow.location.protocol !== 'about:') {
        // reload immediately if protocol is valid
        applyReload(rootWindow, intervalId);
      } else {
        rootWindow = rootWindow.parent;
        if (rootWindow.parent === rootWindow) {
          // if parent equals current window we've reached the root which would continue forever, so trigger a reload anyways
          applyReload(rootWindow, intervalId);
        }
      }
    });
  }

  function applyReload(rootWindow, intervalId) {
    clearInterval(intervalId);
    log.info('[WDS] App updated. Reloading...');
// 刷新页面
    rootWindow.location.reload();
  }
}
```
client接收到发出的webpackHotUpdate 事件，执行module.hot.check来进行更新。（找不到对应更新而回退到浏览器进行更新逻辑也在这一步实现）
```javascript
// 接收webpackHotUpdate事件，执行check()
hotEmitter.on("webpackHotUpdate", function(currentHash) {
  lastHash = currentHash;
  if (!upToDate() && module.hot.status() === "idle") {
    log("info", "[HMR] Checking for updates on the server...");
    check();
  }
});
// check执行module.hot.check()，并检查HMR状态消息，进行冒泡更新，如果冒泡后还找不到需要更新的热更新，则刷新整个页面。
var check = function check() {
  module.hot
    .check(true)
    .then(function(updatedModules) {
      if (!updatedModules) {
        log("warning", "[HMR] Cannot find update. Need to do a full reload!");
        log(
          "warning",
          "[HMR] (Probably because of restarting the webpack-dev-server)"
        );
        window.location.reload();
        return;
      }

      if (!upToDate()) {
        check();
      }

      require("./log-apply-result")(updatedModules, updatedModules);

      if (upToDate()) {
        log("info", "[HMR] App is up to date.");
      }
    })
    .catch(function(err) {
      var status = module.hot.status();
      if (["abort", "fail"].indexOf(status) >= 0) {
        log(
          "warning",
          "[HMR] Cannot apply update. Need to do a full reload!"
        );
        log("warning", "[HMR] " + err.stack || err.message);
        window.location.reload();
      } else {
        log("warning", "[HMR] Update failed: " + err.stack || err.message);
      }
    });
	};
```
#### webpack-hot-middleware
和使用webpack-hot-middleware不同的事，webpack-hot-middleware使用eventsource来实现客户端和webpack之间的通信。

eventsoure，即[使用服务器发送事件 - Server-sent events | MDN](https://developer.mozilla.org/zh-CN/docs/Server-sent_events/Using_server-sent_events)，易如它所说：
> 在Web应用程序中使用服务器发送事件很简单.在服务器端,只需要按照一定的格式返回事件流,在客户端中,只需要为一些事件类型绑定监听函数,和处理其他普通的事件没多大区别.  

在浏览器中通过http连接到服务器，使用evtSource接口监听事件流，在服务器以text/event-stream 格式发送事件流。使用的是HTTP协议，单向通信，只能从服务器发送到浏览器中。

首先，服务端初始化evtSource(middleware.js)
```javascript
// 源码做的事件很简单，初始化事件流后（初始过程中，通过req.socket.setKeepAlive(true);保持长连接），根据webpack的编译状态，根据钩子来针对不同状态对客户端发起不同的消息事件。webpack编译完成（包含首次），给客户端发送built事件消息。然后再发起sync消息来告诉客户端热更新已经准备好了。
function webpackHotMiddleware(compiler, opts) {
  opts = opts || {};
  opts.log = typeof opts.log == 'undefined' ? console.log.bind(console) : opts.log;
  opts.path = opts.path || '/__webpack_hmr';
  opts.heartbeat = opts.heartbeat || 10 * 1000;

  // 初始化eventstream事件流
  var eventStream = createEventStream(opts.heartbeat);
  var latestStats = null;

  // 对webpack重新编译的事件进行监听
  if (compiler.hooks) {
    compiler.hooks.invalid.tap("webpack-hot-middleware", onInvalid);
    compiler.hooks.done.tap("webpack-hot-middleware", onDone);
  } else {
    compiler.plugin("invalid", onInvalid);
    compiler.plugin("done", onDone);
  }
  function onInvalid() {
    latestStats = null;
    if (opts.log) opts.log("webpack building...");
    eventStream.publish({action: "building"});
  }
  function onDone(statsResult) {
    // Keep hold of latest stats so they can be propagated to new clients
    latestStats = statsResult;
    // 给客户端发送built事件
    publishStats("built", latestStats, eventStream, opts.log);
  }
  var middleware = function(req, res, next) {
    if (!pathMatch(req.url, opts.path)) return next();
    eventStream.handler(req, res);
    if (latestStats) {
      // Explicitly not passing in `log` fn as we don't want to log again on
      // the server
      // 给客户发送异步的更新请求
      publishStats("sync", latestStats, eventStream);
    }
  };
  middleware.publish = eventStream.publish;
  return middleware;
}
```

然后，客户端处理evtSource(client.js)
```javascript
// building, built, sysnc分别对应于服务端发送的三种消息事件
// 每次修改文件执行built消息时，会执行built和sync的逻辑，built仅仅是打印了当前时间，而sync才是执行processUpdate的流程，在processUpdate中包含module.hot.apply和module.hot.check的逻辑。
function processMessage(obj) {
  switch(obj.action) {
    case "building":
      if (options.log) {
        console.log(
          "[HMR] bundle " + (obj.name ? "'" + obj.name + "' " : "") +
          "rebuilding"
        );
      }
      break;
    case "built":
      if (options.log) {
        console.log(
          "[HMR] bundle " + (obj.name ? "'" + obj.name + "' " : "") +
          "rebuilt in " + obj.time + "ms"
        );
      }
      // fall through
    case "sync":
      if (obj.name && options.name && obj.name !== options.name) {
        return;
      }
      if (obj.errors.length > 0) {
        if (reporter) reporter.problems('errors', obj);
      } else if (obj.warnings.length > 0) {
        if (reporter) reporter.problems('warnings', obj);
      } else {
        if (reporter) {
          reporter.cleanProblemsCache();
          reporter.success();
        }
        processUpdate(obj.hash, obj.modules, options);
      }
      break;
    default:
      if (customHandler) {
        customHandler(obj);
      }
  }

  if (subscribeAllHandler) {
    subscribeAllHandler(obj);
  }
}
```
processUpdate内执行module.hot.check
#### 模块热更新处理
查看每个模块打包后的代码，我们发现每个模块都初始化了module hot的逻辑。
![打包后包含module hot的逻辑](https://haitao.nos.netease.com/7e53c2a7-01f9-4982-b598-1a0d0658734b.png)
hotCreateModule主要是通过ajax获取热更新文件的内容，内部包含hotChekc和hotApply两个接口。这两部分为热更新实现的核心。

hotCheck
```javascript
function hotCheck(apply) {
	// 状态判断，保证编译成功
	if(hotStatus !== "idle") throw new Error("check() is only allowed in idle status");
	hotApplyOnUpdate = apply;
	hotSetStatus("check");
// hotDownloadManifest建立ajax请求，获取manifest json文件
	return hotDownloadManifest(hotRequestTimeout).then(function(update) {
		...
		// 从manifest文件中拿到chunkId后，查找chunkID是否存在且需要更新，然后调用hotDownloadUpdateChunk下载更新的js文件，往页面中增加该js文件执行更新
		for(var chunkId in installedChunks)
		{ // eslint-disable-line no-lone-blocks
			/*globals chunkId */
			hotEnsureUpdateChunk(chunkId);
		}
		// 
		if(hotStatus === "prepare" && hotChunksLoading === 0 && hotWaitingFiles === 0) {
			hotUpdateDownloaded();
		}
		...
	});
}
```
其中，hotDownloadManifest请求xxx.hot-updata.json文件
![](https://haitao.nos.netease.com/9fa7cd86-2a1f-44ec-b148-bf3690f2929e.png)
hotDownloadUpdateChunk请求xxx.hot-update.js文件，然后将文件作为script插入页面head头中。
![](https://haitao.nos.netease.com/d2eb58aa-60a8-441d-b024-2a692ed6b47f.png)
下载下来的文件内容大概如下：
![](https://haitao.nos.netease.com/ba9d25eb-12ca-4fed-a995-82df42596569.png)

执行函数webpackHotUpadate在打包过程中已经注入。
![](https://haitao.nos.netease.com/66e28e2f-012a-4f6d-bf56-8913b7ea68be.png)

webpackHotUpadate回去调用hotApply逻辑来执行更新
hotApply的代码较长，主要的过程主要是一次次冒泡，找到和当前更新模块有依赖的所有模块，查看子模块和父模块是否接收更新，如果接受，则标记为过期模块，不接受，则一直向上冒泡，直到顶部入口点。然后针对标记的模块进行accept更新处理，并删除原有依赖，建立新的依赖。

### 业务代码要做的事情
注入module.hot.accept，即可接收热更新。实现无需刷新页面而更新的逻辑都在accept内部实现。
#### css
对CSS来说，style-loader已经集成了热更新逻辑，本质上是把更新后的样式放在<style></style>标签内加载
![](https://haitao.nos.netease.com/ef27945e-e988-4cba-9d46-ead511688f5c.png)

#### vue
开启热更新后，vue-loader会在每一个vue组件构建的代码都会增加一段hotAPI，本质是运用组件的render方法，重新render组件，实现无刷新更新。
具体可见[实现API](https://github.com/vuejs/vue-hot-reload-api)
![](https://haitao.nos.netease.com/cab8ff9f-28f6-4445-b5ef-38a213a6da85.png)

#### react
[react-hot-loader](https://github.com/gaearon/react-hot-loader)，通过运用react的render重新渲染每一个组件
![](https://haitao.nos.netease.com/970cafd9-4443-47c2-a075-6d3347394284.png)


## END
最后，完整的细节流程图如下
![](https://haitao.nos.netease.com/269bf471-5428-45b2-bc10-015984651e8a.png)

### 参考
[细说 webpack 之流程篇 | Taobao FED | 淘宝前端团队](http://taobaofed.org/blog/2016/09/09/webpack-flow/)

[webpack-dev-server使用方法，看完还不会的来找我~ - JSer - SegmentFault 思否](https://segmentfault.com/a/1190000006670084)

[手把手深入理解 webpack dev middleware 原理與相關 plugins](https://andyyou.github.io/2016/05/30/webpack-dev-middleware-in-express/)

[模块热替换(Hot Module Replacement)](https://doc.webpack-china.org/concepts/hot-module-replacement/)

[当年校招时，我就死在这个问题上… - CSDN博客](http://blog.csdn.net/gitchat/article/details/78341649)

[GitHub - liangklfangl/webpack-hmr: 这篇文章来自于我的github文章全集,欢迎star https://github.com/liangklfangl/react-article-bucket](https://github.com/liangklfangl/webpack-hmr)

[Webpack HMR 原理解析](https://zhuanlan.zhihu.com/p/30669007)

[Webpack 热更新实现原理分析](https://zhuanlan.zhihu.com/p/30623057)

by [dj](www.kaola.com)
