---
title: SVG动画实践篇-音量变化效果
date: 2017-04-04
---

## 说明

这个动画的效果就是多个线条的高度发生变化,使用了两种写法(css,svg)来实现。

<!-- more -->

#### CSS实现

- 定义线条的节点,可以使用伪元素实现。
- 使用 CSS3 的 animation 属性给元素定义动画样式。
- 每个元素定义的动画的延时时间不固定。

```
@-webkit-keyframes slide{
    0%{height:0;}
    100%{height:50px;}
}
```

```
.m-box .line:nth-child(1){
    -webkit-animation:slide 1.2s linear .5s infinite alternate;
}
.m-box .line:nth-child(3){
    -webkit-animation:slide 1.2s linear .75s infinite alternate;
}
```


#### SVG实现

使用animate元素来实现。原理一样,通过改变元素的高度。

- x="20",通过改变 x 坐标的值来给动画元素定位。(这里指的橙色线条)
- 修改 animate 标签上的 begin 属性值来定义元素动画的延时时间。
- svg 动画无法像 CSS 动画一样,定义轮流反向播放动画的效果。所以动画有些生硬。


```
<svg width="300" height="300" version="1.2" xml:space="default">
    <rect height="0" width="5" rx="2.5" style="fill:#f60;">
        <animate attributeName="height" attributeType="XML" from="0" to="50" begin="0.5s" dur="1.2s" calcMode="linear" repeatCount="indefinite" />
    </rect>
    <rect height="0" width="5" rx="2.5" x="10" style="fill:#f60;">
        <animate attributeName="height" attributeType="XML" from="0" to="50" begin="0s" dur="1.2s" calcMode="linear" repeatCount="indefinite" />
    </rect>
    <rect height="0" width="5" rx="2.5" x="20" style="fill:#f60;">
        <animate attributeName="height" attributeType="XML" from="0" to="50" begin="0.75s" dur="1.2s" calcMode="linear" repeatCount="indefinite" />
    </rect>
    <rect height="0" width="5" rx="2.5" x="30" style="fill:#f60;">
        <animate attributeName="height" attributeType="XML" from="0" to="50" begin="0.25s" dur="1.2s" calcMode="linear" repeatCount="indefinite" />
    </rect>
    <rect height="0" width="5" rx="2.5" x="40" style="fill:#f60;">
        <animate attributeName="height" attributeType="XML" from="0" to="50" begin="0.5s" dur="1.2s" calcMode="linear" repeatCount="indefinite" />
    </rect>
</svg>
```


#### 结论

- svg 动画无须定义样式,完全通过定义标签的属性来定义动画。
- svg 动画不能定义轮流反向播放动画的效果。

git 地址：https://github.com/rainnaZR/svg-animations/tree/master/src/pages/step2/volumn