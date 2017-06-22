---
title: art-template学习笔记
date: 2017-06-12
---

> `art-template`是一个性能卓越的javascript模板引擎。除了高性能之外，Ta的语法简洁易懂、支持基础的js语法、能满足绝大多数的使用场景，而且其出色的调试功能，编写模板再无后顾之忧，能在模板中定位语法错误。在前后端分离方案中，几经对比，最后选用了[art-template](https://github.com/aui/art-template)这个前后端通用的轻量的模板。

<!-- more ->

### 选用模板的几大原则

- 性能；毕竟是要用在实际项目中的，性能差的，必弃之；
- 语法功能方面：
    - 基本的条件表达式、循环遍历等功能，是必备的；
    - 模板嵌套，include子模板等功能，即模板重用功能也是必备的；
    - 支持模板语法与js语句混写，算数运算、逻辑运算、比较运算等在做模板中能做常规处理也很重要；
    - 支持功能函数；在变量输出前，需要做些特定的处理，如日期format等，这种使用场景不在少数，作为一个功能完善的模板，也是必不可少的。
    - 语法简洁、可读性强；
- 模板调试，加分项；
- 支持主流的node框架以及打包工具等；
- 模板引擎活跃度高、持续维护、有完整的文档等；


当然，`art-template`都满足以上条件，使用之后也证实这是一个开发体验很棒的模板。下面来介绍下Ta。

---

### 语法

*art-template 同时支持两种模板语法。标准语法可以让模板更容易读写；原始语法具有强大的逻辑处理能力。下文均为标准语法。*

#### 内容输出

```
{{value}}
{{data.key}}
{{data['key']}}
{{a ? b : c}}
{{a || b}}
{{a + b}}
```
支持三元运算符、逻辑运算符等；


#### 不转义输出

```
{{@value}}
```
不进行转义的输出，存在xxs风险，要谨慎使用；


#### 设置变量
```
{{set tlt = data.content.title}}
```
类freemarker中的`assign`进行变量设置，在多次取层级较深的对象属性时，设置变量来使用还是比较清晰的。

#### 条件输出

```
{{if code >= 0}} ... {{/if}}

{{if topic.processState == 0}} 
... 
{{else if topic.processState==1}} 
...
{{else}}
... 
{{/if}}
```

#### 遍历输出

```
{{each hotDiscussionList as list}}
    <span class="userName">{{list.nickName}}</span>
{{/each}}
```

#### 子模板


```
{{include './header.html'}}
{{include './header.html' data}}
```
在父模板中，引入子模板，同时可以给子模板传入参数，如上；
在实际项目中，页头、页脚、公用模板就都可以提取出来作为子模板，在各个页面中引入进来，实现模板的公用；


#### 模板继承

```
{{extend './layout.html'}}
{{block 'head'}} ... {{/block}}
```
模板继承可以用来构建页面结构相似的基本模板“骨架”，快速构建多个相似页面，各自页面中实现差异部分。
实例：
layout.html

```
<!DOCTYPE html>
<html>
<head>
    {{include '../partials/meta.html'}}
    {{block 'title'}}<title>网易考拉海购</title>{{/block}}
</head>
<body id="kaola-{{pageName}}">
    {{block 'content'}}{{/block}}
    {{block 'footerWidget'}}
       	{{include '../partials/footer.html'}}
    {{/block}}
    {{block 'others'}}{{/block}}
</body>
</html>
```
home.html

```
{{extend '../../layouts/page.html'}}

{{block 'title'}}
    {{title}}
{{/block}}

{{block 'head'}}
    <link rel="stylesheet" href="custom.css">
{{/block}}

{{block 'content'}}
    <p>This is just an awesome page.</p>
{{/block}}

```

#### 过滤器

注册过滤器：

```
var template = require('art-template');
template.defaults.imports.dateFormat = function(date, format){/*...*/};
```

使用过滤器：
```
{{date | timestamp | dateFormat 'yyyy-MM-dd hh:mm:ss'}}
```

过滤器语法类似管道操作符，可以多个过滤器叠加；


### 模板变量
通过`template.defaults.imports`给模板中导入变量，通过 $imports ，可以访问到模板外部的全局变量与导入的变量。

模板中内置的变量有：

- $data 传入模板的数据
- $imports 外部导入的变量以及全局变量
- print 字符串输出函数
- include 子模板载入函数
- extend 模板继承模板导入函数
- block 模板块声明函数

### 注释
支持html注释，模板注释如下，不会在html中输出：

```
<%# 当前页面名称 %>
window.__pageName = "{{pageName}}";
```

### 性能

对于art-template高效的秘密，可以查阅作者的解释：[高性能JavaScript模板引擎原理解析](http://cdc.tencent.com/2012/06/15/%E9%AB%98%E6%80%A7%E8%83%BDjavascript%E6%A8%A1%E6%9D%BF%E5%BC%95%E6%93%8E%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)

---


### 参考文档
- [https://github.com/aui/art-template](https://github.com/aui/art-template)
- [https://aui.github.io/art-template/docs/](https://aui.github.io/art-template/docs/)

