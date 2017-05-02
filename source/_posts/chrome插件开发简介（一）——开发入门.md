---
title: chrome插件开发简介（一）——开发入门 
date: 2017-03-31
---

# chrome插件开发简介（一）——开发入门

> chrome作为目前最流行的浏览器，备受前端推崇，原因除了其对于前端标准的支持这一大核心原因之外，还有就是其强大的扩展性，
基于其开发规范实现的插件如今已经非常庞大，在国内也是欣欣向荣，
如天猫开发了大量的扩展，用于检测页面质量以及页面性能，淘宝开发了许多的扩展以供运营工具的优化等等。其强大性不言而喻。

这里我们来讲一下其插件开发

<!-- more -->

## 基础概念

1. 与chrome应用类似，chrome扩展主要是用于扩充chrome浏览器的功能。他体现为一些文件的集合，包括前端文件（html/css/js）,配置文件manifest.json。
主要采用JavaScript语言进行编写。

> 个别扩展可能会用到DLL和so动态库，不过出于安全以及职责分离的考虑，在后续的标准中，将会被舍弃，这里不再赘述

2. chrome扩展能做的事情：

- 基于浏览器本身窗口的交互界面
- 操作用户页面：操作用户页面里的dom
- 管理浏览器：书签，cookie，历史，扩展和应用本身的管理，其中甚至修改chrome的一些默认页面，地址栏关键字等，包括浏览器外观主题都是可以更改的
> 这里需要着重提一下，前端开发者喜欢的DevTools工具，在这里也能进行自定义
- 网络通信：http，socket，UDP/TCP等
- 跨域请求不受限制：
- 常驻后台运行
- 数据存储：采用3种方式（localStorage，Web SQL DB，chrome提供的存储API（文件系统））
- 扩展页面间可进行通信：提供runtime相关接口
- 其他功能：下载，代理，系统信息，媒体库，硬件相关（如usb设备操作，串口通信等），国际化


3. chrome扩展的限制：

- 环境限制：基本上功能性操作都需要通过chrome提供的API来完成，这跟实际的页面js又有一些差异，看上去并没有那么的完全自由。
如chrome扩展的页面脚本，可以获取并操作页面dom，但是，出于安全性的考虑，页面脚本的域是独立区分开来的，即js成员变量不共享。
即：插入到用户页面的js脚本可执行，但是与原始脚本不同执行域，互相不会影响。
content_script不能使用除了chrome.extension之外的chrome.* 的接口,不能访问它所在扩展中定义的函数和变量,不能访问web页面或其它content script中定义的函数和变量,不能做cross-site XMLHttpRequests

- chrome扩展，其功能受限于chrome的API，比如说文件系统，必须通过chrome的fileSystem接口，而这些接口仅仅只是对html5已有的文件系统接口的扩展，
它允许Chrome应用读写硬盘中用户选择的任意位置，而HTML5本身提供的文件系统接口则只能在沙箱中读写文件，并不能获取用户磁盘中真正的目录。

## 举个栗子:apple:

> 下面以一个简单的例子来简述一下我们开发扩展过程中会遇到哪些问题

> 这个栗子主要用于去自定义用户页面的样式（比如此处为改滚动条样式），插入一个自定义的脚本到用户页面中执行（此处为输出一些简单信息）

### 1.编辑扩展程序所需要的主要文件夹

#### 文件结构说明

```
 ./
 ├─ manifest.json //扩展的配置项
 ├─ Custom.js     //自定义js脚本
 ├─ Custom.css    //自定义css样式
 ├─ icon.png      //扩展程序的icon
 └─ popup.html    //扩展的展示弹窗
```

- 自定义js脚本：浏览器中执行，但并非真正意义上的“插入”，与原来页面的js域隔离开
> 例如，custom.js中的window跟页面js中的window不是同一个对象，变量也无法共享。

