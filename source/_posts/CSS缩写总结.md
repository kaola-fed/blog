---
title: css缩写总结
date: 2018-03-20
---
### CSS缩写总结
常用缩写主要包括以下几点：  
* margin
* padding
* border
* border-radius
* font
* background
* list-style
* flex
* transition
* animation

#### margin 外边距
##### 语法：[ \<length> | \<percentage> ]{1,4}
##### 示例：
margin-top:1px;
margin-right:2px;
margin-bottom:3px;
margin-left:4px;
可以简写为：
margin:1px 2px 3px 4px;
##### 缩写说明：margin: 上 右 下 左
* 四个值：上右下左 TRBL
* 三个值：上右下，然后左复制右
* 二个值：上右，上复制下，左复制右
* 一个值：四边都是这个值
* 引申：padding border类似

#### padding 内边距
和margin类似
##### 语法：[ \<length> | \<percentage> ]{1,4}
##### 缩写说明：padding: 上 右 下 左

#### border 边框
##### 语法：<line-width> || <line-style> || <color>
##### 示例：
border-width:1px;
border-style:solid;
border-color:#000;
可以简写为：
border:1px solid #000;
##### 缩写说明：border: 宽度 样式 颜色
顺序随意，也可省略，但至少出现一个
但是一般为了可读性，均按照宽度、样式、颜色来写

#### border-radius 圆角半径
##### 语法：[ \<length> | \<percentage> ]{1,4} [ / [ \<length> | \<percentage> ]{1,4} ]?
##### 示例：
-moz-border-radius-bottomleft:6px;
-moz-border-radius-topleft:6px;
-moz-border-radius-topright:6px;
-webkit-border-bottom-left-radius:6px;
-webkit-border-top-left-radius:6px;
-webkit-border-top-right-radius:6px;
border-bottom-left-radius:6px;
border-top-left-radius:6px;
border-top-right-radius:6px;
可以简写为：
-moz-border-radius:6px6px0;
-webkit-border-radius:6px6px0;
border-radius:6px6px0;
##### 缩写说明：border-radius: 左上 右上 右下 左下
设置或检索对象使用圆角边框。提供2个参数，2个参数以“/”分隔，每个参数允许设置1~4个参数值，第1个参数表示水平半径，第2个参数表示垂直半径，如第2个参数省略，则默认等于第1个参数
* 水平半径：如果提供全部四个参数值，将按上左(top-left)、上右(top-right)、下右(bottom-right)、下左(bottom-left)的顺序作用于四个角。
* 如果只提供一个，将用于全部的于四个角。
* 如果提供两个，第一个用于上左(top-left)、下右(bottom-right)，第二个用于上右(top-right)、下左(bottom-left)。
* 如果提供三个，第一个用于上左(top-left)，第二个用于上右(top-right)、下左(bottom-left)，第三个用于下右(bottom-right)。
* 垂直半径也遵循以上4点。
* 对应的脚本特性为borderRadius。

#### font 字体
##### 语法：[ [ <' font-style '> || <' font-variant '> || <' font-weight '> ]? <' font-size '> [ / <' line-height '> ]? <' font-family '> ] | caption | icon | menu | message-box | small-caption | status-bar
##### 示例：
font-style:italic;
font-variant:small-caps;
font-weight:bold;font-size:1em;
line-height:140%;
font-family:"Lucida Grande",sans-serif;
可以简写为：
font:italic small-caps bold 1em/140% "Lucida Grande",sans-serif;
##### 缩写说明：
如果你缩写字体定义，至少要定义font-size和font-family两个值。
font: 字体样式 文本是否为小型的大写字母 文本字体的粗细 文本字体尺寸/文本字体的行高 文本使用某个字体或字体序列
前面的字体样式、文本是否为小型的大写字母、文本字体的粗细顺序可变且可省略

#### background
##### 语法：
[ \<bg-layer>, ]* \<final-bg-layer>
\<bg-layer> = \<bg-image> || \<position> [ / \<bg-size> ]? || \<repeat-style> || \<attachment> || \<box> || \<box>
\<final-bg-layer> = \<bg-image> || \<position> [ / \<bg-size> ]? || \<repeat-style> || \<attachment> || \<box> || \<box> || \<background-color>

