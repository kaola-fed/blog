---
title: JS模块化 - 浅谈 CommonJS require 函数实现
date: 2017-03-26
---


## 何以模块化 
> 最早的前端，没有模块加载规范，  
> 只能在HTML中通过\<script\>来引入js文件，同时无法区分函数来源于哪个js文件，而且要用过多全局变量。
> 而随着前端工程复杂度的提升，使用这种方式已经无法满足日益增长的开发需求，js的模块化应运而生。

<!-- more -->

CommonJS 是属于 Node.js 的模块化方案，最早是叫 ServerJS，随着 Node.js 的火爆发展而成名的。Module1.0 规范在 Node.js 上实践的很好。

而 JavaScript 在当时（ES6 Modules 规范还未诞生）是没有模块化方案的，所以又更名从 CommonJS，想要统一服务端与客户端的模块加载方案。

但是，require 函数是同步的，在浏览器端由于网络的瓶颈而不适用。于是，AMD 和 CMD 的规范相继涌现，和 CommonJS 一起服务于 JavaScript 模块化。
而正是规则的不统一，这也是目前兼容方案 UMD 会出现的原因。

不过AMD 和 CMD 的浏览器端模块化，有很明显的问题：
1. 导致浏览器端请求数过多；
2. 受限于网络，所有模块都成功加载完只是一个承诺。

现如今，当打包这一环节被引入了前端工程化，CommonJS 以与服务端可以类库共用和 NPM(Node Package Manager) 这个后台的优势，成为了 es5 JavaScript 模块化的首选

