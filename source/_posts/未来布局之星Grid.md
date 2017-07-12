---
title: 未来布局之星Grid
date: 2017-06-30
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
## grid是一个趋势
Grid-layout不是为了取代flex-layout，它是flex的补充。grid擅长二维布局，flex擅长一维布局。他们需要各司其职。

千呼万唤始出来的grid-layout终于在2017年3月开始支持得到了部分浏览器的支持。
![](https://ooo.0o0.ooo/2017/06/30/5956428c59396.png)
flex并不能满足我们对于页面整体布局的需求，[Don't use flexbox for overall page layout](https://jakearchibald.com/2014/dont-use-flexbox-for-page-layout/)这篇文章阐述了这么一个观点：不要用弹性盒子布局整体的页面。而我们在平常使用中也能感受到flex在做整体页面布局的时候，还是有不完善的地方，还是需要大量的inline-box、float来排布内容（// TODO 售后单详情页作为例子）；另外一个问题我们后面了解了grid之后会再说

<!-- more -->

## grid = table2.0?
二维布局方式，可以按照行列方式排列内容。把页面划分为几块，指定不同内容区块的大小、位置和层级。

有观点说grid很像table2.0，他们是有相同之处的。比如都是把元素排列成行和列。但是表格和grid的区别在于，表格是有内容结构的，不能很自由地在里面做布局。而grid内部元素可以自由设定位置，允许重叠和设定层级的样。

## 几个术语
#### 网格容器 grid-container
网格容器为其内容建立新的网格格式化上下文，是内部网格项的边界。

#### 网格线 grid-line
水平垂直分割线，构建出网格轨道、网格单元格和网格区域。就像经纬，分割出南北半球、东西半球，热带、南北温带、南北寒带。网格线是有数字索引的，也可以自己取名字。经纬线都是有数字的，也可以命名，比如本初子午线、赤道。

#### 网格轨道 grid-track
网格内容块之间的水平或垂直的空间。滨盛、滨和、滨兴、滨安、滨康，江陵、江晖、江汉、江虹。（郊区的经典命名，摊手）
 
#### 网格单元格 grid-cell
网格内容的单位区块，是可以放置内容的最小区块。比如用横纵三条网格线划分了页面，那么单元格就是九宫格中的一块

#### 网格区域 grid-area
以网格线为界划定一块区域。原本网易和阿里巴巴都是占用一个单元格，现在都要扩建了，占用两个，两个加起来就是它们各自的网格区域。

## 两个例子
在了解具体的属性之前，[来一个最简单的例子](https://codepen.io/AubreyDDun/pen/KmMmxr)

这样看会觉得grid还是很有趣的，先分块，然后指定每一块的区域范围。块是可以重叠的。但是这只是开始，grid为我们带来了17个新特性，在了解属性之前再来看一个例子，经典布局方案 —— 圣杯布局 holy grail layout。
- 网页常见布局：页眉、页脚、主要内容区块、两边都有一个侧边列。
- 两边带有固定宽度，中间宽度可变
- 中间三列内容等高
- 页脚总处于窗口的底部，即使内容没有填满整个窗口的高度
- 响应式布局，在较小的视窗中所有模块都100%宽度显示
![](https://ooo.0o0.ooo/2017/06/30/595642db36e88.gif)

大家可以迅速在脑子里码一下这个界面写个样式，不至于说复杂 但总之不简单吧。我们[看看grid是怎么做的](http://codepen.io/AubreyDDun/pen/Pmmexe)

```
// key code
.hg__header { grid-area: header;}
.hg__footer { grid-area: footer;}
.hg__main { grid-area: main;}
.hg__left { grid-area: navigation;}
.hg__right { grid-area: ads;}

.hg {
    display: grid;
    grid-template-areas:    "header header header"
                            "navigation main ads"
                            "footer footer footer";
    grid-template-columns: 250px 1fr 300px;
    grid-template-rows: 100px 
                        1fr
                        80px;
    min-height: 100vh;
}
```
步骤：
1. 定义网格

    1. 设定网格区域别名，grid-area: <alias>，指定块所在区域的时候方便引用。前一个简单的例子，grid-area是用来指定网格区域上下左右的网格线各是什么，**所以grid-area既可以指定网格区域的大小和位置，还可以设定区域的别名。**
        ```
        grid-area: main;
            grid-row-start: main;
            grid-column-start: main;
            grid-row-end: main;
            grid-column-end: main;
        ```
    2. grid-template-areas: "- - -" "- - -" "- - -";可以非常直观指定网格的布局。它的值是空格分隔的字符串数组，每一个字符串代表一行。每一个字符串中是空格分隔的网格单元格列表，一个网格区域要跨几列就写几次。比如例子中header、footer写了三次，它们都是跨整个区域宽度的。

2. 设定单元格的高度和宽度

    有一个css3的新单位，fr，在一串数值中出现的话表示根据比例分配某个方向上的剩余空间。设定行高的时候分行写是为了清晰一点。

3. 设定固定位置的页脚
4. 响应式布局，使用媒体查询。重置布局和行高

## grid-container
#### 1. grid-template-columns | grid-template-rows
```grid-template-columns: <track list>```，**定义网格的行数、列数、网格大小**。

有很多中形式，常见的是这么几种：
```
grid-template-columns: 100px 1fr;
grid-template-columns: [linename] 100px;    // 定义网格线名字
grid-template-columns: [linename1] 100px [linename2 linename3]; // 一条网格线多个名字
grid-template-columns: minmax(100px, 1fr);  // 最小100px, 最大满屏
grid-template-columns: fit-content(40%);    // 指定最大值不超过屏宽40%
grid-template-columns: repeat(3, 200px);    // 三列200px
```

```
// 给网格线指定名字
.box {
    grid-template-columns: [first] 40px [line2] 50px [line3] auto [col4-start] 50px [five] 40px [end];
    grid-template-rows: [row1-start] 25% [row1-end] 100px [third-line] auto [last-line];
}
```
![](https://ooo.0o0.ooo/2017/06/30/59564310ef48e.png)

#### 2. grid-template-areas
定义网格区域，使用grid-area调用声明好的网格区域名称放置对应的网格项目。定义一个显式的网格区域。

#### 3. grid-row-gap、grid-column-gap
定义网格之间的间距（不包括grid-item到容器边缘的间距）

#### 4. justify-items: start | end | center | stretch(默认);
定义网格子项的内容水平方向上的对齐方式，类似于flex-container的justify-content，只不过没有space-around和space-between

![1bc096d4ee73927e.png](https://ooo.0o0.ooo/2017/06/30/595645fdcd8e6.png)
![35fecd4fbf754a84.png](https://ooo.0o0.ooo/2017/06/30/59564646583a3.png)
![9a2b4b82c9b6a65b.png](https://ooo.0o0.ooo/2017/06/30/5956465cded77.png)
![85e7fb64351898e4.png](https://ooo.0o0.ooo/2017/06/30/5956465cde14e.png)

#### 5. align-items: start | end | center | stretch(默认);
定义网格子项的内容垂直方向上的对齐方式，类似于flex-container的align-items

![7a410b6d17acd72e.png](https://ooo.0o0.ooo/2017/06/30/595646b3bac49.png)
![20f8f733235797ea.png](https://ooo.0o0.ooo/2017/06/30/595646b3bbae7.png)
![9e1f32b77d573d04.png](https://ooo.0o0.ooo/2017/06/30/595646b3bc94a.png) ![85e7fb64351898e4.png](https://ooo.0o0.ooo/2017/06/30/5956465cde14e.png)

#### 6. justify-content: start | end | center | stretch | space-around | space-between | space-evenly;
当出现网格容器容量大于网格总大小，比如每一个网格子项都用了固定值的时候，指定网格在网格容器中和纵轴的对齐方式。后面三个属性值的区别在：

1. space-around: 始末两端的间距是网格间距的一半
2. space-between: 始末两端的间距为零
3. space-evenly: 始末两端的间距与网格间距相等

 ![](https://ooo.0o0.ooo/2017/06/30/595643c22a6f6.png) ![](https://ooo.0o0.ooo/2017/06/30/595643c21892b.png)
 ![](https://ooo.0o0.ooo/2017/06/30/5956456fe1790.png) ![](https://ooo.0o0.ooo/2017/06/30/595643c207705.png) ![](https://ooo.0o0.ooo/2017/06/30/595643c2179ad.png)
 ![](https://ooo.0o0.ooo/2017/06/30/595643c21b9d2.png)
 ![](https://ooo.0o0.ooo/2017/06/30/595643c21c6d4.png)

#### 7. align-content: start | end | center | stretch | space-around | space-between | space-evenly;

和justify-content对齐方向垂直，指定网格和横轴的对齐方式。

![e14e12bdfbf2e3e4.png](https://ooo.0o0.ooo/2017/06/30/59564729c2235.png)
![8006fb09977ce5c8.png](https://ooo.0o0.ooo/2017/06/30/59564729c3a50.png)
![61ffebd9e0da3f0b.png](https://ooo.0o0.ooo/2017/06/30/59564729c2e4a.png) ![28e49f66a9c4455a.png](https://ooo.0o0.ooo/2017/06/30/5956475e38737.png)
![b9bfc04642c818d7.png](https://ooo.0o0.ooo/2017/06/30/5956475e397f7.png)
![f1845a4b578d8191.png](https://ooo.0o0.ooo/2017/06/30/5956475e3a809.png)
![f06633de75642a36.png](https://ooo.0o0.ooo/2017/06/30/5956475e3b8e4.png)

#### 8. grid-auto-columns、grid-auto-rows; grid-auto-flow
grid-auto-columns | grid-auto-rows作用是网格单元格不够的时候创建隐式的网格放置grid-item。看一个[例子](https://codepen.io/AubreyDDun/pen/RVVywG)

我们只定义了一个1×1的网格容器，box1放了进来，然后其他的三个怎么办呢？漏出来。box2接在box1后面渲染至屏幕右侧，box3和box4在底下渲染，高度仅仅为内容高度。

指定了```grid-auto-columns： 200px; grid-auto-rows: 200px;```，相当于在容器中横纵都创建了更多的隐式的200*200的网格单元来盛放可能多出来的元素。

与之相关的还有另一个属性：grid-auto-flow，在我们没有设定这个属性的时候，多余的元素也按照从左到右从上到下的顺序排列，这个属性是控制自动布局算法的。

```grid-auto-flow: row | column | row dense | column dense;```
1. row为默认值，代表自动布局算法在每一行中依次填充，只有必要时才会添加新行。
2. column代表自动布局算法在每一列中依次填充，只有必要时才会添加新行。
3. dense代表告诉自动布局算法如果更小的子项出现时尝试在网格中填补漏洞。（不建议使用，可能会使布局产生混乱）



##  grid-item
#### 1. grid-column-start | grid-column-end | grid-row-start | grid-row-end
```
grid-row: grid-row-start / grid-row-end
grid-column: grid-column-start / grid-column-end | grid-column-start | span <value>
```
![f0988aebf7ce8786.png](https://ooo.0o0.ooo/2017/06/30/59564780830e7.png)

如果没有显式设置grid-column-end/grid-row-end，网格子项将默认跨越一个网格单元。此外，网格子项可以互相重叠，可以使用z-index来控制他们的层叠顺序。

有一些元素，我们想让它贯穿整个视口，比如像 header、footer，对于小屏幕，两列布局:
```
.header, .footer {
  grid-column: 1 / 3;
}
```
不幸的是，当我们换到大屏的时候，一行12列，这些元素将仅仅占满前两列，并不会占满12列，我们需要定义新的grid-column-end，并且把他的值设为 13. 这种方式比较麻烦，还有一种简单的方式，grid-column: 1 / -1;，这样不论在什么屏幕尺寸下，它们都是占满整行的了。就像下面这样：
```
.header, .footer {
  grid-column: 1 / -1;
}
```

#### 2. grid-area
```grid-area: <name> | grid-row-start / grid-column-start / grid-row-end / grid-column-end```

#### 3. justify-self: start | end | center | stretch 网格单元格内容水平方向上的对齐方式 。与flex中的justify-self。

![c096a95f6300d932.png](https://ooo.0o0.ooo/2017/06/30/5956482659678.png)
![bb01f2f4ea312d78.png](https://ooo.0o0.ooo/2017/06/30/595648265c3d6.png)
![3ca2ef564834e834.png](https://ooo.0o0.ooo/2017/06/30/595648265b54a.png)
![caddebe3320088bf.png](https://ooo.0o0.ooo/2017/06/30/595648265a684.png)
#### 4. align-self: start | end | center | stretch 网格单元格内容垂直方向上的对齐方式 。类似与flex中的align-self。

![1f1b1806e925fe2b.png](https://ooo.0o0.ooo/2017/06/30/59564a4d3fd2f.png)
![grid-align-self-end.png](https://ooo.0o0.ooo/2017/06/30/59564a338ffd8.png)
![0ad87bbf53ecc6a3.png](https://ooo.0o0.ooo/2017/06/30/59564a338f364.png)
![caddebe3320088bf.png](https://ooo.0o0.ooo/2017/06/30/595648265a684.png)

## grid-layout带来的函数
#### 1. repeat()
repeat()提供了一个紧凑的声明方式。如果行列太多并且是规则的分布，我们可以用函数来做网格线的排布。
```
grid-template-columns: repeat(3, 20px [col-start]) 5%;
// 等价于
grid-template-columns: 20px [col-start] 20px [col-start] 20px [col-start] 5%;
}
```
#### 2. minmax()
minmax()相当于为网格线间隔指定一个最小到最大的区间。如果min>max，这个区间就失效了，展示的是min。

#### 3. fit-content()
fit-content(<length-percentage>)相当于 min('max-content', max('auto', argument));


## flex布局的另一个问题
flex-layout的布局另外一个问题是，在有大量内容绘制到页面上、或者内容更改的情况下，在2g或者网络加载不稳定的时候，页面是不稳定的。[Flexbox vs Grid page layout](https://www.youtube.com/watch?v=vPryjyFP5FM&feature=youtu.be)。内容以流式从服务端获取用户可以在页面全部加载出来之前就看到内容，但是在flex布局下会导致布局的重排。正是因为flex本身就是一个弹性的布局。但grid也不是完全可以避免布局重排的问题——有前提：

    必须让你的网格划分是可以预先确定的，比如是根据视窗宽高确定的。如果是根据内容而定，那么也是会崩坏的。

下面的例子中，网格的列宽根据内容而定，因此也会根据内容而变。后面的aside并没有定义在网格容器当中，是一个动态创建的元素。一旦被创建就会导致页面重绘
```
.container {
    display: grid;
    grid-template-columns:
    (foo)   max-content,
    (bar)   min-content,
    (hello) auto;
}

aside {
    grid-column: 4;
}
```
但是也不要放弃flex-layout，它是目前为止最厉害的页面布局属性，是时代召唤的结果，只是它并不适合布局整个页面框架。flex在响应式布局中是很关键的，它是内容驱动型的布局。不需要预先知道会有什么内容，可以设定元素如何分配剩余的空间以及在空间不足的时候如何表现。显得较为强大的是一维布局的能力，而grid优势在于二维布局。这也是他们设计的初衷。

大概可以设想，网格布局被广泛支持之后会出现很多网格布局内嵌flex的布局情形。

subgrid暂时还没有浏览器支持，规范也是在改动之中的，所以先不介绍了。

### 参考

[Don't use flexbox for overall page layout](https://jakearchibald.com/2014/dont-use-flexbox-for-page-layout/)

[(译)原生CSS网格布局学习笔记](https://segmentfault.com/a/1190000007651321)

[网格布局（CSS Grid Layout）浅谈](https://fe.ele.me/wang-ge-bu-ju-css-grid-layout-qian-tan/)

[CSS Grid Layout](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Grid_Layout)

[拥抱未来的CSS布局方式：flex与grid布局](http://www.xingbofeng.com/css-grid-flex/)

[A Complete Guide to CSS Grid Layout](http://chris.house/blog/a-complete-guide-css-grid-layout/)