##### 示例：
background-color:#f00;
background-image:url(background.gif);
background-repeat:no-repeat;
background-attachment:fixed;
background-position:00;
可以简写为：
background:#f00 url(background.gif) no-repeat fixed 0 0;
##### 缩写说明：
background: 颜色 图片 重复 随对象内容滚动还是固定的 位置;
注意这里顺序可变，也就是说前后顺序并不影响，但是一般来说，我们会把color放到最后，因为
<' background-color '> 只能设置一次，且由于写在前面的背景会叠在之后的背景之上，所以背景色通常都定义在最后一组上，避免背景色将图像盖住。
你可以省略其中一个或多个属性值，如果省略，该属性值将用浏览器默认值，默认值为：
* color: transparent
* image: none
* repeat: repeat
* attachment: scroll
* position: 0% 0%

#### list-style 取消默认的圆点和序号
##### 语法：<' list-style-type '> || <' list-style-position '> || <' list-style-image '>
##### 示例：
list-style-type:square;
list-style-position:inside;
list-style-image:url(image.gif);
可以简写为：
list-style:square inside url(image.gif);
##### 缩写说明：
顺序可变，可省略，但至少出现一个
* <' list-style-type '>：设置或检索对象的列表项所使用的预设标记
* <' list-style-position '>：设置或检索作为对象的列表项标记如何根据文本排列
* <' list-style-image '>：设置或检索作为对象的列表项标记的图像

#### flex
##### 语法：none | [ <'flex-grow'> <'flex-shrink'>? || <'flex-basis'> ]
##### 示例
flex-grow: 1;
flex-shrink: 2;
Flex-basis: 10%;
可简写为：
flex: 2 2 10%;
##### 缩写说明  
初始值： 0 1 main-size
* 说明：所能分配到的剩余空间的比例  空间超出时（剩余空间为负时），元素空间怎么分配 初始宽/高
* flex值可以为一个参数、两个参数或者三个参数：
	* 一个参数时，输入数字，表示flex-grow，输入width，表示flex-basis，或者输入none_auto_initial。如果缩写「flex: 1」, 则其计算值为「1 1 0%」；如果缩写「flex: auto」, 则其计算值为「1 1 auto」；如果「flex: none」, 则其计算值为「0 0 auto」；如果「flex: 0 auto」或者「flex: initial」, 则其计算值为「0 1 auto」，即「flex」初始值。
	* 两个参数时，第一个参数一定为数字且表示flex-grow。第二个参数可以是数字，表示flex-shrink，或者为width表示flex-basis。
	* 三个参数时，必须输入数字、数字、width，则分别表示flex-grow、flex-shrink:、Flex-basis
* flex-flow: <'flex-direction'> || <'flex-wrap'> 换行和排列方式，顺序可变

#### transition过渡
##### 语法：
\<single-transition>[,\<single-transition>]*
\<single-transition> = [ none | \<single-transition-property> ] || \<time> || \<single-transition-timing-function> || \<time>
##### 缩写说明：
transition： 对象的过渡属性  持续时间  时间函数（动画类型） 延迟时间
注意顺序均可变，但是如果只提供一个<time>参数，则为 <' transition-duration '> 的值定义；如果提供二个\<time>参数，则第一个为 <' transition-duration'> 的值定义，第二个为 <' transition-delay '> 的值定义
可以为同一元素的多个属性定义过渡效果
如果定义了多个过渡的属性，而其他属性只有一个参数值，则表明所有需要过渡的属性都应用同一个参数值

#### animation 自动运行动画
##### 语法：
\<single-animation>[,\<single-animation>]*
\<single-animation> = \<single-animation-name> || \<time> || \<single-animation-timing-function> || \<time> || \<single-animation-iteration-count> || \<single-animation-direction> || \<single-animation-fill-mode> || \<single-animation-play-state>
##### 缩写说明：
animation：动画名称 持续时间 过渡类型 延迟时间 循环次数 动画方向 保持状态 运行和暂停
* 注意顺序可变，但是如果只提供一个\<time>参数，则为 <' animation-duration '> 的值定义；如果提供二个\<time>参数，则第一个为 <' animation-duration '> 的值定义，第二个为 <' animation-delay '> 的值定义
* 多个属性值以逗号分隔，和transition类似

#### 参考
[CSS | MDN](https://developer.mozilla.org/en-US/docs/Web/CSS)
[快速索引-CSS3参考手册 - CSS3参考手册](http://www.css88.com/book/css/)

by [dj](http://www.kaola.com)