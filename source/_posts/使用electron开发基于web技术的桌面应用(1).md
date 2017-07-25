---
title: 使用electron开发基于web技术的桌面应用
date: 2017-07-03
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

随着互联网的发展, 越来越多C/S模式的应用正逐渐变成B/S模式, 有些没有S端的应用, 也从桌面转移到web, 这得益于web的轻量, 开发迅速, 不用装卸等优点.

但是, web应用有一个局限--无法对文件系统执行操作.

当我们既想要web开发的快捷, 又想要对操作系统的api调用时, 基于浏览器内核的桌面开发技术应运而生.
没记错的话, 有道云笔记客户端就是基于这一技术开发的, 具体解决方案叫什么, 我忘了, 总之今天我们不介绍这个.

<!-- more -->

今天要介绍的, 是[electron](https://electron.atom.io/).

electron是基于nodejs, chromium内核, V8引擎的一套web应用执行环境.

基于electron开发出来的产品, 笔者用过 文本编辑器Atom, 版本管理工具GitKraken, VSC, github desktop等. 总体来说, 除了性能偶尔比较差之外, 其他体验都是比较不错的.

闲言少叙, 下面正式开始介绍使用姿势:

1. 安装
找一个你喜欢的文件夹, 打开terminal, 执行 `npm init`, 一路填下来, 就生成了一个基本的npm 项目结构, 然后执行`npm i electron`

或直接新建一个`package.json`文件, 内容如下:
```
{
  "name": "electron-demo",
  "version": "1.0.0",
  "main": "main.js",
  "scripts": {
    "start": "./node_modules/.bin/electron ."
  },
  "author": "your name",
  "license": "ISC",
  "devDependencies": {
    "electron": "~1.6.2"
  }
}
```
然后执行`npm i`

总之, 一个最简单的electron项目, 项目依赖方面, 只需要安装一个electron包就可以了

ok, electron的开发环境准备完毕, 然后可以开始coding了.

下面讲一下electron的文件组成. 一个最简单的electron项目, 需要有一个html文件作为视图, 一个后台js文件用于启动工程. 当然, 这样仅仅能渲染出一个静态的页面. 所以如果想要页面中有交互逻辑, 前台js是必不可少的.

所谓前台js, 是指html文件引用到的js. 而后台js, 则是运行主进程的js. 既然提到了主进程, 就介绍下electron的进程结构.

electron有两种进程: 主进程(Main Process)和渲染进程(Render Process). 主进程用来将生成`BrowserWindow `实例, 也就是网页执行的环境, 每一个BrowserWindow对象, 对应有一个渲染进程. 一个BrowserWindow被销毁, 它的渲染进程也被销毁. 主进程管理所有的网页对象和它们对应的渲染进程. 渲染进程之间是隔离的, 渲染进程只关心自己所运行的网页对象.

简而言之, 主进程和渲染进程是一对多的关系.package.json中的main字段, 指向的就是主进程的逻辑入口


下面来新建一个index.html文件, 内容如下
```
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Test electron</title>
  </head>
  <body>
    <div id="page">hello world</div>
  </body>
</html>

```

文件内容十分简单, 用浏览器打开看看, 页面里一行文字: hello world

当然, 用浏览器打开, 就不是electron应用了. 我们要用electron把这个页面渲染出来.
前面说过, 渲染一个页面, 需要有主进程来生成BrowserWindow实例.
那么接下来写主进程 main.js, 代码如下:
```
const { app, BrowserWindow } = require('electron');
const path = require('path');
const url = require('url');

// Keep a global reference of the window object, if you don't, the window will
// be closed automatically when the JavaScript object is garbage collected.
let win;
function loadFile(src) {
    win.loadURL(url.format({
        pathname: path.join(__dirname, src),
        protocol: 'file:',
        slashes: true
    }));
}

function createWindow() {
    // Create the browser window.
    win = new BrowserWindow({
        width: 1000,
        height: 800
    });

    loadFile('index.html');

    // 打开你喜欢的 DevTools, 看看你写的bug
    win.webContents.openDevTools();

    // Emitted when the window is closed.
    win.on('closed', () => {
        win = null;
    });
}

app.on('ready', createWindow);

app.on('activate', () => {
    // On macOS it's common to re-create a window in the app when the
    // dock icon is clicked and there are no other windows open.
    if (win === null) {
        createWindow();
    }
});

```

代码很简单, 定义了一个`createWindow`方法, 当app收到`ready`信号或`activate`信号时, 执行它.
从第一行的require可以看出, app是electron对外提供的一个模块, 指我们构建的这个应用.
createWindow方法也很简单, new了一个BrowserWindow对象, 定了个长宽, 然后调用`loadFile`方法, 加载刚才我们写好的index.html.

两个文件写好了, 怎么执行呢?

有两种方式.
1. 如果你全局安装了electron, 直接在当前代码文件路径下, 执行`electron .`
2. 否则, 运行package.json里定义的命令 `npm run start`

此时, 一个浏览器窗口样的东西展现在屏幕上, 还开了devTool.

到此, 一个静态的electron应用就开发好了, 它的功能就是现实一句话.

下一期, 我们将讲解如何添加交互逻辑, 以及主进程和渲染进程的通信等细节.

同学们, 再见 

P.S. 全部实验代码可以到[这里](https://github.com/yubaoquan/electron_study)下载
