---
title: 视口（viewport）简介 — Web移动手册阅读笔记
date: 2017-09-30
---
做过移动端的同学对下面这2段代码肯定是不陌生的：
```html
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">
```
```css
@media screen and (max-width: 480px) {
    // 书写宽度不超过480px时的样式
}
```
也就是**meta视口标签**和**宽度媒体查询**，下面我们将深入讲解这里面的细节实现。

题外话：
> meta视口是Apple公司创造的，别的移动端浏览器都直接复制了它

## 像素

像素是网页布局的基础，最开始web开发者基本靠直觉使用它；那么到底什么是一个像素？

简单的说法：一个像素就是计算机屏幕显示一种特定颜色的最小区域；设备宽度相同，像素越高，画面就更细致更清晰

![img1](https://www.evernote.com/l/Aaw36HN1P1VG5K_sz1q1U7KTwReNqT3UBncB/image.png)

我们平时说的像素，其实有2种，**设备像素和CSS像素**

设备像素：
> 设备的物理像素，任何设备的物理像素都是固定的，比如6S 1334x750，plus 1920x1080 X：2436 * 1125

CSS像素：
> 为Web开发者创造的，在css和js中使用的抽象层

举例：
当你给一个元素设置200px时，那么它到底会跨越多少个**设备像素**呢，这个取决于屏幕特性和缩放；比如2倍视网膜屏就是跨越了400像素，如果用户放大，那么跨越的**设备像素**就越多；每一个css和js测试返回的元素宽度仍然为200px；开发者就不用关心这个元素到底跨越了多少个**设备像素**，而是把复杂的计算交给了浏览器，这就是为什么说**CSS像素**是专门为Web开发者创造的一个抽象层。

## 三个视口
### 视口基础

给一个div元素设置width为35%的时候, 是相对于什么的35%？
是相对于父级，而CSS中没有设置宽度的块级元素默认宽度是100%，根元素html标签没有设置宽度，所以默认为100%，**而在桌面浏览器中，视口的宽度就等于浏览器窗口的宽度**，所以这个div会占用视口宽度的35%，如图所示：
![img](https://www.evernote.com/l/AaxbEiD4WnlKUIcXQLyk2agbfz6hYiuoS4IB/image.png)

### 布局视口

一般pc端的网站宽度是800px到1024px，如果移动端的视口表现和pc的表现一致的话，可以想象网站的内容将会变得非常的小，所以为了避免这种情况：
> 在手机上，视口与移动端浏览器的宽度不再相关联，而是完全独立的了。我们称它为布局视口---css布局会根据它来计算，并被它约束

如图：
![img](https://www.evernote.com/l/Aay93nPSACRLcp-yWM-tlvCd9i86uZW4gj0B/image.png)
### 视觉视口

> 它是用户正在看到的网站的区域。用户可以通过缩放来操作视觉视口，同时不会影响布局视口，布局视口仍保持原来的宽度。

![img](https://www.evernote.com/l/Aaw1hebd1vhKxKIe6ow8fNPlpOObv0nbFAQB/image.png)
### 理想视口

布局视口的默认宽度并不是一个理想的宽度，大家从上面的图就可以看出来了，所以苹果公司就引进了**理想窗口**这个概念：
> 它是对设备来说最理想的布局视口尺寸。显示在理想视口中的王者拥有最理想的浏览和阅读的宽度，用户刚进入页面时不再需要缩放。

```html
<meta name="viewport" content="width=device-width">
```
上面这行代码就是告诉浏览器，布局视口的宽度应该与理想视口的宽度一致。

我们平时在chrome控制台里看到的尺寸就是理想尺寸：
![img](https://www.evernote.com/l/Aawvv1d7141CCZ6RP5KKLRsO9oTNAHhc1ZoB/image.png)

注意：
- 理想窗口的尺寸是由浏览器厂商决定的，同一设备可以有不同的尺寸
- 不同设备的相同浏览器理想窗口也会不同，比如手机和平板
- 而且会随着设备转向改变

虽然有那么多不同尺寸的理想视口，但是平时开发我们只要告诉浏览器使用它的理想视口（也就是上面那行代码），就没问题了。

### 总结

- 桌面中浏览器窗口就是约束你的css布局窗口，它也决定用户可以看到什么
- 在手机上，桌面窗口被拆分成了两个：布局视口会限制你的CSS布局；视觉视口决定用户能看到什么。
- 移动端浏览器还有一个理想视口，它是对于特定设备上的特定浏览器的布局视口的一个理想尺寸
- 可以把布局视口的尺寸设置为理想视口。实际上，这就是响应式设计的基础。

## 缩放

手机上放大，视觉视口缩小，布局视口不变，所以我们看到的css布局是不变的。
桌面上放大，视觉视口缩小，由于桌面的布局视口和视觉视口是相同的，所以布局视口也变小，这就是我们在桌面端缩放的时候样式有时候会错乱的原因。

据说移动端css布局不改变也是因为移动端进行重新布局的成本太高

### 禁止缩放

meta标签里的 user-scalable=no 值就表示禁止缩放。
作者建议不要禁用，因为用户有时候有需要，争议大这里暂时不讨论。

## 分辨率

它有2个概念，一个是设备的分辨率，一个是css/js里的分辨率，这2个东西不是并不是一回事。

### 物理分辨率

DPI：像素数量/宽度（以英寸为单位）得到设备每英寸的点数
无论如何js都无法获取设备的物理分辨率, `screen.width`并不可靠

### 设备像素比

设备像素比：Device Pixel Ratio（简称DPR），是设备像素和理想视口的比值。
比如，iphone早起设备像素是320，理想视口宽度也是320，所以DPR就是1。后来的视网膜屏，iphone设备像素变成了640，理想窗口不变320，所以DPR也就变成了2。

- js通过`window.devicePixelRatio`, css通过`device-pixel-ratio`(基于webkit)来获取。
- DPR也是浏览器厂商决定的，所以也有很多种，并不一定都为整数

#### ddpx
window.devicePixelRatio和媒体查询的device-pixel-ratio，单位是ddpx: 每一个像素的点数，但是实际上不允许使用单位的，所以下面是正确的写法：
```js
if(window.devicePixelRatio >=2){
    // DPR大于等于2时生效
}
```
```css
@media all and (-webkit-min-device-pixel-ratio: 2) {
    // DPR大于等于2时生效
}
```

ie11及以下不支持ddpx，所以必须要用dpi来表示。由于1英寸对应了CSS中的96个像素，所以1ddpx等于96dpi, 就有了如下写法：

```css
@media all and ((-webkit-min-device-pixel-ratio: 2),
                (min-resolution: 192dpi)) {
    // DPR大于等于2时生效
}
```

## meta视口标签

格式:
```html
<meta name="viewport" content="name=value, name=value">
```

content**名/值对**列表：
- width: device-width，宽度等于理想视口宽度
- initial-scale: 初始缩放程度，一般设置为1，并且加入meta标签，可以兼容不同浏览器
- minimum-scale/maximum-scale: 最小最大可缩放的程度，没啥好说的
- user-scalable: no; 用户不缩放

### 在css里写viewport设置
据说对微软的平板有用：
```css
@-ms-viewport {
    width: device-width;
}
```

## 媒体查询

媒体查询其实就是CSS的if语句

### 语法

```css
@media all and (max-width: 400) {
    div.container {
        // 宽度小于 400 的设备里才会生效这个样式
    }
}
```
- all代表所有设备，一般用这个就行了，print也可以用，别的设备不考虑了，支持不好。
- min-, max-是必要的前缀
- and等同于 &&， 逗号'**,**'等同于||

复杂例子：
```css
@media all and (max-width: 400) and (orientation: portrait) 
    and ((max-resolution:144dpi), (-webkit-max-device-pixel-ratio:1.5)) {
        // 只有宽度不超过400，竖屏模式，dpr小于等于1.5的时候才会生效
    }
```

## javascript相关

厂商间基本遵循的规范：
window.innerWidth：表示视觉窗口宽度，一般不会用；
document.documentElement.clientWidth： 表示布局窗口宽度，可进行类似媒体查询；
screen.width: 表示理想窗口宽度，兼容性据说差别很大；一般没啥用；

可以通过js模拟类似media查询的功能，在布局足够宽的时候才加载某些第三方组件：
```js
if(document.documentElement.clientWidth >=600) {
    // 加载facebook和twitter组件
}
```

- orientationchange事件，只要设备改变了方向都会触发，兼容性好；
- 移动端最好不要用resize事件，支持很差;

### 移动端, 桌面端对比表
![img](https://www.evernote.com/l/AaxhyIi1t_xLkIMUiOWc4w18lbelVDCy0CIB/image.png)
![img](https://www.evernote.com/l/AawFvhKSsdhPXaY1hzZtKrJmfoTvXm8vHhkB/image.png)

by <a href="https://github.com/jerryni" __blank>jerryni kaola</a>
<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
