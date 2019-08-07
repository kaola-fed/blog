---
title: 从搭建工程讲到CSS Modules
date: 2017-09-04
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

### 背景

这周的主要的工作就是搭建新工程的架子，项目基于[vue-cli](https://github.com/vuejs/vue-cli)构建。基本的功能在脚手架里都已经具备，但是还是需要针对具体的业务场景来做一些定制。

### mock
以现在前后端分离的开发模式来讲，一个正常的开发流程大概是这样：
<img src="http://oksj4hknl.bkt.clouddn.com/develop-liucheng.png">

这样前端可以不用等后端接口完全写完才可以开发，前后端并行开发，提高团队效率。

而脚手架提供的功能只有一个*proxy*，不能满足我们的需求。所以需要我们自己写一个`mock-middleware`。大概是这样的一个功能，就不贴`mock-middleware`的代码了：

```
module.exports = () => {
    const argv = Array.prototype.slice.call(process.argv, 2);
    const proxyAddress = argv[0];
    if (!argv.length || proxyAddress === 'mock') {
        return mockMiddleware;
    } else {
        return proxyMiddleware(proxyAddress);
    }
}
```
最终的效果就是运行 `npm run dev mock` ，会将api请求（注意：这里只会代理api请求，而不代理静态资源）转发到mock数据。在前后端开发完成之后，可以运行`npm run dev {proxy_ip}`将api请求代理到某台开发服务器进行联调。

当然这篇文章的重点不是这个，而是想总结下在SPA应用里如何规范的写CSS。

### CSS模块化
故事的起点从组件库的选择说起，项目选择了开源的[element](https://github.com/ElemeFE/element) 作为组件库。而这套组件库并不符合考拉前端组的视觉规范，而目前也没有计划fork一条分支来维护，于是就采取了比较简单粗暴的做法，样式覆盖。先来看一段element中的样式：

```
@component-namespace el {
  @b progress {
    position: relative;
    line-height: 1;

    @e text {
      font-size:14px;
      display: inline-block;
      vertical-align: middle;
      margin-left: 10px;
      line-height: 1;
    }
    @m circle {
      display: inline-block;
    }
  }
}  
```
可能有同学在刚看到这段样式的时候会有点蒙，但是仔细看@b，@e，@m，不就是bem规范么。
<img src="http://oksj4hknl.bkt.clouddn.com/yilianmengbi.jpeg">

#### BEM规范
关于BEM规范，网上有很多文章介绍。知乎上也有一篇[文章](https://www.zhihu.com/question/21935157)来讨论其优劣，这里不去讨论BEM规范的好处和坏处，我觉得只要在一个工程里约定好一种规范并严格执行，这样总是不会错的。

一方面为了和组件库的CSS规范保持统一，另一方面我个人觉得BEM的优点还是大于缺点，因此在项目里也准备按照这个规范来写。

那么问题来了，浏览器是不认识上面这些@b，@e，@m的语法的，这个时候就需要[postcss](https://github.com/postcss/postcss)来帮忙了。

#### postcss
postcss和gulp，webpack等工具一样，他本身并不是一种预处理器或者后处理器，而是通过各种插件来完成转换（比如很流行的Autoprefixer就是它的一个插件）。

[postcss-salad](https://github.com/ElemeFE/postcss-salad)可以认为是一个postcss插件的集合，支持最新的css语法，一些sass嵌套的语法以及bem转化。这时候先来一份配置文件`postcss.config.js`：

```
module.exports = {
  plugins: [
    require('postcss-salad')({
      browsers: ['ie > 9', 'last 2 versions'],
      features: {
        bem: {
          shortcuts: {
            component: 'b',
            descendent: 'e',
            modifier: 'm'
          },
          separators: {
            descendent: '__',
            modifier: '--'
          }
        }
      }
    })
  ]
}
```

bem之间的连接符可以自定义，这边是为了和element的组件保持一致。这时候我们可以先通过[postcss-cli](https://github.com/postcss/postcss-cli#context)这个工具来看一下效果。我们就用上述那段css来用postcss+postcss-salad进行转换，得到结果如下：


```
.el-progress {
    position: relative;
    line-height: 1
}

.el-progress__text {
    font-size: 14px;
    display: inline-block;
    vertical-align: middle;
    margin-left: 10px;
    line-height: 1
}

.el-progress--circle {
    display: inline-block
}
```

已经达到了我们预想的效果，那么接下来要想在webpack中使用postcss肯定会需要一个loader，也就是[postcss-loader](https://github.com/postcss/postcss-loader)。

然后将刚才的postcss.config.js作为postcss-loader的配置文件导入。

```
{
  loader: 'postcss-loader',
  options: {
    config: {
      path: 'path/to/postcss.config.js'
    }
  }
}
```

至此，就可以在webpack项目中，用这种嵌套的bem语法来进行css的书写了。

#### CSS Modules
故事还没有结束，大家都知道在写单页面应用的时候，一个常见的需求是希望组件间的css作用域是互相隔离的。这时候第一反应想到就是*Scoped CSS*，vue-loader也是支持Scoped CSS的（实际上对CSS进行转换的还是postcss）。那么Scoped CSS是如何来处理这个问题的呢：

```
<style scoped>
.example {
  color: red;
}
</style>

=>

<style>
.example[_v-f3f3eg9] {
  color: red;
}
</style>
```
可以看到Scoped CSS会在class后面加上一段hash，从而来实现CSS作用域的隔离，但是在实践过程中，会发现几个问题：

1. Scoped CSS将同时作用在父组件和子组件上，也就是无法做到父子组件样式的隔离。
2. 由于Scoped CSS改变了DOM中的class，因此就无法在组件内去覆盖element组件的全局样式了。

这时候就需要[CSS Modules](https://github.com/css-modules/css-modules)出场了，CSS Modules是目前CSS模块化方案中被接受度较高的一种方案，网上也有很多的文章去介绍它。同样的，我们需要先看下CSS Modules能做什么事情呢？

刚才讲到，postcss有很多的插件，那么肯定也会有CSS Modules的[插件](https://github.com/css-modules/postcss-modules)了。我们同样通过之前的postcss-cli来进行测试，首先配置上这个插件：

```
require('postcss-modules')({
  generateScopedName: '[local]--[hash:base64:5]'
})
```

> 这里的[local]代表类名，[hash:base64:5]按照给定规则生成的hash值，还可以用的变量有[name]代表标签名，[path]代表路径等
> CSS Modules 生成的class可以自定义规则，因此也可以用自定义的规则，而不用bem规范，可以看项目的具体情况来定

还是同样那段css代码，看下转换的结果：

```
.el-progress--3tuDF {
    position: relative;
    line-height: 1
}

.el-progress__text--1W8n3 {
    font-size: 14px;
    display: inline-block;
    vertical-align: middle;
    margin-left: 10px;
    line-height: 1
}

.el-progress--circle--3OD0E {
    display: inline-block
}

```

跟我们预设的样式规则一致，这样就可以解决上述的第一个问题，做到每个组件内的样式是唯一的。

但是我们发现CSS Modules同样会改变class，那么也同样无法在组件内覆盖element组件的全局样式，这时候可以用CSS Modules的全局样式写法，`:global{.class}`：

```
:global(.el-progress) {
  position: relative;
  line-height: 1;
}

=>

.el-progress {
  position: relative;
  line-height: 1;
}
```

这样就不会在class上加上hash后缀，从而可以达到覆盖全局样式的目的。

####  CSS Modules 在Vue+webpack项目中的实践
首先在Vue-loader的配置文件中配置生成class规则：

```
cssModules: {
    localIdentName: '[local]--[hash:base64:5]',
    camelCase: true
}
```

然后在组件内通过在style上添加module打开CSS Modules。

```
<style module>
</style>
```

css-loader 会将一个 `$style`对象注入到当前组件。所以在实际中使用大概是这个样子：

```
<header :class="$style['titan-header']">
</header>
```

有时候我们会有这样的用法：

```
<div :class="{ 'active': selectedIndex == index} ">
</div>
```
这时候样式就会作为对象的属性名，而我们知道用了CSS Modules，就必须用`$style.active`来替换`'active'`，还好我们有ES6！
ES6有一个特性是用双括号支持用计算属性作为属性名，也就是这样：
```

<div :class="{ [$style.active]: selectedIndex == index} ">
</div>
```


### 总结
bem+CSS Modules在新项目的实践已经有一周的时间，并没有发现什么问题，才写下这篇文章来总结。以上只是我在本次工程搭建过程中的一些总结，并不保证观点完全正确，供大家参考。

