---
title: SVG动画基础篇
date: 2017-03-30
---

参考资料: 
http://www.w3school.com.cn/svg/index.asp

https://msdn.microsoft.com/zh-cn/library/gg193979

动画效果参考：https://github.com/rainnaZR/svg-animations

<!-- more -->

## 简介
SVG 指可伸缩矢量图形。是使用 XML 来描述二维图形和绘图程序的语言。

SVG 代码的根元素是以 &lt;svg&gt; 元素开始，&lt;/svg&gt;结束。width 和 height 属性可设置 SVG 文档的宽度和高度。version 属性可定义所使用的 SVG 版本，xmlns 属性可定义 SVG 命名空间。


## 使用说明
### 如何画圆
```
<svg width="300px" height="300px" version="1.1" xmlns="http://www.w3.org/2000/svg">
    <circle cx="100" cy="100" r="40" stroke="black" stroke-width="2" fill="#f60"/>
</svg>
```

- SVG 的 &lt;circle&gt; 用来创建一个圆。
- cx 和 cy 属性定义圆中心的 x 和 y 坐标。如果忽略这两个属性，那么圆点会被设置为 (0, 0)。
- r 属性定义圆的半径。
- stroke 和 stroke-width 属性控制如何显示形状的轮廓,也就是边框。stroke定义边框的颜色,stroke-width定义边框的宽度。
- fill 属性设置形状内的颜色,也就是背景色(填充色)。


### 如何画椭圆形
```
<svg width="300" height="300" version="1.2" xml:space="default">
    <ellipse cx="100" cy="100" rx="50" ry="80" style="fill:#f60;stroke:#000;stroke-width:5;"/>
</svg>
```

- SVG的&lt;ellipse&gt;用于创建一个椭圆形。
- cx 属性定义圆心的 x 坐标。
- cy 属性定义圆心的 y 坐标。
- rx 属性定义水平半径。
- ry 属性定义垂直半径。


### 如何画矩形
```
<svg width="100%" height="100%" version="1.1" xmlns="http://www.w3.org/2000/svg">
    <rect x="50" y="50" rx="20" ry="20" width="300" height="100" style="fill:rgb(0,0,255);fill-opacity:.5;stroke-width:2;stroke:#f60;stroke-opacity:.5;opacity:0.6" />
</svg>
```

- SVG 的 &lt;rect&gt; 用来创建一个矩形。
- x 属性定义矩形的左侧位置(矩形到浏览器窗口左侧的距离)。
- y 属性定义矩形的顶端位置(矩形到浏览器窗口顶端的距离)。
- rx 和 ry 属性可使矩形产生圆角。
- rect 元素的 width 和 height 属性可定义矩形的高度和宽度。
- style 属性用来定义 CSS 属性。
- CSS 的 fill 属性定义矩形的填充颜色(背景色)。
- CSS 的 fill-opacity 属性定义填充颜色透明度。
- CSS 的 stroke 代表边框的颜色。
- CSS 的 stroke-width 代表边框的宽度。
- CSS 的 stroke-opacity 代表边框颜色的透明度。
- CSS 的 opacity 定义整个元素的透明度。


### 如何画线条
```
<svg width="300" height="300" version="1.2" xml:space="default">
    <line x1="50" y1="50" x2="200" y2="200" style="stroke:#f00;stroke-width:10"/>
</svg>
```

- SVG的&lt;line&gt;标签用于创建线条。
- x1 属性表示线条起始的 x 坐标。
- y1 属性表示线条起始的 y 坐标。
- x2 属性表示线条结束的 x 坐标。
- y2 属性表示线条结束的 y 坐标。
- CSS 的 stroke 表示边框的颜色,这里也就是指线条的颜色。
- CSS 的 stroke-width 表示线条的宽度。


### 如何画多边形
```
<svg width="300" height="300" version="1.2" xml:space="default">
    <polygon points="50,50 250,50 150,150" style="fill:#f60;stroke-width:5;stroke:#000;"/>
</svg>
```

- SVG 的&lt;polygon&gt;用来创建含有不少于三个边的图形。
- points 定义多边形每个角的 x 和 y 坐标。


### 如何画折线
```
<svg width="300" height="300">
    <polyline points="0,0 50,0 50,50 100,50 100,100 150,100 150,150" style="fill:#f60;stroke:#000;stroke-width:5"/>
</svg>
```

- SVG 的 &lt;polyline&gt; 标签用来画折线。
- points 属性定义每个角的 x 和 y 坐标。


### 如何画复杂路径
```
<svg width="300" height="300" version="1.2" xml:space="default">
    <path d="M0 0 L150 100 V200 H100 Z"/>   
</svg>
```

- SVG 的 &lt;path&gt; 用来定义路径。
- M = moveto,L = lineto,H = horizontal lineto,V = vertical lineto,C = curveto,S = smooth curveto,Q = quadratic Belzier curve,T = smooth quadratic Belzier curveto,A = elliptical Arc,Z = closepath,以上所有命令均允许小写字母。大写表示绝对定位，小写表示相对定位。

  
### 如何给元素定义滤镜
```
<svg width="500" height="500" version="1.1" xmlns="http://www.w3.org/2000/svg">
    <defs>
        <filter id="Gaussian_Blur">
            <feGaussianBlur in="SourceGraphic" stdDeviation="3" />
        </filter>
    </defs>

    <circle cx="100" cy="100" r="40" stroke="black" stroke-width="2" fill="#f60" style="filter:url(#Gaussian_Blur);"/>
</svg>
```

