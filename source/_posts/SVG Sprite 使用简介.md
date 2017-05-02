---
title: SVG Sprite 使用简介
date: 2017-04-25
---

# SVG简介

SVG即可缩放矢量图形 (Scalable Vector Graphics)的简称, 是一种用来描述二维矢量图形的XML标记语言. SVG图形不依赖于分辨率, 因此图形不会因为放大而显示出明显的锯齿边缘.

<!-- more -->

# icon sprite

当我们需要使用多个icon的时候, 为了节省请求和方便管理, 通常会把icon合并到一个文件中, 在使用时再通过一定的方法从icon集合文件中取出所需的图形并显示. 目前使用得最多的应该就是我们所熟悉的CSS Sprite和Icon Font.

## CSS Sprite

CSS Sprite的原理是将多个icon按一定规律整理到一个图片文件中, 使用时利用`background-image`和`background-position`将图片中特定部分显示出来. CSS Sprite技术已经被广泛应用了很长的一段时间, 目前有许多自动化生成Sprite图片和CSS文件的工具, 例如(gulp.spritesmith)[https://github.com/twolfson/gulp.spritesmith].

```css
.icon1 {
    background-image: url(/res/icon1.png)
}
.icon1-increase {
    background-position: -10px -10px;
}
```

```html
<i class="icon1 icon1-increase"/>
```

CSS Sprite技术成熟, 兼容性好, 但是缺点也比较明显. 如在实际需求中, 对应形状相同但颜色不同的icon, 就需要为不同颜色的icon各保存一份; 有时候需要对已有icon放大显示时, 发现锯齿严重, 那么又要再保存一份放大版的icon. 因此, Sprite文件会随着时间越变越大, 同时内容越来越乱, 逐渐变得难以管理.

## Icon Font

Icon Font的基本原理是将Icon定义为图片字体, 在CSS中用`@font-face`引入Icon Font自定义字体, 再利用`font-family`和字符码显示出指定的图标.

```css
@font-face {
    font-family: 'iconfont';
    src: url(/res/icon2.ttf) format('truetype');
}
.icon2 {
    font-family: 'iconfont';
}
```

```html
<i class="icon2">&#33</i>
```

由于使用的是字体, 因此可以通过`color`, `font-size`设置icon的样式. Icon Font拥有比CSS Sprite图片更小的文件体积, 维护也比图片更方便, 但是icon font通常只能使用单一的颜色, 字体文件生成也比CSS Sprite更复杂.

# SVG Sprite

通常在使用SVG的时候, 我们是直接写到`svg`标签当中:

```html
    <svg xmlns="http://www.w3.org/2000/svg" width="150" height="100" viewBox="0 0 3 2">
        <rect width="1" height="2" x="0" fill="#008d46" />
        <rect width="1" height="2" x="1" fill="#ffffff" />
        <rect width="1" height="2" x="2" fill="#d2232c" />
    </svg>
```

此时SVG图形会直接在页面当中显示. SVG属性中, 可以利用(`symbol`)[https://developer.mozilla.org/zh-CN/docs/Web/SVG/Element/symbol]封装图形, 并利用(`use`)[https://developer.mozilla.org/zh-CN/docs/Web/SVG/Element/use]将其实例化, 从而实现SVG Sprite的功能.

SVG Sprite实例:

```html
<svg style="height:0;width:0;display:none;" version="1.1" xmlns="http://www.w3.org/2000/svg">
    <symbol id="icon-italy" width="150" height="100" viewBox="0 0 3 2">
        <rect width="1" height="2" x="0" fill="#008d46" />
        <rect width="1" height="2" x="1" fill="#ffffff" />
        <rect width="1" height="2" x="2" fill="#d2232c" />
    </symbol>

    <symbol id="icon-france" width="150" height="100" viewBox="0 0 3 2">
        <rect width="1" height="2" x="0" fill="#002496" />
        <rect width="1" height="2" x="1" fill="#ffffff" />
        <rect width="1" height="2" x="2" fill="#ee2839" />
    </symbol>
</svg>

<svg><use xlink:href="#icon-italy"/></svg>
<svg><use xlink:href="#icon-france"/></svg>
```

这样就实现了SVG Sprite.

## 原理

通过devtool可以观察到, `use`标签是利用shadow dom实现的.

![use.shadowdom](http://og40ypzfa.bkt.clouddn.com/use_shadowdom.png)

它通过`xlink:href`这个XML的`attribute`引用指定的SVG `symbol`, 在渲染时指定`symbol`标签中的内容就会被渲染显示在页面当中. 这意味着, 如果无法直接对`use`标签中的shadow dom进行访问和修改. 例如像`use#rect`这样的选择器是无法生效的, 因此不能通过一般的css选择器对`use`中图形不同的部分进行控制.

## CSS样式

更多的时候, 我们通常都只需要改变图标的大小和颜色, 在SVG Sprite当中, 实现起来并不复杂.

1. 大小

通过改变`svg`容器的大小, 可以轻松地对图标大小进行控制.

```css
.icon{
    width: 120px;
    height: 80px;
}
.icon.icon-small{
    width: 60px;
    height: 40px;
}
```

```html
<svg class="icon"><use xlink:href="#icon-italy"/></svg>
<svg class="icon icon-small"><use xlink:href="#icon-italy"/></svg>
```

![svg.icon.size](http://og40ypzfa.bkt.clouddn.com/svg.icon.size.png)

2. 颜色

- 单色

颜色由于不能直接对`use`的shadow root中的图形标签进行选择, 因此在为图标定义颜色时需要利用`fill: inherit`这个一属性.

```css
svg path{
    fill: inherit;
}
.icon2-green{
    fill: #008d46;
}
.icon2-red{
    fill: #dc352f;
}
```

```html
<svg class="icon2 icon2-green">
    <use xlink:href="#icon-increase"/>
</svg>
<svg class="icon2 icon2-red">
    <use xlink:href="#icon-increase"/>
</svg>

```

![svg.icon.color](http://og40ypzfa.bkt.clouddn.com/svg.icon.color.png)

利用`inherit`, 还可以定义`stroke-width`, `stroke`等属性.

- 多色

除了对图标整体颜色进行定义以外, 还可以根据需要对图形不同部分进行颜色定义. 这里使用到了css 的自定义属性([CSS Custom Properties](https://www.w3.org/TR/css-variables/)).

这里需要对`icon.svg`进行修改, 将图形个部分的颜色设置为`fill: var(--*[, default])`的形式.

```html
<symbol id="icon-flag" width="150" height="100" viewBox="0 0 3 2">
    <rect width="1" height="2" x="0" style="fill: var(--color0, #008d46)" />
    <rect width="1" height="2" x="1" style="fill: var(--color1, #fff)"/>
    <rect width="1" height="2" x="2" style="fill: var(--color2, #d2232c)"/>
</symbol>
```

定义css样式

```css
.flag-belgium {
    --color0: #201b18;
    --color1: #f1ee3d;
    --color2: #dc352f;
}
```

```html
<svg class="icon">
    <use xlink:href="#icon-flag"/>
</svg>
<svg class="icon flag-belgium">
    <use xlink:href="#icon-flag"/>
</svg>
```

![svg.icon.color.parts](http://og40ypzfa.bkt.clouddn.com/svg.icon.color.parts.png)

除此以外, 还有其他一些样式定义的形式, 更详细的内容可以阅读[Styling SVG <use> Content with CSS](http://tympanus.net/codrops/2015/07/16/styling-svg-use-content-css/).

# 实际使用

## 引用外部svg

上文介绍的是使用内联svg, 但其实还可以使用外部svg的.

```html
<svg class="icon">
    <use xlink:href="/res/svg/icon.svg#icon-flag"/>
</svg>
```

然而这种方法不兼容IE9~10, 但如果非要使用外部svg的形式, 可以引入一个pollyfill[svg4everybody](https://github.com/jonathantneal/svg4everybody), 当运行在不支持外部svg的情况下, 这个pollyfill会通过异步请求加载svg委文件并将其注入到页面当中, 将引用方法转换成内联svg.

## 在ftl中使用

可以在`svg.ftl`中定义一系列`macro`, 用以加载`.svg`文件.

```html
<#macro svgicon path>
    <#include "${path}">
</#macro>

<#macro svgicon3 >
    <@svgicon "./icon3.svg"/>
</#macro>
```

在页面中引用`demo.ftl`:

```html
<@svgicon1/>
<svg>
    <use xlink:href="#icon-italy"/>
</svg>
```

## 在regular中使用

`xlink:href`是使用了命名空间的XML特性, 如果是写在`.html`页面的标签, 该特性能够正常被浏览器解析并完成svg渲染. 如果该svg变量是通过DOM API创建出来的话, 则需要使用特定的方法进行处理([SVG with USE tag not rendering](http://stackoverflow.com/questions/25975523/svg-with-use-tag-not-rendering)).

即需要利用`createElementNS`和`setAttributeNS`方法在创建的同时声明命名空间.

```javascript
var use = document.createElementNS('http://www.w3.org/2000/svg', 'use');
use.setAttributeNS('http://www.w3.org/1999/xlink', 'xlink:href', '#icon-increase');
document.querySelector('#svgid').appendChild(use);
```

只要是需要动态创建`use`元素, 都需要使用上面这种方法才能使SVG Sprite中的元素实例化. 但在目前的Regular(0.4.3)中, 不会对命名空间进行处理, 这意味着, 如果直接将`<svg><use xlink:href/></svg>`写在Regular组件的模版中时, 该标签是无法正常渲染的. 为此可以增加一个指令`r-xlink:href`以完成手动设置命名空间的操作.

```javascript
Regular.directive('r-xlink:href', function (elem, val) {
    if (val&& val.type === 'expression') {
        this.$watch(val, function (newVal) {
            elem.setAttributeNS('http://www.w3.org/1999/xlink', 'href', newVal);
        });
    } else {
        elem.setAttributeNS('http://www.w3.org/1999/xlink', 'href', val);
    }
});
```

那么在组件模版中就可以像在普通`.html`中那样使用SVG Sprite了. 当然, 首先得确保SVG Sprite被写到页面中.

```html
<svg>
    <use r-xlink:href="#icon-italy"/>
</svg>
```

在Regular的SVG 实践中得到了郑海波大神的指点, 否则得绕更大的路才能把问题解决, 在此表示感谢.

# 小结

对比前文提到的CSS Sprite和Icon Font, SVG有着明显的优势:

- 放大缩小不会失真
- 大小, 颜色等属性自定义灵活
- 体积小, 同时管理方便

虽然SVG Sprite有着高度的灵活性, 但于此同时, SVG兼容性有待考究, 同时其渲染性能也不及图片和字体那么高, 可能在某些情况下不适用. 不过在一般的场景中, svg sprite还能够给开发带来很大的便利的.

# 参考

1. [SVG元素参考](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Element)
2. [SVG with USE tag not rendering](http://stackoverflow.com/questions/25975523/svg-with-use-tag-not-rendering)
3. [未来必热：SVG Sprite技术介绍](http://www.zhangxinxu.com/wordpress/2014/07/introduce-svg-sprite-technology/)
4. [Styling SVG <use> Content with CSS](http://tympanus.net/codrops/2015/07/16/styling-svg-use-content-css/)
5. [Icon System with SVG Sprites](https://css-tricks.com/svg-sprites-use-better-icon-fonts/)

