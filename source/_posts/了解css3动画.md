---
title: 了解css3动画
date: 2018-01-03
---

## css3中的动画属性

- transition
- animation && keyframes

### transition

只能设置简单的动画，通过固定的值改变，并设置一个动画时间

```css
/* Apply to 1 property */
/* property name | duration */
transition: margin-left 4s;

/* property name | duration | delay */
transition: margin-left 4s 1s;

/* property name | duration | timing function | delay */
transition: margin-left 4s ease-in-out 1s;

/* Apply to 2 properties */
transition: margin-left 4s, color 1s;

/* Apply to all changed properties */
transition: all 0.5s ease-out;

/* Global values */
transition: inherit;
transition: initial;
transition: unset;
```

### animation && keyframes

可以定义复杂的动画

通常由2个组件组成:

- 一个描述动画方式的类animation: xxx-keyframes-name duration...
- 一个描述开始,结束状态的样式keyframes, 也可以是中途的样式

css3 animation很好的教程:
https://css-tricks.com/snippets/css/keyframe-animation-syntax/#article-header-id-0

#### 可以配合animation或者translate的`transform`属性，实现复杂的3d变幻效果

> 可以改变元素的坐标空间,如x,y,z的值, 可以放大缩小,旋转,倾斜

```css
/* Function values */
transform: matrix(1.0, 2.0, 3.0, 4.0, 5.0, 6.0);
transform: translate(12px, 50%);
transform: translateX(2em);
transform: translateY(3in);
transform: scale(2, 0.5);
transform: scaleX(2);
transform: scaleY(0.5);
transform: rotate(0.5turn);
transform: skew(30deg, 20deg);
transform: skewX(30deg);
transform: skewY(1.07rad);
transform: matrix3d(1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0, 11.0, 12.0, 13.0, 14.0, 15.0, 16.0);
transform: translate3d(12px, 50%, 3em);
transform: translateZ(2px);
transform: scale3d(2.5, 1.2, 0.3);
transform: scaleZ(0.3);
transform: rotate3d(1, 2.0, 3.0, 10deg);
transform: rotateX(10deg);
transform: rotateY(10deg);
transform: rotateZ(10deg);
transform: perspective(17px);

/* Multiple function values */
transform: translateX(10px) rotate(10deg) translateY(5px);

/* Global values */
transform: inherit;
transform: initial;
transform: unset;
```

### 暂停动画

```css
/* paused | running */
.paused {
    -webkit-animation-play-state: paused; 
    -moz-animation-play-state: paused;
    -o-animation-play-state: paused;
    animation-play-state: paused;
}
```

### 一个比较难理解的属性animation-fill-mode

4个值：
none: 执行完，回到原始样式
forwards: 执行完，停留在最后一帧的样式
backwards: 如果有`animation-delay`属性不为0，可以设置这段时间的图像为第一帧0%;
举例：animation: myAnim 2s both 1s;这个设置表示动画有1s的延迟（也就是animation-delay:1s），那么他这段等待时间内的样式就是动画的0%的样式；（动画等待的那段时间内，元素的样式将设置为动画第一帧的样式；）

both：forwards和backwards的效果都存在

参考：
https://www.w3cplus.com/css3/understanding-css-animation-fill-mode-property.html

## css3中的动画性能

浏览器可以使用GPU渲染的高效的动画属性:

![img1](https://www.evernote.com/l/Aax8Ux7X5RNGX4vg40SfwDkJLGFdh-09k6MB/image.png)

google说明高效的动画: http://www.html5rocks.com/en/tutorials/speed/high-performance-animations/?redirect_from_locale=es