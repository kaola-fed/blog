---
title: SVG动画实践篇-无中生有的线条动画
date: 2017-04-07
---


## 说明

这个动画实现的是线条动画,主要用到的是 SVG 的 path 标签。

<!-- more -->


### &lt;path&gt; 标签命令

使用 &lt;path&gt; 标签的 d 属性标识路径集合,勾画线条的形状。

- M = moveto
- L = lineto
- H = horizontal lineto
- V = vertical lineto
- C = curveto
- S = smooth curveto
- Q = quadratic Belzier curve
- T = smooth quadratic Belzier curveto
- A = elliptical Arc
- Z = closepath

例如:

```
<svg width="300" height="300" version="1.2" xml:space="default">
    <path d="M0 0 L150 100 V200 H100" stroke="#f00" stroke-width="1"/>   
</svg>
```



## 动画实现

### 步骤一: 定义SVG线条

定义SVG线条,除了使用 d 属性定义路径外,还需要用到两个重要的属性, stroke-dasharray 和 stroke-dashoffset, 这两个属性值可以在 path 标签上定义,也可以在样式表中定义。

- stroke-dasharray 定义短划线和缺口的长度,实现画虚线的效果。例如4px 2px/4px,2px,数与数之间可用空白或逗号隔开。
- stroke-dashoffset 标识的是整个路径的偏移值。

svg代码如下:
```
<svg width="500" height="200" version="1.2" xml:space="default">
    <path id="path" d="M0,150c0,0,0-61,72-44c0,0-47,117,81,57s5-110,10-67s-51,77.979-50,33.989" stroke="#f00" stroke-width="1" stroke-dasharray="4px,2px" stroke-dashoffset="10px" fill="none"/>
</svg>
```


### 步骤二: 给path标签使用CSS3动画

定义 css3 的 animation,通过改变 path 标签的 stroke-dasharray 或 stroke-dashoffset 值来使路径动起来。
path 路径的长度可使用 js 的 document.getElementById(‘path’).getTotalLength() 来获得。

#### 方法一: 改变 stroke-dasharray 来实现动画

css 代码如下:
```
#path{
    -webkit-animation:slide 2s linear infinite;
}


@keyframes slide {
    0%{
        stroke-dasharray:0 511px;   /* 511px 为整个路径的长度 */
    }
    100%{
        stroke-dasharray:511px 511px;
    }
}
```

- stroke-dasharray:0 511px; 实线宽度为0,空隙宽度为整个path路径的宽度,所以刚开始路径没有实线,是不可见的。
- stroke-dasharray:511px 511px; 实线宽度为整个 path 路径长度,所以整条路径可见。
- css3 animation 动画定义路径从不可见到可见的变化。
 
 
#### 方法二: 改变 stroke-dashoffset 来实现动画
 
css 代码如下:
```
#path{
    stroke-dasharray:511px 511px;
    -webkit-animation:slide2 2s linear infinite;
}

@keyframes slide2 {
    0%{
        stroke-dashoffset:511px;
    }
    100%{
        stroke-dashoffset:0px;
    }
}
```       

- stroke-dasharray:511px 511px; 给 path 标签定义实线宽度和空隙宽度都为整个path 的长度。这个时候如果不用动画,则线条会全部展示。
- 0%{stroke-dashoffset:511px;}  path 路径左偏移 511px, 则会显示 511px 的空隙宽度。此时路径没有实线,是不可见的。
- 100%{stroke-dashoffset:0px;} path 路径偏移量为0,则恢复到最初始状态,显示全部的实线。
- css3 animation 动画定义路径从不可见到可见的变化。

#### 多条 path 的动画或文字动画

- 使用 symbol 定义和 use 实例化来画出SVG路径。
- 使用 CSS3 的 animation 属性来修改实例化路径的 stroke-dasharray 或 stroke-dashoffset 的值,从而实现动画效果。
- 可新建多个同样的 SVG 路径,并且每个路径的颜色和动画效果都不一样,最终形成错落的完整的动画。

git: https://github.com/rainnaZR/svg-animations/tree/master/src/pages/step2/path
参考资料: http://www.alloyteam.com/2017/02/the-beauty-of-the-lines-break-lines-svg-animation/