```js
console.log('执行init');
console.log(document.title);
console.log(document.getElementById("abc"));
// 实例函数，可以供popup.html中调用
function helloWorld(name){
    console.log(`${name} say 'hello world!'`);
    alert(`${name} say 'hello world!'`);
}
//...
```
> 这样的脚本路径将以 chrome://extensions/扩展的id串码/path/to/you/js/xxx.js出现在控制台调试时候。


![image](https://github.com/Froguard/crxs/raw/master/doc/res/injectJsPath.jpg)

- 自定义样式：同样，css也不是真正的插入到页面的document中，而是浏览器的样式生效策略中会加入这些样式规则

![image](https://github.com/Froguard/crxs/raw/master/doc/res/injectCssPath.jpg)

```css
/* 重置滚动条 */
::-webkit-scrollbar {width:4px!important;height:7px;}
::-webkit-scrollbar-button {display:none;}
::-webkit-scrollbar-thumb {border-radius:3px;background: #45A5DB;}
::-webkit-scrollbar-track {width:4px;height:7px;background:transparent;}
::-webkit-scrollbar-track-piece {background:transparent;}

/* ... */

```

- 小窗口popup.html：这里面是一个按钮，点击后去调用custom.js里面的helloWorld函数

```html
<!doctype html>
<html>
  <head>
    <title>hello world</title>
  </head>
  <body>
    <h2>hello world</h2>
	<p><button onclick="dealClick('王小明')">按钮</button></p>
    <script>
        function dealClick(name){
            //这里值得注意，大部分功能性操作，比如执行脚本，执行函数，都是不可以直接执行，而需要通过chrome.*这样方式进行
            chrome.tabs.executeScript(
                null,
                {
                    code: `helloWorld('${name}')` // 这里调用的是上面的custom.js重定义好的function
                }
            );
        }
    </script>
  </body>
</html>
```

- **核心配置文件manifest.json**：

```js
{
  "name": "扩展名称",
  "version": "1.0.0",
  "manifest_version": 2,
  "description": "扩展描述",
  "icons" : {           // 扩展的icon
    "16" : "icon.png",
    "48" : "icon.png",
    "128" : "icon.png"
  },
  "browser_action": {   // browser_action表示程序图标会出现在地址栏右侧，若要出现在地址栏，则写成page_action
    "default_title": "日报工具",
    "default_icon": "icon.png",
    "default_popup": "popup.html"
  },
  "content_scripts": [  //content_scripts是在Web页面内运行的javascript脚本。
                        //通过使用标准的DOM，它们可以获取浏览器所访问页面的详细信息，并可以修改这些信息。
    {                   //这里的值是数组，可以针对多个站点进行不同的操作配置
      "matches": [
        "http://www.google.com/*"
      ],
      "css": [
        "custom.css"
      ],
      "js": [
        "custom.js"
      ],
      "all_frames": true,
      "run_at": "document_idle"
    }
  ],
  "permissions": [   //一些权限的配置，
    "cookies",       //比如cookie权限，比如系统通知权限，类似于notify这样的东西，在window系统上未右下角的小气泡
    "notifications"
  ]
}
```

page_action，browser_action类型的扩展对应位置

![image](https://github.com/Froguard/crxs/raw/master/doc/res/action_pos.jpg)


### 2.打包生成插件包

浏览器打开 [chrome://extensions/](chrome://extensions/)（或者‘更过工具->扩展程序’）,左上角有一个 ```打包扩展程序``` 按钮

![image](https://github.com/Froguard/crxs/raw/master/doc/res/pack.jpg)

然后，将生成的*.crx文件拖到浏览器即可完成安装。


> 到这步插件其实已经差不多了，当然，这里的功能都比较简单，你可以自己尝试一些更高级的功能

> 由于自己DIY的扩展是没有发布到web app store的，所以下次打开浏览器时会被禁掉，解决方法见第二篇文章《chrome插件开发简介（二）——如何添“加浏览器扩展白名单”》


