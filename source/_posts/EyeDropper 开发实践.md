---
title: EyeDropper 开发实践
date: 2017-03-31
---

## 1. 什么是 EyeDropper

Chrome Devtools 的颜色提取器 [EyeDropper](https://developers.google.com/web/tools/chrome-devtools/inspect-styles/edit-styles)，用惯了 Chrome 的前端开发者并不陌生。

<!-- more -->

![image](http://7xidng.com1.z0.glb.clouddn.com/eyedropper.jpg)

但它并不支持在页面中使用，想在页面中使用只能自己实现一个。

那么接下来就介绍一下如何自己实现一个 EyeDropper。

## 2. 原理解读

要实现 EyeDropper，必须先学习一下基本的色彩知识。

物品被光线照射并反射出来，被人的眼睛接收，进而传递到人脑中形成对「色彩」的认知，称之为人的「视觉效应」。

### 色彩的三属性

#### 1. 色相（hue）

最最基本的颜色术语、通常用来表示物体的颜色。

当我们说红、绿、黄时，我们说的就是色相。将色相按照波谱顺序排列，首位相连形成环状则为「色相环」。虽然人们习惯将其分为七种颜色：红、橙、黄、绿、青、蓝、紫，但实际上的光谱应该是连续的。

![image](http://7xidng.com1.z0.glb.clouddn.com/hue.png)

#### 2. 饱和度（saturation）

指在特定的光照条件下颜色是如何呈现的，也就是色彩的鲜艳程度。

饱和度取决于颜色中含色成分和消色成分（灰色）的比例，即纯度最低的是灰色（无彩色）。高纯度表现为生机朝气，低纯度表现为厚重沉稳。

#### 3. 明度（value）

也被称作亮度，它是指颜色的明亮程度。

在任何颜色中添加白色，明度上升，添加黑色，明度下降。明度相差越远的两种颜色搭配，色彩之间的交界感就越明显，视觉上也就越清晰。

三者可以简单用下图综合表示：

![image](http://7xidng.com1.z0.glb.clouddn.com/colorful.jpg)

## 3. 方案设计

#### 1. 模块梳理

理解了基础的颜色原理后就好办事了，拿 Chrome Devtools EyeDropper 分析：

![image](http://7xidng.com1.z0.glb.clouddn.com/eytedropperDetail.png)

1. 饱和度和亮度选择器。
2. 色相选择器。
3. 透明度选择器。
4. 色彩转换器。点击可以在 RGBA、HSL 和 HEX 之间切换。
5. 调色板。点击直接选择不同的特定颜色。
6. 取色器。在屏幕上直接选择需要的颜色。

结合上述色彩知识，加上这块分析就可以开始进入代码层面的设计。

#### 2. 实际划分

在组件化大行其道的时代，以网易惯用的 Regular 进行组件化开发。

根据模块划分，划分基础的：

 - 饱和度和亮度选择组件；
 - 色相选择组件；
 - 透明度选择组件；
 - 色彩输入及转换器组件。

考虑简洁性，「取色器」及「调色板」不做实现。

## 4. 组件实现

#### 1. 色彩获取功能实现

 - 饱和度和亮度选择组件（就是 EyeDropper 最上面那块）

① 该组件以色相组件选择的色相(hue)为背景，若是想直接使用 hue 作为 CSS 背景，需要使用 hsl(hue, saturation, value) 格式，设置为：

```js
hsl(hue, 100%, 50%)
```

此时饱和度应设为 100%，因为饱和度为 0% 时为灰色，100% 时为原色。

![image](http://7xidng.com1.z0.glb.clouddn.com/saturation.png)

而亮度是指颜色偏向于白色还是黑色。50%的亮度值表示颜色位于黑色和白色中间，这时颜色会基本保持原来的颜色不变。

![image](http://7xidng.com1.z0.glb.clouddn.com/value.png)

② 同时利用线性渐变 `linear-gradient` 做色层叠加实现。

```css
.saturation-white {
	background: linear-gradient(to right, #fff, rgba(255,255,255,0));
}

.saturation-black {
	background: linear-gradient(to top, #000, rgba(0,0,0,0));
}
```

 - 色相组件（取颜色的那个条）

上面谈到了色相环的概念，色相组件利用的就是色相环原理。

将 EyeDropper 中的色相条与色相环对比，是不是有异曲同工之妙？答对了，直接将圆环拍平即可。

获取色值时只需要获取当前位置对于最左端的百分比，换算成圆环角度。

```js
Math.round(360 * percent / 100);
```

 - 色彩输入及转换器组件

其中总共有 RGBA、HSL 和 HEX 三种格式的切换。

在 Regualr 中，内嵌组件的传入属性会挂在子组件的 data 上，并实现数据绑定。但考虑数据处理的统一性，在所有颜色的获取处并不对颜色做处理，而是通过事件传递的方式，统一 $emit 到外层做统一的色值转换。

```js
this.$emit('change', {
    h: hue,
    s: saturation,
    v: bright,
    a: alpha
});

// 外层接收后统一处理
this.$on('change', processor);
```

接收到不同格式的颜色后，利用 [tinycolor2](https://www.npmjs.com/package/tinycolor2) 对颜色进行处理，使所有格式转换为同一种颜色。

```js
var color = tinycolor(colors);
var hsl = color.toHsl();
var hsv = color.toHsv();
```

#### 2. 需要注意的问题

 - 在使用鼠标拖动选择颜色时，需要做好节流处理，避免爆炸；
 - 由于 JS 对于浮点数的奇妙控制，需要在输入框做好统一截断处理；
 - 在鼠标拖动的时间绑定及解绑中，注意绑定对象与解绑对象的一致性。

代码可以 [戳我](https://github.com/Deol/regular-color) 查看，具体实现如下：

![image](http://7xidng.com1.z0.glb.clouddn.com/regular-color.jpg)

## 附录：

1. [EyeDropper 介绍](//developers.google.com/web/tools/chrome-devtools/inspect-styles/edit-styles)

2. [Totally Tooling Tips with Addy Osmani & Matt Gaunt](//www.youtube.com/playlist?list=PLNYkxOF6rcIB3ci6nwNyLYNU6RDOU3YyL)