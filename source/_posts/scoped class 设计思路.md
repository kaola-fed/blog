---
title: scoped class 设计思路
date: 2017-11-03
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

# scoped class 设计思路

`scoped class` 是为了解决样式文件中class命名冲突而设想的一种以打包过程中动态分析文件标注来生成不重名class的解决方案, 借鉴了vue的`scoped style`.

## 当前开发模式面临的问题

### 问题本质: 工程大, 多文件情况下维护类名唯一性比较困难

### 类名系统过于复杂

- 现有规则不好记, 不好写.
- 目前维护类不重名的机制是依靠最外层的类名不重复, 内层类名如何取名, 依据什么格式, 似乎没有统一的规范
- 为了避免同名而维护一个复杂的namespace系统, 起名时需要搜索若干样式文件确保不重名, 降低了开发效率;
- 样式前缀类似匈牙利命名法, 使开发变成了面向编译器编程.
- 由于中划线被占用到类名扩展系统中(据说下划线也不推荐使用), 迫使开发者只能使用驼峰命名类名, 甚至全部小写, 导致类名难以辨认, 意义不明;

1. nec官网上的前缀就有很多:
- .m-
- .u-
- .g-
- .f-
- .s-
- .z-

2. 实际开发中发现, 为了不与某些公用模块的类.m-冲突, 又引入了
- .n-

3. 还有代码中有, 且无(lan)从(de)考证的一些前缀
- .v-
- .w-
- .fur-
- .com-
- .js-
- .phone-

### 类名互相覆盖难以避免
- 目前大部分类仅仅依靠套在最外层的m-xxx来进行区分; 如果m-xxx出现同名, 或者有不套.m-xxx的类, 很容易产生同名污染
- 样式文件中绝大多数最外层类名都是.m-开头. 如此看来, 统一去掉.m-, 似乎也影响不大
- 一旦同名污染带到线上, 如果是一个页面的样式污染了另一个页面的样式, 难以第一时间被发现
- 如果多个样式文件使用了同名的选择器, 打包出来的样式文件会简单的包含多个同名选择器和样式, 如下:

style1.mcss
```css
.foo {
    color: red;
}
```
style2.mcss
```css
.foo {
    color: blue;
}
```
style3.mcss
```css
.foo {
    color: green;
}
```
output.css
```css
.foo{color:#008000;}
.foo{color:#00f;}
.foo{color:#f00;}
```

### 独立组件类名同名

- 在ftl中引入多个样式文件的情况下, 现有规则不能很好的避免同名类产生, 尤其是页面中引用了多个独立开发的公用组件时, 容易产生样式污染;



包括不限于

    1. 同模块类的类名污染
    2. 同扩展类的同序号污染
    3. 同元件类名污染
    4. 不使用现有规则, 自己随意取名造成的对其他文件中同名类污染
    5. 一个模块内部引用另一个模块, 造成的类名污染

如下:

component.mcss
```less
.m-container { // bug1
    .left {
        .tips {

        }
    }
    .right {

    }
}
.base.base-1 { // bug2

}
.u-btn { // bug3
    &.u-btn-1 {

    }
}
```

component-b.mcss
```less
.m-container {
    .left {
        .tips {

        }
    }
    .right {

    }
}
.base.base-1 {

}
.u-btn {
    &.u-btn-2 {

    }
}
```

wtf.mcss
```css
// bug4
.left {

}
.right {

}
.u-btn-1 {

}
.u-btn-2 {

}
.u-btn-3 {

}
```


### 模块引用造成的类名污染示例

#### 源文件

component.html
```html
<div class="m-component">
    <span class="bar">{#include this.$body}</span>
</div>
```
component.mcss
```less
.m-component {
    .bar {
        font-size: 12px;
    }
}
```
index.html
```html
<div class="m-container">
    <Component>
        <span class="bar">xxx</span>
    </Component>
</div>
```

index.mcss
```less
.m-container {
    .bar {
        font-size: 14px;
    }
}
```

#### 打包结果 (**.bar 被污染**)

index.html
```html
<div class="m-container">
    <div class="m-component">
        <span class="bar">
            <span class="bar">xxx</span>
        </span>
    </div>
</div>
```

