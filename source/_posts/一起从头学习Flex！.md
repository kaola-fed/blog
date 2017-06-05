---
title: 一起从头学习Flex！
date: 2017-05-26
---
## 前言
“其实我觉得flex很像现在的人——是适应性很好的生物，在家秋衣秋裤在外香肩外露。flex container看作一个环境或者大佬，flex-item是其中个体或者马仔。个体如何存在很大程度上取决环境的设定，当然也会有一些出世的人 比如```position: absolute;``` 存在感就有比较酷” —— 炖炖

<!-- more -->

## 背景：布局模式
页面拆开了看，无非是由众多尺寸各异的box以不同层叠水平和顺序组合而成。

CSS2.1定义了下面前4种布局模式，不同的布局模式下box内的渲染规则不同。

| | 布局 | 特性
---|---|---
block layout | 块级布局 | 独立的渲染区域，与外界无关
inline layout | 行内布局 | 为布局文本设计
table layout | 表格布局 | 在表格格子布局二维数据
positioned layout | 定位布局 | 不考虑文件流中其他元素，布局有非常明确的位置
**flex layout** | **弹性布局** | **呈现复杂的应用与页面**
flex-layout和block-layout非常相似。

块级布局中一些比较复杂的属性，比如浮动、多列，在弹性布局中是没有的；

反过来弹性布局有一些很简单但厉害的属性去实现复杂的网页布局中常见的需求。

