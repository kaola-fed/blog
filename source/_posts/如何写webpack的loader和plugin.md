---
title:如何写webpack的loader和plugin
date: 2017-11-13
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

Webpack是前端领域常见的打包工具. 平时使用webpack的机会也很多, 各种loader和plugin也引入的不少, 然而自己并没有实际开发过loader和plugin, 今天大概看了一下相关文档, 发现写发并不是很难, 下面介绍一下:

### Loader

Loader的作用, 是读入特定类型的文件, 然后输出一个模块, 给外部方使用.

举个栗子: 现在有一个entry: index.js 引用了一个namelist.txt 文件, 文件内容是一个产品名单, 每行一个名字, 现在我需要一个产品名单的数组, 接下来就来写一个namelist-loader, 读入namelist.txt, 输出名单数组. 
代码如下:

- index.js
``` javascript
import namelist from './namelist.txt'
console.info(namelist)
```
- namelist-loader.js
``` javascript
module.exports = function(source) {
    const result = `['` + source.replace(/\n/g, `',\n'`) + `']`
    return `export default ${JSON.stringify(result)}`
}
```

- webpack.config.js
``` javascript
// ...
module: {
    rules: [
        {
            test: /namelist.txt$/,
            use: [
                {
                    loader: path.resolve(__dirname, '../webpack-loader/namelist-loader.js'),
                    options: {
                        a: 123,
                    },
                },
            ],
        },
    ]
}
// ...
```

目录结构:
```
src-------
        |
        |
        index.js
        namelist.txt
      
build-----
        |
        |
        webpack.config.js
      
webpack-loader----
                |
                |
            namelist-loader.js
```

OK, 搞定, 跑起来以后, index引入的将是一个字符串数组, 每个元素对应产品名字.

有一些loader可以在配置中添加一些配置数据, 即`options`字段对应的数据, 想要在loader中读取到配置信息, 需要调用一个API, 代码如下:

``` javascript
import { getOptions } from 'loader-utils';

export default function loader(source) {
    const options = getOptions(this);
   // ......
};
```

### Plugin

相比于loader, plugin更复杂些. 

Loader只需要提供一个读入文件内容, 导出模块的函数即可, plugin则不同, 除了需要提供一个函数(姑且称之为主函数)外, 还要在函数的prototype上挂一个`apply`方法. 

主函数的入参是可选的, 即plugin的配置信息; apply方法的入参是compiler对象, 这个compiler对象来自webpack底层, 包含webpack环境所有的信息, 包括loader, plugin, 和各种配置参数. 

apply 方法在webpack编译器安装plugin时执行一次

一个简单的plugin代码如下:
```javascript
function HelloWorldPlugin(options) {
  // Setup the plugin instance with options...
}

HelloWorldPlugin.prototype.apply = function(compiler) {
  compiler.plugin('done', function() {
    console.log('Hello World!'); 
  });
};

module.exports = HelloWorldPlugin;
```
值得一提的是, webpack中大部分的对象都集成于`Tapable`类, 这个类对外提供了plugin接口, 方便用户在各种编译阶段插入逻辑.

使用plugin时, 只需new一个plugin实例, 塞到webpack配置中的plugins数组中即可, 如下:
``` javascript
var HelloWorldPlugin = require('hello-world');

var webpackConfig = {
  // ... config settings here ...
  plugins: [
    new HelloWorldPlugin({options: true})
  ]
};
```

通过上面提到的`compiler`对象, 我们还可以获取到webpack的compilation对象, 此对象中保存了webpack在编译期间的各种信息, 如当前模块资源的状态, 编译后的资源, 发生改变的文件和监听的依赖等. 

在启用webpack development middleware的情况下, 当有文件发生变化时, webpack就重新执行一次编译, 随即产生一个新的compilation. 通过这个对象, 可以向webpack编译期间的各种生命周期添加钩子, 以此来实现plugin的功能.

```javascript
compiler.plugin("compilation", function(compilation) {
    console.log("The compiler is starting a new compilation...");

    compilation.plugin("optimize", function() {
          console.log("The compilation is starting to optimize files...");
    });
});
```

如果plugin的逻辑中存在异步操作, 在回调中, 除了compilation之外, 还要另一个callback作为第二个入参. 用来在plugin逻辑执行完成后通知webpack, 继续执行后续的逻辑

举例如下:
```javascript
function HelloAsyncPlugin(options) {}

HelloAsyncPlugin.prototype.apply = function(compiler) {
  compiler.plugin("emit", function(compilation, callback) {

    // Do something async...
    setTimeout(function() {
      console.log("Done with async work...");
      callback();
    }, 1000);

  });
};

module.exports = HelloAsyncPlugin;
```

可供plugin执行插入的地方有很多, 这里介绍一个: `emit`
`emit`是webpack执行完编辑, 即将在目标位置生成打包文件的阶段, 此阶段也是所有生命周期中最后一个可以获取到 `assets` 数组的阶段

webpack水很深, 这篇文章仅仅提供了对loader和plugin最粗浅的认识, 可能存在错误, 望大家包涵. 更多webpack使用姿势, 敬请期待.

### ref
[writing a loader](https://webpack.js.org/contribute/writing-a-loader/)
[writing a plugin](https://webpack.js.org/contribute/writing-a-plugin/)
[how to write a plugin](https://github.com/webpack/docs/wiki/how-to-write-a-plugin)
[plugins](https://github.com/webpack/docs/wiki/plugins)