本文内容，是从 `require` 函数这个切面，带读者了解到 CommonJS 模块化的原理，所有代码整理在 [git 仓库](https://github.com/ImHype/MockRequire)

## 简介
CommonJS 是一个旨在构建涵盖web服务器端、桌面应用、命令行app和浏览器JS的JS生态系统。

## 标准
CommonJS 的标准符合 [module1.1](http://wiki.commonjs.org/wiki/Modules/1.1) 规范，暴露给使用者的有三个全局变量：

1. require 是一个全局方法，用来加载模块  
2. exports 一个全局对象，用来导入模块的属性或方法  
3. module 一个全局对象。涵盖当前模块的必要信息，有一个只读的id属性，有一个uri属性，还有其它的一些命名规范，可以查看 [CommonJS 规范的文档](http://wiki.commonjs.org/wiki/CommonJS)。

## 面向 `require` 这个切面
### 写一个 `main` 函数
本文讲的是如何模拟一个 $require 函数，先来捋一捋 require 函数的主逻辑

![](http://www.infoq.com/resource/articles/nodejs-module-mechanism/zh/resources/image1.jpg)

根据这个逻辑，我们先写一个 `main` 函数，以及定义一些需要的接口

```javascript
// 缓存模块 require 的结果
var cached = {}
// 初始化的时候就会装载进来
var coreModule = {}

// 获取入口 js
// 用于后续的相对路径转绝对路径
var ENTRYJS = path.resolve(process.cwd(), process.argv[1])
var stack = new Stack(getParent(ENTRYJS))

function $require(pathname) {
    // 是否已存在缓存
    if (cached[pathname]) {
        return cached[pathname]
    }

    // 是否核心模块
    if (coreModule[pathname]) { 
        cached[pathname] = coreModule[pathname]
        return cached[pathname]
    }
    
    // 获取到模块的真实位置
    // 1, 如果是绝对路径，则转换相对路径
    // 2. 如果是包，则查找是否存在 package.json，并读取 main 属性
    var targetModule = getModuleLocation(stack, pathname)

    if (-1 === targetModule) {
        throw new Error('模块' + pathname + '未找到') // 包未找到
    }

    // 1. 如果是目录，则添加 /index
    // 2. 如果是文件，先检测文件是否存在
    // 3. 若不存在，则检测添加后缀名后的文件是否存在， ['', '.js', '.json', '.node']
    var targetFile = getRealPath(targetModule)
    if (-2 === targetFile) {
        throw new Error('模块' + pathname + '未找到') // 文件未找到
    }

    stack.push(getParent(targetFile))

    // 找到具体的文件，并执行，执行完成赋值到缓存中
    cached[pathname] = _require(targetFile)
    stack.pop()
        
    return cached[pathname]
}


// 需要实现的接口
// 数据结构 - 栈
function Stack() {}
// 获取文件的所在目录的绝对路径
function getParent(pathname) {}


// 查找模块的所在位置
function getModuleLocation(pathname) {}
// 从模块的所在位置，获取到真实的 js 路径
function getRealPath(pathname) {}


// 纯粹的模块执行处理函数
function _require(pathname) {}

```

至此已经有一个不可执行的 require 函数了，逻辑中会依次判断是否有缓存，是否核心模块，加载相应文件以及加载执行模块并缓存。

可以看到没有具体实现，只有接口被定义出来，这种编码方式，同样可以借鉴到在其他的开发需求中：
1. 在开始编码前，进行尽可能合理的功能模块划分，可以让代码逻辑清晰，减少重复步骤 **(DRY)**，并增强后期的代码可维护性
2. 定义你需要哪些接口。如果是比较复杂的功能，且不是独立开发的话，这一环节做的好坏，合理地划分与合理地分配，决定团队合作开发是否可以配合恰当


### 逐一实现接口
这一步骤，主要是对上述过程需要的接口进行实现。  
思想上是一个分而治之的思想，实现一个很复杂的东西比较困难，但是实现具体的功能要求，且在一定输入输出的限制下，每个人都能轻易的写出符合需求的算法，并进行调优。

#### 1. 数据结构与工具函数类
##### a. 栈 -- 存储当前模块的所在目录
```javascript
function Stack(...args) {
    this._stack = new Array(...args);
}
Stack.prototype = {
    top: function () {
        return this._stack .slice(-1)[0]
    },
    push: function (...args) {
        this._stack.push(...args)
    },
    pop: function () {
        this._stack.pop()
    },
    constructor: Stack
}
```
这个栈的作用是存放当前模块的所在目录，用于模块内 `require` 函数传入相对路径时，解析成绝对路径

#### b. 获取文件所在目录
```javascript
function getParent(pathname) {
    return path.parse(pathname).dir
}
```

### 2. 具体的模块文件查找逻辑
![](http://www.infoq.com/resource/articles/nodejs-module-mechanism/zh/resources/image2.jpg)
#### a. 检测模块类型与定位包的位置
这个函数要做下面的事情
1. 检测模块类型：绝对路径，相对路径 或是 在 `node_modules` 内
2. 如果是模块，则需要就近寻找 node_modules 有无这个模块，并且读取 `pacakge.json` 的 `main`属性

```javascript
function getModuleLocation(pathname) {
    var moduleType = getModuleType(pathname)
    var map = {
        [MODULE_TYPE.ABSOLUTE_PATH]: function () {
            return pathname
        },
        [MODULE_TYPE.RELEALITIVE_PATH]:: function () {
            return path.resolve(stack.top(), pathname)
        },
        [MODULE_TYPE.IN_NODE_MODULES]: function () {
            var parent = stack.top()
            while (!fs.lstatSync(path.resolve(parent, 'node_modules', pathname))) {
                parent = getParent(pathname)
                if (!parent) {
                    return -1
                }
            }
            // 从包名 到 绝对路径
            pathname = path.resolve(stack.top(), 'node_modules', pathname)
            
            // 输出包的 main 文件
            pathname = getPackageMain(pathname)
            
            return pathname
        }
    }

    return map[moduleType]()
}
/**************** 分割线 *******************/
// 以下部分，是 getModuleLocation 自身需要实现的接口，往往是开发过程中自行提炼的，其他模块不通用
var MODULE_TYPE = {
    "ABSOLUTE_PATH": 1,
    "RELEALITIVE_PATH": 2,
    "IN_NODE_MODULES": 0
}

function getModuleType(pathname) {
    if (path.isAbsolute(pathname)) {
        return MODULE_TYPE.ABSOLUTE_PATH
    }

    if (pathname.MODULE_TYPE('.')) {
        return MODULETYPE.RELEALITIVE_PATH
    }

    return MODULE_TYPE.IN_NODE_MODULES
}

function getPackageMain(pathname) {
    try {
        var packageJson = fs.readFileSync(path.resolve(pathname, 'package.json')).toString('utf-8')
        var json = JSON.parse(packageJson)    
        if (json.main) {
            return path.resolve(pathname, json.main)
        }
    } catch(e) {
        return pathname
    }
}
```

#### b. 定位引用的真实路径
1. 如果是目录，则添加'/index'后缀
2. 对 '.js'，'.node', '.json' 可能有的后缀省略进行补齐

```javascript
function getRealPath(pathname) {
    var list = ['', '.js', '.json', '.node']
    var stats

    try {
        stats = fs.lstatSync(pathname)    
    } catch (e) {
        return -2
    }

    if (stats) {
        if (stats.isFile()) {
            return pathname    
        } else if (stats.isDirectory()) {
            pathname = path.join(pathname, 'index')
        }
    }

    for(var i = 0; i < list.length; i++) {
        var fullname = pathname + item
        try {
            fs.lstatSync(fullname)
            return fullname
        } catch (e) {}
    }

    return -2
}
```

### 3. _require 函数
这个函数是 CommonJS 模块化的核心体现，理解这个函数，对 `module` 和 `exports` 的实际使用也会有帮助

1. Node.js 的模块化实质是为每个模块的 js 包裹一层 'function module_exports(){}' 用以隔离作用域； 
2. module / exports / require 被作为传参而传入，而 exports 实质是 module.exports 的引用
3. __dirname / __filename ，其实是这个模块内被先于执行函数定义的变量 
3. JS `new Function` 的特性常用来动态定义和执行某个方法，这边也同样用来执行模块

```javascript
function _require(pathname) {
    // 定义一个Module对象
    var Module = function() {
        this.exports = {}

        // 添加其他必须属性，如 id
    }
    
    // 引入nodejs 文件模块 下面是nodejs中原生的require方法
    var fs = require('fs')

    // 同步读取该文件
    var sourceCode = fs.readFileSync(pathname, 'utf8')

    // 字符串转换成函数
    var moduleExportsFunc = new Function('module', 'exports', `${sourceCode}; return module.exports;`)

    // 实例化一个Module 里面有一个exports属性
    var module = new Module()

    // 把module 和 它内部的module.exports都作为参数传进去 
    // 并得到挂在到module.exports 或 exports上的功能
    var res = moduleExportsFunc(module, module.exports)

    // 最终我们拿到了path代表的文件模块提供的API
    return res 
}
```

留下一些问题留给同学们思考：
1. 如果出现异步 `require` 的情况，由于当前模块已经执行完，会清空存储模块目录的 stack ，会出现相对路径查找失败的问题，如何解决？
2. 当发生循环依赖的时候，`CommonJS` 内部的加载流程是是否会陷入死循环，如果不会那会带来什么其他影响？（且听下回分解）

## End
本文是我参考一些 Node.js require 特性后，利用 JavaScript 简单的实现后的总结，主要提供一个思路。
本文中实现的 $require 函数存放在了 git 仓库 [MockRequire](https://github.com/ImHype/MockRequire) 中


至此，我们粗暴地模拟了 Node.js 中的 require，多谢阅读，如有疑问，欢迎指出