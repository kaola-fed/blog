---
title: 前端工程-从原理到轮子之JS模块化
date: 2017-03-29
---

> 目前，一个典型的前端项目技术框架的选型主要包括以下三个方面：
> 1. JS模块化框架。(Require/Sea/ES6 Module/NEJ)
> 2. 前端模板框架。(React/Vue/Regular)
> 3. 状态管理框架。(Flux/Redux)
> 系列文章将从上面三个方面来介绍相关原理，并且尝试自己造一个简单的轮子。

<!-- more -->

本篇介绍的是**JS模块化**。
JS模块化是随着前端技术的发展，前端代码爆炸式增长后，工程化所采取的必然措施。目前模块化的思想分为CommonJS、AMD和CMD。有关三者的区别，大家基本都多少有所了解，而且资料很多，这里就不再赘述。

模块化的核心思想：
1. **拆分**。将js代码按功能逻辑拆分成多个可复用的js代码文件（模块）。
2. **加载**。如何将模块进行加载执行和输出。
2. **注入**。能够将一个js模块的输出注入到另一个js模块中。
3. **依赖管理**。前端工程模块数量众多，需要来管理模块之间的依赖关系。

根据上面的核心思想，可以看出要设计一个模块化工具框架的关键问题有两个：一个是如何将一个模块执行并可以将结果输出注入到另一个模块中；另一个是，在大型项目中模块之间的依赖关系很复杂，如何使模块按正确的依赖顺序进行注入，这就是依赖管理。

下面以具体的例子来实现一个简单的**基于浏览器端**的**AMD**模块化框架（类似NEJ）,对外暴露一个define函数，在回调函数中注入依赖，并返回模块输出。要实现的如下面代码所示。
```javascript
define([
    '/lib/util.js', //绝对路径
    './modal/modal.js', //相对路径
    './modal/modal.html',//文本文件
], function(Util, Modal, tpl) {
	/*
	* 模块逻辑
	*/
	return Module;
})
```

#### 1.  模块如何加载和执行
先不考虑一个模块的依赖如何处理。假设一个模块的依赖已经注入，那么如何加载和执行该模块，并输出呢？
在浏览器端，我们可以借助浏览器的**script**标签来实现**JS模块文件**的引入和执行，对于**文本模块文件**则可以直接利用**ajax**请求实现。
具体步骤如下：
- 第一步，获取**模块文件的绝对路径**。
要在浏览器内加载文件，首先要获得对应模块文件的完整网络绝对地址。由于**a标签**的href属性总是会返回绝对路径，也就是说它具有把相对路径转成绝对路径的能力，所以这里可以利用该特性来获取模块的绝对网络路径。需要指出的是，对于使用相对路径的依赖模块文件，还需要递归先获取当前模块的网络绝对地址，然后和相对路径拼接成完整的绝对地址。代码如下：

```javascript
var a = document.createElement('a');
a.id = '_defineAbsoluteUrl_';
a.style.display = 'none';
document.body.appendChild(a);

function getModuleAbsoluteUrl(path) {
    a.href = path;
    return a.href;
}

function parseAbsoluteUrl(url, parentDir) {
    var relativePrefix = '.',
        parentPrefix = '..',
        result;
    if (parentDir && url.indexOf(relativePrefix) === 0) {
        // 以'./'开头的相对路径
        return getModuleAbsoluteUrl(parentDir.replace(/[^\/]*$/, '') + url);
    }
    if (parentDir && url.indexOf(parentPrefix) === 0) {
        // 以'../'开头的相对路径
        return getModuleAbsoluteUrl(parentDir.replace(/[\/]*$/, '').replace(/[\/]$/, '').replace(/[^\/]*$/, '') + url);
    }
    return getModuleAbsoluteUrl(url);
}
```
- 第二步，**加载和执行模块文件**。

对于JS文件，利用**script**标签实现。代码如下：
	
```javascript
var head = document.getElementsByTagName('head')[0] || document.body;

function loadJsModule(url) {
    var script = document.createElement('script');
    script.charset = 'utf-8';
    script.type = 'text/javascript';
    script.onload = script.onreadystatechange = function() {
        if (!this.readyState || this.readyState === 'loaded' || this.readyState === 'complete') {
            /*
             * 加载逻辑, callback为define的回调函数, args为所有依赖模块的数组
             * callback.apply(window, args);
             */
            script.onload = script.onreadystatechange = null;
        }  
    };
}
```
   对于文本文件，直接用**ajax**实现。代码如下：

