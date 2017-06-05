---
title: 自定义webpack配置实战总结
date: 2017-05-24
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
-->

## 背景

在优惠券二期的任务中，需要进行整体重构。 在和[yubaoquan](https://github.com/yubaoquan)讨论之后，我们希望的技术栈是这样：

- 打包用webpack2
- js用es2015
- css用mcss
- js框架regularjs
- ui框架nek-ui(基于regularjs)

上面这些问题不是很大。 

有个比较麻烦的地方就是，前端和后端的连接方式有所改变，详情请看下文。

<!-- more -->

## 以前的方式

此项目是非spa项目，所以会有多个页面。

比如，根目录下有2个页面：

```ruby
/WEB-INF/ftl/pageA.ftl
/WEB-INF/ftl/pageB.ftl
```

pageA中引入`js/pageA/entry.js`，`entry.js`再引入业务js。pageB同理。

后端controller中连接到pageA的方式很简单，只要`return "/WEB-INF/ftl/pageA"`这样即可。

备注：
ftl文件：是freemarker（后端用的模板引擎）的模板文件，可以理解为html页面

## 现在的方式

与之前的不同就是， 每个entry.js都要经过一次webpack打包，再重新注入到ftl中，这里会面临2个问题：

- 如何将打包完成的entry.js注入到ftl文件中
- 如何配置webpack多入口

项目不断扩大，添加了新页面，配置又要重新写一次吗？迎面而来第3个问题：

- 如何动态生成webpack配置

## 解决方案

### 如何将打包完成的entry.js注入到ftl文件中

由于之前接触过[`htmlwebpackPlugin`](https://github.com/jantimon/html-webpack-plugin)插件，可以动态的生成html文件，并将entry.js插入到html文件中。那么生成ftl文件也应该没问题，查看了文档之后，确实只需要修改filename就行了。

### 如何配置webpack多入口

在github上找到了[答案](https://github.com/jantimon/html-webpack-plugin/issues/218#issuecomment-183066602)

简单解释下：
- entry中加入多个字段如page1, page2;
- output中路径加入[name]的动态字段
- htmlwebpackPlugin配置中，设置chunks字段未入口的配置的字段即可

### 如何动态生成webpack配置

大家熟悉的`webpack.config.js`文件：
```js
module.exports = {
    entry: "./entry.js",
    output: {
        path: __dirname,
        filename: "bundle.js"
    },
    module: {
        loaders: [
            { test: /\.css$/, loader: "style!css" }
        ]
    }
};
```

可以看出这个是commonjs语法，导出一个对象，那我们就可以对这个对象进行一些预处理，最后再导出这个经过处理的新对象即可。

就像这样：
```js
let myConfig = {}

myConfig.entry = "./entry.js"
// ....anything you want
module.exports = myConfig
```

## 实际开发中遇到的问题记录

### commonjs导出异步对象

由于遍历文件夹，返回文件是一个异步操作，所以模块导出需要导出一个promise;

代码如下：

```js
module.exports = new Promise((resolve, reject) => {
    recursive(PAGE_PATH, (err, files) => {
        // Files is an array of filename
        let filerFilesFunc = file => /\.ftl$/.test(file);

        if (err) {
            console.log(err);
            return;
        }
        files = files.filter(filerFilesFunc);
        resolve(generateWebpackConfig(files, baseWebpackConfig));
    });
});
```

### ftl中使用相对路径，路径找不到

[stackoverflow上的答案](http://stackoverflow.com/questions/34620628/htmlwebpackplugin-injects-relative-path-files-which-breaks-when-loading-non-root)

需要设置`publicPath`属性为根目录（请根据实际需求修改）：
```js
output: {
    publicPath: '/' 
}
```

## 其他webpack插件介绍

[ExtractTextPlugin](https://github.com/webpack-contrib/extract-text-webpack-plugin): 可以把样式模块提取成单独的css文件
[CommonsChunkPlugin](https://webpack.js.org/plugins/commons-chunk-plugin/): 可以把共用的模块提取成单个文件，不同入口之间可以通用
[chunkhash](https://webpack.js.org/guides/caching/): 单独给文件创建hashes，缓存文件

## 结语

目前项目已上线，中间很多坑都踩过，其中很多问题都是[yubaoquan](https://github.com/yubaoquan)解决的，合作愉快（握手）。`webpack.config.js`代码在这[code here](https://github.com/jerryni/blog/blob/master/webpack.config.js)，供大家参考，有什么问题可以一起讨论

by [Kaola nrz](https://github.com/jerryni)