- 使用 &lt;filter&gt; 标签用来定义 SVG 滤镜, &lt;filter&gt; 标签必须嵌套在 &lt;defs&gt; 内。
- &lt;filter&gt; 上的 id 属性为滤镜定义一个唯一的名称。（同一滤镜可被文档中的多个元素使用）
- &lt;feGaussianBlur&gt; 标签定义滤镜效果。
- &lt;feGaussianBlur&gt; 标签的 in="SourceGraphic" 定义了由整个图像创建效果。
- &lt;feGaussianBlur&gt; 标签的 stdDeviation 属性定义模糊的程度。
- 在元素的样式上添加 filter:url('#滤镜ID名') 来使用滤镜。

SVG 中,可使用的滤镜如下:feBlend, feColorMatrix, feComponentTransfer, feComposite, feConvolveMatrix, feDiffuseLighting, feDisplacementMap, feFlood, feGaussianBlur, feImage, feMerge, feMorphology, feOffset, feSpecularLighting, feTile, feTurbulence, feDistantLight, fePointLight, feSpotLight

SVG滤镜 | 说明
---|---
feColorMatrix | 应用matrix转换
feComponentTransfer | 执行数据的 component-wise 重映射(没懂)
feOffset | 相对当前图像的移动位置
feMerge | 创建累积而上的图像


### 如何给元素添加渐变效果
```
<svg width="800" height="800" version="1.2" xml:space="default">
    <defs>
        <linearGradient id="orange_white" x1="0%" y1="0%" x2="0%" y2="100%">
            <stop offset="0%" style="stop-color:#f60;stop-opacity:1;"/>
            <stop offset="100%" style="stop-color:#fff;stop-opacity:1;"></stop>
        </linearGradient>
        
        <radialGradient id="orange_blue" cx="50%" cy="50%" r="50%" fx="50%" fy="50%">
            <stop offset="0%" style="stop-color:#f00;stop-opacity:1;"/>
            <stop offset="100%" style="stop-color:blue;stop-opacity:1;"/>
        </radialGradient>
    </defs>

    <rect height="200 " width="200" style="fill:url(#orange_white);stroke:#000;stroke-width:5;" />
    <circle r="50" cx="300" cy="300" style="fill:url(#orange_blue)" />
```

#### 线性渐变
- 使用 &lt;linearGradient&gt; 可用来定义 SVG 的线性渐变。&lt;linearGradient&gt; 内嵌在 &lt;defs&gt; 标签中。
- &lt;linearGradient&gt; 标签的 id 属性定义渐变的唯一名称。
- &lt;linearGradient&gt; 标签的 x1、x2、y1、y2 属性可定义渐变的开始和结束位置。如果x1 == x2,y1 != y2,则为垂直渐变。如果 x1 != x2,y1 == y2,则为水平渐变。如果x1 != x2,y1 != y2,则为角形渐变。
- 通过 &lt;stop&gt; 标签来规定每种颜色的渐变属性。渐变的颜色可以有多种。
- &lt;stop&gt; 标签的 offset 属性定义渐变的开始和结束位置。
- &lt;stop&gt; 标签的 style 属性里可定义stop-color,stop-opacity等属性。
- 元素通过 fill:url(#渐变ID) 来使用渐变效果。

#### 放射性渐变
- 使用 &lt;radialGradient&gt; 可用来定义 SVG 的放射性渐变。
- &lt;radialGradient&gt; 标签的 id 属性定义渐变的唯一名称。
- &lt;radialGradient&gt; 标签的 cx、cy 和 r 属性定义外圈，而 fx 和 fy 定义内圈。
- 通过 &lt;stop&gt; 标签来规定每种颜色的渐变属性。渐变的颜色可以有多种。


### 给元素加超链接
```
<a xlink:href="http://www.jd.com" target="_blank">
    <rect x="20" y="20" width="250" height="250" style="fill:blue;stroke:pink;stroke-width:5;opacity:0.9"/>
</a>
```


### 如何给元素添加动画
```
<svg width="800" height="800">
    <circle r="100" cx="200" cy="400" fill="#f60">
        <animate attributeName="opacity" attributeType="CSS" from="1" to="0" dur="5s" repeatCount="indefinite"/>
        <animate attributeName="r" attributeType="XML" begin="0s" dur="5s" from="100" to="150" repeatCount="indefinite"/>
        <animateMotion path="M 0 0 L 100 100" dur="5s" fill="freeze" repeatCount="indefinite"/>
    </circle>

    <rect x="400" y="400" width="200" height="200" style="fill:#f00;">
        <animateTransform attributeName="transform" attributeType="XML" type="rotate" from="-30" to="0" begin="1s" dur="5s" fill="freeze"/>
        <animateTransform attributeName="transform" attributeType="XML" type="scale" from="1" to="0.5" additive="sum" begin="1s" dur="5s" fill="freeze"/>
    </rect>
</svg>
```

- &lt;animate&gt; 标签给元素添加动画。
- &lt;animateMotion&gt;	标签使元素沿着动作路径移动。
- &lt;animateTransform&gt;	标签对元素进行动态的属性转换。