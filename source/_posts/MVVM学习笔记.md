---
title: MVVM学习笔记
date: 2017-05-30
---

## 前言
最近在学习MVVM的实现原理，刚好在sf上看到了[剖析Vue原理&实现双向绑定MVVM](https://segmentfault.com/a/1190000006599500)一文，写的非常好，摘出Vue.js中的部分源码，改造后完成了一个简单的MVVM实现。实现了双向数据绑定，我自己在学习的过程中，也照着这篇文章中的源码重新实现了一遍。不同之处在于，我尽量将原来的实现写成了ES6的写法，比如使用`class`代替构造函数，将`observer`,`dep`,`watcher`,`compiler`分成不同的模块，然后使用`import`,`export`来互相引入，导出，最后使用[rollup-babel-lib-bundler](https://github.com/frostney/rollup-babel-lib-bundler)打包了一下。所以这篇文章是对上面文章的学习总结，不会写的很细。大家也可以读一下上面的文章，简单易懂。

<!-- more -->

## 整体结构
这个简易的MVVM总共由`index.js(入口文件)`,`compiler.js`,`dep.js`,`observer.js`,`watcher.js`几部分组成。

    .
    ├── README.md
    ├── dest
    │   ├── toy.es2015.js
    │   ├── toy.js
    │   └── toy.umd.js
    ├── examples
    │   └── index.html
    ├── package.json
    ├── rollup.config.js
    └── src
        ├── compiler.js
        ├── dep.js
        ├── index.js
        ├── observer.js
        └── watcher.js

`index.js`是整个框架的入口，比如我给这个框架起了个名字叫`Toy`，入口文件导出的其实就是`Toy`的构造函数：

    //引入其它模块
    import { observe } from './observer.js'
    import { Compiler } from './compiler.js'
    import { Watcher } from './watcher.js'

    //具体实现
    class Toy {
        constructor(options){
            //...
        }
    }

    //导出模块
    export { Toy }

初始化的过程分两步：
1. 劫持监听所有属性，通过`Object.defineProperty`将数据变成响应式的，同时在`get`和`set`上做一些手脚。
2. 编译html模板，事实上我们在使用框架时写的html已经填充了很多框架自己的指令，语法，所以要先进行编译替换才能正确展示视图。

实现所有属性的监听就是通过`Object.defineProperty`递归地定义所以属性。每一个对象都会有一个对应的`Observer`实例，其中的每一个属性都对应有一个`Dep`的实例`dep`，`dep`使用自增的`uid`标识，作用是记录这个属性被那些订阅者(`Watcher`的实例)订阅了，好在属性变化时，通过遍历`dep.subs`去通知所有订阅了这个属性的`watcher`去做对应的更新。

实现`Compiler`就是对带有框架特殊API的模板进行编译，指令解析。同时将DOM与数据关联起来(其实是通过Watcher实现的)。

## 本质上说

每个部分负责的事情我是这样理解的：
- **index.js** 框架的入口，提供对外的构造函数。
- **observer.js** 将数据变成响应式，同时通过`dep`收集依赖(Watcher实例)。
- **dep.js** 收集依赖用的，在`get`中收集依赖，在`set`中通知对应依赖更新。
- **watcher.js** 数据的订阅者，一个Watcher的实例由`vm`,`exp`,`cb`,`deps`等几部分组成，`vm`是对ViewModel的引用，触发`get`方法将`watcher`自身添加至`dep`的`subs`中时会用到，`exp`则是当前Watcher实例监听的表达式，即数据的`key`，`cb`则是更新数据的回调。
当`vm`的数据改变后，会触发对应的`set`方法，这个属性对应的`dep`会通知所有的`subs`去执行自身的`update`方法，而这个`update`方法的内容其实只是`this.cb.call(this.vm, value, oldValue)`，`cb`实际上是调用了`updateFn`(在`compiler.js`中绑定的)，这时才将DOM的数据真正更新。
- **compiler.js** 编辑DOM模板，并为每个`node节点`通过`new Watcher`的方式将属性表达式`exp`，`updateFn(真正更新DOM的函数)`与`node`关联，然后配合响应式数据就做到了`view`与`model`的双向绑定。

所以整个框架的运行过程是这样的：

1. `observe`所有数据，改写了每个数据的get和set方法，并为每个数据关联了一个dep(通过闭包实现)。
2. `new Compiler`开始编译模板，编译过程中，可以提取出指令，`v-text`,`v-html`等，可以分析出事件函数`v-click`和绑定的表达式，这时通过`self.compileText(node, RegExp.$1)`,`self.compile(node)`将DOM节点和表达式建立关联。
3. 建立的关联，是DOM节点和数据表达式的关联，这一步是通过`new Watcher`实现的     
4. `new Watcher`的时候，Watcher实例会将Dep.target这个全局属性指向自身，然后出发一下需要监听属性的getter，这时`dep`会将Watcher实例添加到它的`subs`中，Watcher实例也会标记一下这个dep已经添加过自己了，防止重复添加。这时`dep`和Watcher实例已经关联起来了，数据的变化可以通知到对应的Watcher实例，Watcher实例的update方法会正确地更新DOM。

其实到这里，数据的双向绑定就已经实现了。

## 过程中学习到的一些细节
记录一些在学习过程中遇到的小tips，其实都是很基础的东西。
- [Node.textContent](https://developer.mozilla.org/zh-CN/docs/Web/API/Node/textContent): 表示一个节点及其内部节点的文本内容。之前一直都是用`innerText`的，看了MDN才知道`innerText`原来是IE私有的，`textContent`才是标准属性。而且`innerText`受样式影响，还会触发重排，所以还是用`textContent`代替吧。
- [Node.appendChild](https://developer.mozilla.org/zh-CN/docs/Web/API/Node/appendChild): 这个API有一个很有意思的行为：**如果被插入的节点已经存在于当前文档的文档树中,则那个节点会首先从原先的位置移除,然后再插入到新的位置.**，当时我在看`compiler.js`的`node2Fragment`方法：

```
    node2Fragment(el){
        let fragment = document.createDocumentFragment()
        let child
        while(child = el.firstChild){
            fragment.appendChild(child)
        }
        return fragment
    }
```

当时很不解为什么while循环能成按照预期执行，我在浏览器多次调用`el.firstChild`拿到的也始终是第一个子节点，看了这个API的文档才发现还有这么个行为！
- [Node.attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes): 可以方便地获取DOM节点的属性，返回值是一个对象，其中`name`是属性名，`value`是属性值。

## 最后
终于明白了简易MVVM框架的运作原理，也发现了一些底层API的知识，写成一些总结，这篇文章中没有贴很多代码去说实现，因为[剖析Vue原理&实现双向绑定MVVM](https://segmentfault.com/a/1190000006599500)一文已经很详细了，我也是按照这个去学习的，所以我记录的是我个人的一些思想上的总结，所以可能要先看代码才能了解。分享出来，希望能有人从中受益 :)





