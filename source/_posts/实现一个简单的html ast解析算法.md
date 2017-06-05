---
title: 实现一个简单的html ast解析算法
date: 2017-05-23
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
-->

## 概述

html ast解析算法的过程是将一段html字符串，解析成一个javascript对象。

<!-- more -->

## 示例字符串

```html
<div>
    <input r-value="{name}" />
    <p>
        <span></span>
        <span style="display:block;">
            描述信息2
            <span>{name}</span>
        </span>
    </p>
</div>
```

## 思路分析

整体实现分析：
 
1. 通过正则匹配出token，设置node，添加到ast对象中，同时截取掉剩余的字段
2. 主要的token: 开标签`<div>`,`<p>`等，自闭合标签`<input />`，闭合标签`</div>`, `</p>`等
3. 实现范围可以先从标签名开始，再添加属性，再解析文本(textNode)

代码思路：

1. 用一个数组储存树结构，children表示分支
2. 先匹配到一个`div`开标签，push进数组
3. 匹配到input自闭合标签放入div.children
4. 匹配到p标签放入div.children
5. 匹配到span放入p.children
6. 匹配到`</span>`结束标签，表示后面的标签需要加入到p.children而不是span.children
7. 递归到没有开标签为止

## 代码实现

分布解析的动态demo，代码每一步都有详细的解释:

[online demo](https://jerryni.github.io/algorithom/demos/html.ast.parser.html)

[github源码](https://github.com/jerryni/algorithom/blob/master/demos/html.ast.parser.html)

by [Kaola nrz](https://github.com/jerryni)