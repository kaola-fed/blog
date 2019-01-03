---
title: 由element-ui升级引发的连锁反应
date: 2017-12-21
---


**背景**： 一个基于vuejs的后台工程

**项目相关依赖**：
    
    | - element-ui@1.2.3
    | - vue@2.3.0
    | - vue-loader@10.3.0
    | - vue-style-loader@2.0.0
    | - vue-template-compiler@2.3.0
    ...
    
**事件**：`element-ui`升级为@2.0.8
> 话说，此次升级，组件库的样式，都有相应调整；为了保持跟之前版本的UI一致，不得不去重置了一下；

**现象**：

本地开发时（`npm run dev`）无特殊问题；本地打包（`npm run build`）即报如下error：

![image](https://haitao.nos.netease.com/a85f453e-21e1-4b76-9b63-84b99bdc2249.jpg)

**排查过程**：

查看该打包出来的文件定位到该行数，发现是箭头`=>`处报错；以为是load配置问题，没有支持es6的语法；但查看`webpack.base.conf.js`的配置如下：
```
...
{
test: /\.js$/,
loader: 'babel-loader',
include: [resolve('src'), resolve('test')].concat(
  process.env.NODE_ENV === 'production'
    ? [resolve('node_modules/element-ui/src/utils')]
    : []
)
}
...

```
确实是有设置babel-loader，.babelrc内设置了presets为"stage-2"; 箭头函数一直有在使用，之前build并没报过此错；且以上的配置，在生成环境下会把element-ui的代码也经过babel-loader，之前的低版本也无此问题；猜想是element-ui升级导致的；

遂谷歌之~

查到网友的修复方案：
http://blog.csdn.net/hayre/article/details/77231568

于是按此法修改了webpakc的配置；
```
...
{
test: /\.js$/,
loader: 'babel-loader',
include: [resolve('src'), resolve('test')].concat(
  process.env.NODE_ENV === 'production'
    ? [resolve('node_modules/.2.0.8@/element-ui/src'), resolve('node_modules/.2.0.8@/element-ui/src')]
    : []
)
}
...

```
再次打包，成功！！！

---

But！!

让其他小伙伴pull代码重试，他们居然还是报一样的错啊！！what a f**k!!（此时心里想的是，肯定是他们的电脑有问题，系统问题，你看，我这边是正常打包的啊）
此时此刻，不如试试`重启大法`吧~~

冉鹅，并没有什么懒用

淡定！先忙别的一会儿放空下~

N多个番茄时间之后...

再来看看
台式机没问题了，那到笔记本上重新试一次吧；

```
//拉最新代码
git pull

//删除本地依赖
rimraf node_modules/

// 重新安装
cnpm install

// build之
npm run build

```
居然出现了一样的错误！~
不得不承认，这确实是个没解决的问题；
于是查看了下node_modules目录，终于发现了问题所在，npm版本不一样！安装的包的目录结构也是不同；如下2个图，不通npm版本安装下来的包目录是不一样的；

![image](https://haitao.nos.netease.com/c22d6db4-a5ce-489b-8ca8-ea0ecb560bce.png)

![image](https://haitao.nos.netease.com/22b15fef-1db3-43fa-a900-a814161bca3d.png)


[npm 是如何影响 node_modules 的目录结构的 ？](https://segmentfault.com/a/1190000007681042)此文很好的梳理了不同的npm版本会生成怎样的node_modules的目录；

于是升级了笔记本上的npm版本（v5.5.1）后，重新打包，笔记本上也正常了！

总结：
- 要配置babel-loader来处理element-ui里的es6语法；
- 配置中使用的element-ui的包路径要以实际包所在的路径为准；（npm不同版本所装的包路径是不一样的）

> 另外：
element-ui升级到`2.0.7`之后，需要相应的升级`vue`和`vue-template-compiler`两个包，且这两个包版本要一致；

后续：部署到测试环境时 再一次遇到了问题；因为测试环境的npm版本比较旧（v3.10.9），遂将babel-loader的配置退回到老版本的包目录形式，即不带版本号的目录名：

```

{
    test: /\.js$/,
    loader: 'babel-loader',
    include: [resolve('src'), resolve('test')].concat(
      process.env.NODE_ENV === 'production'
        ? [resolve('/node_modules/element-ui/src'), resolve('/node_modules/element-ui/packages')]
        : []
    )
}

```

以上，一场升级引发的蝴蝶效应；
 by [lzf](https://github.com/lzf0402)