## flex-container档案
1. 元素名：flex container
2. 中文名：弹性容器、弹性盒
3. 身份：大佬
4. 年代：css3，chrome29就已经可以不加前缀地支持了
5. 权力：弹性布局——指定容器中元素的布局方式，子元素在其中是否收缩或伸展
6. 被接受程度：所有的浏览器都支持flexbox（[caniuse](http://caniuse.com/flexbox)）
    ![image](https://raw.githubusercontent.com/lzxb/flex.css/master/shot/caniuse.png)

        flex layout is split into three versions: 
        1. old version  display: box;
        2. transitional version: display:flexbox;
        3. present standard version: display:flex;
        
        Android
        2.3  began to support old version: display: -webkit-box;
        4.4 began to support standard version: display: flex;
        
        IOS
        6.1  began to support old version: display: -webkit-box;
        7.1 began to support standard version: display: flex;
        
7. 如何成为~~大佬~~ **flex**：```display:flex/inline-flex;``` 分别产生块级和行内的弹性容器。
8. 社会关系：弹性容器中的子元素——**flex items**
9. 主要参数：主轴、侧轴，主轴长度、侧轴长度
    ![flexbox](http://p1.bpimg.com/567571/492552df52d4b872.png)
10. 速记几个属性：
    - **flex-flow** 子元素排列的方向和是否换行 ```flex-flow: [<flex-direction> <flex-wrap>]```
        1. **flex-direction** 排列方向和顺序（主轴侧轴方向决定）
            
            ```flex-direction: row | row-reverse | column | column-reverse```
        2. **flex-wrap** 主轴个数和排列顺序
        
            ```flex-wrap: nowrap | wrap | wrap-reverse```
            
            ![](http://i1.piimg.com/567571/a1aab184f0e15918.png)
            ![](http://i1.piimg.com/567571/afebc19a533dea82.png)
            ![](http://p1.bpimg.com/567571/bb4191279cb21de3.png)

            eg. ```flex-flow: row || column wrap || row-reverse wrap-reverse```

    - **justify-content** 安排（~~决定~~）子元素在主轴上如何分配空白空间的方式
        
        ```justify-content: flex-start | flex-end | center | space-between | space-around```
        ![](http://p1.bpimg.com/567571/b554af8f1c5d613b.png)
        
    - **align-items** 安排（~~决定~~）子元素在纵轴上的伸展方式
        
        ```align-items: flex-start | flex-end | center | baseline | stretch```
        ![](http://p1.bpimg.com/567571/de0c8cd891e520b9.png)
        
    - **align-content** 多条主轴如何在纵轴方向上面排布，跟justify-content 在主轴方向对齐元素的方式相似，如果只有一行该属性不生效
        
        ```align-content: flex-start | flex-end | center | space-between | space-around | stretch```
        ![](http://p1.bqimg.com/567571/a25d0fdca940eaa6.png)
        
## flex-items档案
1. 元素名：flex item
2. 中文名：弹性子元素
3. 身份：flex container的马仔
4. 如何成为~~马仔~~ **flex-item**：在flex container里面的子元素自动成为flex item，直接包含在容器中的连续文本会被包裹成一个匿名的flex item，空白的item不会被渲染出来
5. 职责：为其内容创建新的formatting context，flex item是flex-level的块，所以使用弹性容器的弹性格式化上下文而不是BFC（比如在BFC中margin会坍塌在弹性格式化上下文中不会 [戳](https://codepen.io/AubreyDDun/pen/dWYzbb)）
6. 比较酷的**绝对定位**的弹性子元素 特性速记
    - 不参与flex布局，脱离文档流 （[戳](http://codepen.io/anon/pen/NjGjyV)）
    - ```align-self: auto/stretch``` = ```align:flex-start``` 默认布局从主轴开始位置放置
    - ```align-self: center``` 垂直方向居中([chrome遵守了这一规范，IE/FF没有，戳](http://codepen.io/AubreyDDun/pen/vmNZxM))
7. 几个重要属性：
    - **margins** 和 **paddings**：flex item的margins不会重叠（应该尽量避免使用百分比的margins、paddings，在不同浏览器下表现不一样，目前在IE/FF/Chrome还没有测试出来差别啦） 百分比的基准是浏览器宽度
    
    - 有order的元素渲染顺序会重置，相当于指定特定的子元素出现在什么位置
    
    - ```visibility: collapse```  [看例子](https://codepen.io/AubreyDDun/pen/MmaEzy)
    
    - **flex** 子元素的弹性
        
        ```flex: [ <flex-grow> <flex-shrink> <flex-basis>]``` 详见flex缩写探究
    
    - ```margin: auto``` 会“贪吃”掉所有空白的区域 [例子](https://codepen.io/AubreyDDun/pen/YVyYKP)
    
    - **align-self** 单独设置子元素（包括绝对定位的子元素）在纵轴上的定位方式，可以覆盖align-items属性
    
        ```align-self: auto | flex-start | flex-end | center | baseline | stretch``` （auto是跟container的flex-items属性一致）

### flex缩写的探究
```flex: [ <flex-grow> <flex-shrink> <flex-basis>]```

- **flex-grow** 默认1 如何分配container中的空白空间，数值决定权重
- **flex-shrink** 默认1 在空间不够的时候如何收缩元素
- **flex-basis** 指定flex-item的默认尺寸 百分比以父级容器宽度为基数计算（[Flexbox Tester](https://madebymike.com.au/demos/flexbox-tester/)可以帮助理解flex item宽度的工具网站）


缩写 | flex-grow | flex-shrink | flex-basis
---|---|---|---
no declaration | 0 | 1 | auto
auto | 1 | 1 | auto
0 | 0 | 1 | 0%
1 | 1 | 1 | 0%
none | 0 | 0 | auto


### 弹性布局不是块级布局
1. float不生效
2. 相邻子元素的margin不会发生重叠
3. 多列布局column-*不起作用
4. 'float'和'clear'对flex item不产生浮动或者清除浮动的效果，也不会让它脱离文档流
5. ```vertical-align```对于flex item没效果
6.  伪元素```::first-line```、```::first-letter```不适用于弹性容器，弹性容器也不会成为其父元素的first-line或者first-letter

### 为什么你的flexbox不管用
"了解一个人并不代表什么，人是会变的"——《重庆森林》。放在W3C规范和浏览器实现就是“了解一个属性并不代表什么，浏览器实现都是有差异的”。

**W3C只是制定了规范**，但是遵守程度和实现效果各个浏览器是有所差异的。当你清楚你想实现什么效果而使用了对应的属性却发现不起作用，那么很可能是浏览器的差异造成的。比如上面提到的flex缩写，IE10的实现是不符合规范的
Declaration | What it should mean | What it means in IE 10
---|---|---
(no flex declaration) | ```flex: 0 1 auto``` | ```flex: 0 0 auto```
```flex: 1``` | ```flex: 1 1 0%``` | ```flex: 1 0 0px```
```flex: auto``` | ```flex: 1 1 auto``` | ```flex: 1 0 auto```
```flex: initial``` | ```flex: 0 1 auto``` | ```flex: 0 0 auto```

另外一个可能不常见但也能体现浏览器差异的是，嵌套的弹性容器的baseline不作为其他弹性子元素的对齐参考（可以在Firefox[戳例子](http://codepen.io/philipwalton/pen/vOOejZ)看看）。

“当你在构建页面的时候使用了flexbox但是发现他并没有出现你预想的效果，可以在这里找到答案”——[flexbugs](https://github.com/philipwalton/flexbugs)

### 关于flex.css
flex.css就是对flex布局的一种封装，通过简洁的属性设置就能使得它完美的运行在移动端的各种浏览器，甚至能运行在ie10+的各种PC端浏览器中。是一种类似于标签属性声明的方式，指定flex属性。
```
<!-- 设置主轴方向 -->
<div class="flexbox" flex="dir: top">   
    ...
</div>
```

flex.css html | css
---|---
dir: top | flex-direction: cloumn
main: center | justify-content: center
cross: top | align-items: flex-start


**试验flex布局的工具网站：** [flexbox playground and code generator](http://the-echoplex.net/flexyboxes/)

#### 参考

1. [W3C flexbox](https://www.w3.org/TR/css-flexbox-1/#placement)
2. [flex.css快速入门，极速布局](https://gold.xitu.io/post/582d991cc4c9710054407dc3)
3. [flexbugs](https://github.com/philipwalton/flexbugs)
4. [使用弹性盒子进行高级布局](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_Flexible_Box_Layout/%E5%BC%B9%E6%80%A7%E6%A1%86%E7%9A%84%E9%AB%98%E7%BA%A7%E5%B8%83%E5%B1%80)
5. [Flex 布局教程：语法篇](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html?utm_source=tuicool)


by DDun