index.css
```css
.m-component .bar {
    font-size: 12px;
}
.m-container .bar {
    font-size: 14px;
}
```
## 解决方案

假设某个页面中涉及如下模板文件和mcss文件:

| html | mcss |
|:---- | ----:|
|a.html|x.mcss|
|b.html|y.mcss|
|c.html|z.mcss|

生成后的css文件为 `combined.css`

打包过程中生成的html文件为 `a-temp.html` `b-temp.html` `c-temp.html`, (后续合并文件的步骤忽略)

## 声明
- 模板顶部添加注释, 声明要使用的scoped class所在的mcss文件

component.html
```html
<!-- scope="/path/to/component.mcss" -->

<div class="component">
    <span class="bar">{#include this.$body}</span>
</div>
```

- mcss文件顶部添加注释, 声明此文件中的选择器全部进行scoped处理

component.mcss
```less
/* @scoped */
.component {
    color: red;
    .bar {
        font-size: 12px;
    }
}
```

index.html
```html
<!-- scope="/path/to/index.mcss" -->

<div class="main">
    <Component>
        <span class="bar"></span>
    </Component>
</div>
```

index.mcss
```less
/* scoped */
.main {
    color: green;
    .bar {
        font-size: 14px;
    }
}
```

## 处理

### 1. 计算hash

对标记了`@scoped`的mcss文件, 取文件在工程中的相对路径, 对其计算hash, 取前8位字符作为选择器中使用的hash, 并记录到hash表中

hash表数据结构:

|path |hash |
|:----|:----|
|path1|hash1|
|path2|hash2|
|path3|hash3|

### 2. 处理样式文件
对计算了hash的文件中所有的选择器做扁平化处理, 对每个选择器添加后缀`-hash`

### 3. 处理模板文件
- 读取模板文件顶部声明的scoped, 找到对应样式文件的hash值
- 对文件中所有的结构匹配到样式文件中选择器的类名添加hash后缀

## 输出

combined.css

```css
.component-componentHash {
    color: red;
}
.component-componentHash  .bar-componentHash {
    font-size: 12px;
}
.main-mainHash {
    color: green;
}
.main-mainHash .bar-mainHash {
    font-size: 14px;
}
```

index.html
```html
<div class="main-mainHash">
    <div class="component-componentHash">
        <span class="bar-componentHash">
            <span class="bar-mainHash"></span>
        </span>
    </div>
</div>
```

## 总结

### 此方案存在的缺陷及说明

- 加了hash之后, css / html 文件的体积会变大(经实验, 100个hash标志, 占用空间不到1k)
- 对wap工程中的mcss文件以`.`进行搜索, 一个点号代表一个类, 查看了从文件名开头a到n的mcss文件, 大部分文件中点号数量在100以下, 其中大部分在50以下
- 经过观察, 组件模板中`class`关键字一般出现次数不超过20次
- 极端情况: 假设每个页面入口模板中包含100个class且不重复, 引入10个组件, 每个组件包含50个class记, 共600个chass, 算上css文件, 共1200个class, 页面中共产生1200个hash, 共消耗流量约12k, 笔者认为对页面加载没有很大影响

### 此方案带来的好处

1. 开发者可以在scoped样式文件中命名类时获得更大的自由度和灵活性
2. 可以不必排查其他样式文件以避免重名, 只要保证在scoped文件作用域中类名唯一即可
3. scoped样式文件不会对其他样式文件产生污染
4. 模板文件中看到@scope注释, 可以一目了然的知道当前模板样式受哪个样式文件控制

### 实现难度

|改造的库/模块|描述|难度|
|:----|:----|:----|
|hash表|生成文件路径与其hash的对应关系, 保存到内存/临时文件中|☆☆|
|mcss |给scoped style文件中的选择器加后缀|☆☆☆|
|regular  |解析html, 给dom元素的class添加hash|☆☆☆|

(现已基本实现以上功能, 封存待用.)

## 参考资源:
[vue-loader Scoped CSS](https://vue-loader.vuejs.org/en/features/scoped-css.html)

[Can I use camel-case in CSS class names](https://stackoverflow.com/questions/1547986/can-i-use-camel-case-in-css-class-names)

by [Jerry Yu](https://github.com/yubaoquan)