```javascript
var xhr = window.XMLHttpRequest ? new XMLHttpRequest() : new ActiveXObject('Microsoft.XMLHTTP'),
      textContent = '';
	  
xhr.onreadystatechange = function(){
    var DONE = 4, OK = 200;
    if(xhr.readyState === DONE){
        if(xhr.status === OK){
             textContent = xhr.responseText; // 返回的文本文件
        } else{
            console.log("Error: "+ xhr.status); // 加载失败
        }
    }
}

xhr.open('GET', url, true);// url为文本文件的绝对路径
xhr.send(null);
```	

#### 2.  模块依赖管理
一个模块的加载过程如下图所示。
![](https://haitao.nos.netease.com/e9bc3b86-2069-4e53-a04f-c7275bbf6ad8.jpg)
- 状态管理
从上面可以看出，一个模块的加载可能存在以下几种可能的状态。
1. 加载(load)状态，包括未加载(preload)状态、加载(loading)状态和加载完毕(loaded)状态。
2. 正在加载依赖(pending)状态。
3. 模块回调完成(finished)状态。
因此，需要为每个加载的模块加上状态标志(status)，来识别目前模块的状态。

- 依赖分析
在模块加载后，我们需要解析出每个模块的绝对路径(path)、依赖模块(deps)和回调函数(callback)，然后也放在模块信息中。模块对象管理逻辑的数据模型如下所示。
```javascript
{
	path: 'http://asdas/asda/a.js',
	deps: [{}, {}, {}],
	callback: function(){ },
	status: 'pending'
}
```

- 依赖循环
模块很可能出现循环依赖的情况。也就是a模块和b模块相互依赖。依赖分为**强依赖**和**弱依赖**。**强依赖**是指，在模块回调执行时就会使用到的依赖；反之，就是**弱依赖**。对于**强依赖**，会造成死锁，这种情况是无法解决的。但**弱依赖**可以通过现将一个空的模块引用注入让一个模块先执行，等依赖模块执行完后，再替换掉就可以了。**强依赖**和**弱依赖**的例子如下：

```javascript
//强依赖的例子
//A模块
define(['b.js'], function(B) {
  // 回调执行时需要直接用到依赖模块
   B.demo = 1;
   // 其他逻辑
});
//B模块
define(['a.js'], function(A) {
   // 回调执行时需要直接用到依赖模块
   A.demo = 1;
   // 其他逻辑
});

```

```javascript
// 弱依赖的例子
// A模块
define(['b.js'], function(B) {
    // 回调执行时不会直接执行依赖模块
    function test() {
        B.demo = 1;
    }
    return {testFunc: test}
});
//B模块
define(['a.js'], function(A) {
    // 回调执行时不会直接执行依赖模块
    function test() {
        A.demo = 1;
    }
    return {testFunc: test}
});

```

#### 3. 对外暴露**define**方法
对于define函数，需要遍历所有的未处理js脚本(包括**内联**和**外联**)，然后执行模块的加载。这里对于**内联**和**外联**脚本中的define，要做分别处理。主要原因有两点：
1. 内敛脚本不需要加载操作。
2. 内敛脚本中define的模块的回调输出是不能作为其他模块的依赖的。


```javascript
var handledScriptList = [];
window.define = function(deps, callback) {
    var scripts = document.getElementsByTagName('script'),
        defineReg = /s*define\s*\(\[.*\]\s*\,\s*function\s*\(.*\)\s*\{/,
        script;

    for (var i = scripts.length - 1; i >= 0; i--) {
        script = list[i];
        if (handledScriptList.indexOf(script.src) < 0) {
            handledScriptList.push(script.src);
            if (script.innerHTML.search(defineReg) >= 0) {
                // 内敛脚本直接进行模块依赖检查。
            } else {
                // 外联脚本的首先要监听脚本加载
            }
        }
    }
};
```

上面就是对实现一个模块化工具所涉及核心问题的描述。完整的代码[点我](https://github.com/ccfe/modules_load "点我")。