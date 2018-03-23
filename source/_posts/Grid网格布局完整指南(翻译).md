原文链接[A Complete Guide to Grid](https://css-tricks.com/snippets/css/complete-guide-grid/)

css网格布局是css中最强大的布局系统。这是一个二维系统，这意味着可以同时处理列和行，不同于flex布局，它主要是个一维系统，我们可以使用网格布局的css规则来应用于父元素(网格容器)和子元素(网格元素)。

#### 介绍

css网格布局（又名'网格'）是一个二维基于网格布局的系统，它的目的在于完全改变我们设计基于网格布局的用户界面的方式。css一直用来布局我们的网页，但是它从来没有做过很好的工作。首先我们使用`table`，然后是`floats`、`positioning`、`inline-block`，但是上述所有方法都是有漏洞的，并且也遗留了很多重要的功能(例如垂直居中)。Flex布局实现了，但是它的目的是为了跟简单的一维布局，而不是复杂的二维布局(Flexbox和Grid在一起也能很好的工作)。网格布局是第一个专门为解决布局问题而创建的css模块，只要我们一直在制作网站，并解决这些问题。

有两件主要的事情，鼓励我创建这份指南。首先是Rachel Andrew's的书[Get Ready for CSS Grid Layout](http://abookapart.com/products/get-ready-for-css-grid-layout)。它清晰透彻的介绍了网格布局，也是本文的基础。我强烈建议你购买并且阅读。另外一个激励我的事情来自于Chris Coyier的文章[FlexBox完整指南](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)，这是FlexBox最前沿的资源。它帮助了很多人，事实证明，当你google有关FlexBox时，他是第一个出来的结果。你会发现我的文章和他的有很多相似的地方，为什么不从最好的那边去学习呢？

本指南的目的是介绍网格布局，因为它们存在于最新版本的规范中，所以我不会覆盖过时的IE语法，而且随着规范的成熟，我会尽最大努力更新本指南

#### 基础和浏览器支持

首先，你需要定义一个容器元素具有`display:grid`属性，使用`grid-template-columns`和`grid-template-rows`设置行和列的大小，然后通过使用`grid-column`和`grid-row`把子元素放入gird容器中。和FlexBox相似的是，gird元素的排列顺序并不和既定顺序匹配。你可以使用css属性来排列顺序，这使得通过媒体查询重新排列网格元素变得非常容易，想象一下，通过几行css定义整个页面的布局，然后完全重新整理它来适应不通的屏幕宽度，网格布局是有史以来css最强大的模块之一。

截止2017年3月，许多浏览器都提供了原生的，前瞻性的对CSS Grid的支持：Chrome（包括Android），Firefox，Safari（包括iOS）和Opera。另一方面，IE10和11都宣布支持他，但是这是一个过时的语法实现。Edge已经宣布支持，但是目前还没有实现。

这个浏览器支持数据来自Caniuse，它有更多的细节。数字标识浏览器的版本及以上能支持这些功能

##### Desktop

| Chrome | Opera | Firefox | IE   | Edge | Safari |
| ------ | ----- | ------- | ---- | ---- | ------ |
| 57     | 44    | 52      | 11*  | 16   | 10.1   |

##### Mobile / Tablet

| iOS Safari | Opera Mobile | Opera Mini | Android | Android Chrome | Android Firefox |
| ---------- | ------------ | ---------- | ------- | -------------- | --------------- |
| 10.3       | No           | No         | 62      | 62             | 57              |

除了微软以外，浏览器制造商似乎不愿意让grid在外界暴露，知道他成熟为止。这是一件好事儿，这意味着我们不需要担心学习过多的语法。

在生产环境使用Gri只是时间问题了，但是现在是我们学习的时候了。

#### 重要术语

在深入了解网格概念之前，理解术语是很重要的。由于这里所涉及的术语在概念上都是相似的，如果不先记住它们在网格规范中定义的含义，就很容易将它们相互混淆。但是不用担心，并没有多少概念。

##### 网格容器

当`display: grid`应用在元素上的时候。它将变成其他grid元素的直接父元素，在下面的例子中，`.container`就是grid容器

```html
<div class="container">
  <div class="item item-1"></div>
  <div class="item item-2"></div>
  <div class="item item-3"></div>
</div>
```

```css
.container {
  display: grid
}
```

##### 网格元素

网格容器的子元素直接后代，如下例子，`.sub-item`就不是网格元素

```html
<div class="container">
  <div class="item item-1"></div>
  <div class="item item-2">
    <p class="sub-item"></p>
  </div>
  <div class="item item-3"></div>
</div>
```

##### 网格线

构成网格结构的分界线。他们可以是垂直的，也可以是水平的，并位于行或列的任一侧，下图中的黄线就是列网格线

![grid-line](http://oznz1caxs.bkt.clouddn.com/grid-line.png)

##### 网格轨道

两个相邻网格线之间的空间，你可以把他想象成网格的列或行，如下图第二行和第三行之间的空间

![grid-track](http://oznz1caxs.bkt.clouddn.com/grid-track.png)

##### 网格单元格

两个相邻的行和两个相邻的列网格线之间的空间，这是网格里一个单独的单元，如下图：

![grid-cell](http://oznz1caxs.bkt.clouddn.com/grid-cell.png)

##### 网格区域

四个网格线包围的总空间。 网格区域可以由任意数量的网格单元组成，如下图：

![grid-area](http://oznz1caxs.bkt.clouddn.com/grid-area.png)

#### 网格属性(作用于网格容器)

##### display：将元素定义为网格容器，并为它的内容建立新的网格格式上下文。

- **grid**：生成块儿级别的网格
- **inline-grid**：生成一个内联级别的网格
- **subgrid**：如果你的网格容器本身就是一个网格物体（即嵌套网格），你可以使用这个属性来表示你想从它的父节点取得它的行/列的大小，而不是指定它自己的大小。

```css
.container: {
   display: grid | inline-grid | subgrid;
}
/* 注意:column, float, clear, and vertical-align 对grid容器没有影响 */
```



##### grid-template-columns && grid-template-rows：使用一组空格分隔的值定义网格的列和行，这些值表示了轨道的大小，他们之间的空间代表网格线。

- **track-size**：可以是具体的长度，百分比，或者是网格中的一小部分空闲空间（使用fr单位）
- **line-name**：自己选择的任意名称

```css
.container {
  grid-template-coloumns: <track-size> ... | <line-name> <track-size> ...;
  grid-template-rows: <track-size> ... | <line-name> <track-size> ...;
}
```

例子：在轨迹值之间流出空白区域时，网格线会自动分配数字名称

```
.container {
  grid-template-columns: 40px 50px auto 50px 40px;
  grid-template-rows: 25% 100px auto;
}
```

![grid-template-*](http://oznz1caxs.bkt.clouddn.com/grid-numbers.png)

但是你可以选择使用中括号给网格线命名。

```css
.container {
  grid-template-columns: [first] 40px [line2] 50px [line3] auto [col4-start] 50px [five] 40px [end];
  grid-template-rows: [row1-start] 25% [row1-end] 100px [third-line] auto [last-line];
}
```

![grid-names](http://oznz1caxs.bkt.clouddn.com/grid-names.png)

需要注意点是，每个网格线都可以拥有多个名字，在中括号内用空格分割下，如下，第二行网格线有两个名字row1-end和row2-start：

```css
.container {
  grid-template-rows: [row1-start] 25% [row1-end row2-start] 25% [row2-end];
}
```

如果你定义的属性包括重复的部分，则可以用reapeat()方法来简化写法。

```css
.container {
  grid-template-rows: repeat(3, 20px [col-start]) 5%;
}
/* 相当于 */
.container {
  grid-template-rows: 20px [col-start] 20px [col-start] 20px [col-start] 5%
}

```

fr单位允许将轨道的大小设置成网格容器空闲空间的一部分。如下，这会将每个项目设置为网格容器宽度的三分之一：

```css
.container {
  grid-template-columns: 1fr 1fr 1fr;
}
```

可用空闲空指出去其他单位所占之外的空间。如下，fr单元可用的控件总量不包括50px。

```css
.container {
  grid-template-columns: 1fr 1fr 1fr 50px;
}
```



##### grid-template-areas: 通过引用网格区域的属性指定网格区域的名称来定义网格模板。重复网格区域的名称使得内容延伸到这些单元格，一个.符号表示一个空的单元格，这种语法本身提供了网格结构的可视化。值：

- **grid-area-name**: 指定网格区域的名称
- **.**：表示一个空的网格单元格
- **none**：没有网格区域被定义

```css
.container {
  grid-template-areas:  "<grid-area-name> | . | none | ..."
    "...";
}
```

例子：

```css
.item-a {
  grid-area: header;
}
.item-b {
  grid-area: main;
}
.item-c {
  grid-area: sidebar;
}
.item-d {
  grid-area: footer;
}

.container {
  grid-template-columns: 50px 50px 50px 50px;
  grid-template-rows: auto;
  grid-template-areas: 
    "header header header header"
    "main main . sidebar"
    "footer footer footer footer";
}
```

这将创建一个四列三行高的网格。 整个顶行将由标题区域组成。 中间一排将由两个主要区域组成，一个是空单元格，另一个是侧栏区域。 最后一行是所有页脚。

![grid-template-area](http://oznz1caxs.bkt.clouddn.com/grid-template-areas.png)

你的声明中的每一行都需要有相同数量的单元格。

您可以使用任意数量的`.`来声明单个空单元格。 只要这些`.`之间没有空间，他们就代表了一个单一的单元格。

注意你不是用这个语法命名行，只是区域。 当你使用这种语法时，区域两端的行实际上是自动命名的。 如果网格区域的名称是foo，则区域的起始行行和起始列行的名称将是foo-start，并且其最后一行行和最后一行的名称将是foo-end。 这意味着一些行可能有多个名字，比如上面的例子中最左边的一行，它有三个名字：header-start，main-start和footer-start。



##### grid-template：在单个声明中设置grid-template-rows，grid-template-columns, grid-template-areasd的简写。值：

- **none**：将三个属性设置为初始值
- **subgrid**：将`gird-template-rows`和`grid-template-columns`的值设为`subgrid`，并且将`grid-template-ares`设置为初始值
- **grid-template-rows/ grid-template-columns**：将`gird-template-rows`和`grid-template-columns`设为指定的值，并将`gird-template-areas`设置为`none`

```css
.container {
  grid-template: none | subgrid | <grid-template-rows> / <grid-template-columns>;
}
```

他也能接受更复杂但是用起来很方便的属性设置三个值，下面是例子:

```css
.container {
  grid-template:
    [row1-start] "header header header" 25px [row1-end]
    [row2-start] "footer footer footer" 25px [row2-end]
    / auto 50px auto;
}
/* 等同于 */
.container {
  grid-template-rows: [row1-start] 25px [row1-end row2-start] 25px [row2-end]
  grid-template-columns: auto 50px auto;
  grid-template-areas:
    "header header header"
    "footer footer footer"
}
```

由于`grid-template`并没有重置隐式的网格属性(`grid-auto-columns`, `grid-auto-rows`和`grid-auto-flow`)，在绝大多数情况下你会使用`grid`属性替代`grid-template`属性



##### grid-column-gap && grid-row-gap: 指定网格线的尺寸，你可以想象成在行列之间设置沟壑的大小。值：

- **line-size**: 一个长度值

```css
.container {
  grid-column-gap: <line-size>;
  grid-row-gap: <line-size>
}
```

例子：

```css
.container {
  grid-template-columns: 100px 50px 100px;
  grid-template-rows: 80px auto 80px; 
  grid-column-gap: 10px;
  grid-row-gap: 15px;
}
```

![grid-gap](http://oznz1caxs.bkt.clouddn.com/grid-column-row-gap.png)

沟壑只能在行和列之间被生成，不能在外部的边缘生成。



##### grid-gap：`grid-column-gap`和`grid-row-gap`的简写形式。值：

- **grid-row-gap grid-column-gap**：长度单位。

```css
.container {
  grid-gap: <grid-row-gap> <grid-column-gap>
}
```

例子：

```css
.container {
  grid-template-columns: 100px 50px 100px;
  grid-template-rows: 80px auto 80px; 
  grid-gap: 10px 15px;
}
```

如果`grid-row-gap`的值没有被设定，那么他的值会设置为`grid-column-gap`的值。



##### justify-items：沿着行轴对齐网格中的内容（而不是沿着列轴对齐）,这个属性适用于所用在容器中的网格元素。值：

- **start**：将内容对齐到网格区域的左端
- **end**：将内容对齐到网格区域的右端
- **center**：在网格区域中心对齐
- **stretch**(默认)：填充网格区域的整个宽度

```css
.container {
  justify-items: start | end | center | stretch;
}
```

例子：

```css
.container {
  justify-items: start;
}
```

![grid-justify-items-start](http://oznz1caxs.bkt.clouddn.com/grid-justify-items-start.png)

```css
.container {
  justify-items: end;
}
```

![grid-justify-items-end](http://oznz1caxs.bkt.clouddn.com/grid-justify-items-end.png)

```css
.container {
  justify-items: center;
}
```

![grid-justify-items-center](http://oznz1caxs.bkt.clouddn.com/grid-justify-items-center.png)

```css
.container {
  justify-items: stretch;
}
```

![grid-justify-items-stretch](http://oznz1caxs.bkt.clouddn.com/grid-justify-items-stretch.png)

这种行为也可以通过`justify-self`属性设置在单独的网格元素上



##### align-items：沿着列轴对齐网格中的内容（而不是沿着横轴对齐）,这个属性适用于所用在容器中的网格元素。值：

- **start**：将内容对齐到网格区域的顶部
- **end**: 将内容对齐到网格区域的底部
- **center**：在网格区域中心对齐
- **stretch**(默认)：填充网格区域的整个高度

```css
.container {
  align-items: start | end | center | stretch;
}
```

例子：

```css
.container {
  align-items: start;
}
```

![grid-align-items-start](http://oznz1caxs.bkt.clouddn.com/grid-align-items-start.png)

```css
.container {
  align-items: end;
}
```

![grid-align-items-end](http://oznz1caxs.bkt.clouddn.com/grid-align-items-end.png)

```css
.container {
  align-items: center;
}
```

![grid-align-items-center](http://oznz1caxs.bkt.clouddn.com/grid-align-items-center.png)

```css
.container {
  align-items: stretch;
}
```

![grid-align-items-stretch](http://oznz1caxs.bkt.clouddn.com/grid-align-items-stretch.png)

这种行为也可以通过`align-self`属性设置在单独的网格元素上



##### justify-content：有时，网格的总大小可能小于其网格容器的大小。如果您的所有网格项目都使用像px这样的非灵活单位进行大小设置，则可能发生这种情况。在这种情况下，您可以设置网格在网格容器内的对齐方式， 此属性沿着行轴对齐网格（和`align-content`相反而不是沿着列轴方向对齐）。值：

- **start**：将网格对齐在网格容器的左端
- **end**：将网格对齐在网格容器的右端
- **center**：将网格对齐在网格容器的中间
- **stretch**：调整网格项目的大小以允许网格填充网格容器的整个宽度
- **space-around**：在每个栅格项目之间放置一个均匀的空间，在两端放置一半的空间
- **space-between**：在每个网格物体之间放置一个均匀的空间，在两端没有空间
- **space-evenly**：在每个网格物体之间放置一个均匀的空间，在两端放置一样的空间

```css
.container {
  justify-content: start | end | center | stretch | space-around | space-between | space-evenly;
}
```

例子：

```css
.container {
  justify-content: start;
}
```

![grid-justify-content-start](http://oznz1caxs.bkt.clouddn.com/grid-justify-content-start.png)

```css
.container {
  justify-content: end;
}
```

![grid-justify-content-end](http://oznz1caxs.bkt.clouddn.com/grid-justify-content-end.png)

```css
.container {
  justify-content: center;
}
```

![grid-justify-content-center](http://oznz1caxs.bkt.clouddn.com/grid-justify-content-center.png)

```css
.container {
  justify-content: stretch;	
}
```

![grid-justify-content-stretch](http://oznz1caxs.bkt.clouddn.com/grid-justify-content-stretch.png)

```css
.container {
  justify-content: space-around;	
}
```

![grid-justify-content-space-around](http://oznz1caxs.bkt.clouddn.com/grid-justify-content-space-around.png)

```css
.container {
  justify-content: space-between;	
}
```

![grid-justify-content-space-between](http://oznz1caxs.bkt.clouddn.com/grid-justify-content-space-between.png)

```css
.container {
  justify-content: space-evenly;	
}
```

![grid-justify-content-space-evenly](http://oznz1caxs.bkt.clouddn.com/grid-justify-content-space-evenly.png)



##### align-content：有时，网格的总大小可能小于其网格容器的大小。如果您的所有网格项目都使用像px这样的非灵活单位进行大小设置，则可能发生这种情况。在这种情况下，您可以设置网格在网格容器内的对齐方式， 此属性沿着列轴对齐网格（和`justify-content`相反而不是沿着行轴方向对齐）。值：

- **start**：将网格对齐在网格容器的顶部
- **end**：将网格对齐在网格容器的底部
- **center**：将网格对齐在网格容器的中间
- **stretch**：调整网格项目的大小以允许网格填充网格容器的整个宽度
- **space-around**：在每个栅格项目之间放置一个均匀的空间，在两端放置一半的空间
- **space-between**：在每个网格物体之间放置一个均匀的空间，在两端没有空间
- **space-evenly**：在每个网格物体之间放置一个均匀的空间，在两端放置一样的空间

```css
.container {
  align-content: start | end | center | stretch | space-around | space-between | space-evenly;
}
```

例子：

```css
.container {
  align-content: start;
}
```

![grid-align-content-start](http://oznz1caxs.bkt.clouddn.com/grid-align-content-start.png)

```css
.container {
  align-content: end;	
}
```

![grid-align-content-end](http://oznz1caxs.bkt.clouddn.com/grid-align-content-end.png)

```css
.container {
  align-content: center;	
}
```

![grid-align-content-center](http://oznz1caxs.bkt.clouddn.com/grid-align-content-center.png)

```css
.container {
  align-content: stretch;	
}
```

![grid-align-content-stretch](http://oznz1caxs.bkt.clouddn.com/grid-align-content-stretch.png)

```css
.container {
  align-content: space-around;	
}
```

![grid-align-content-space-around](http://oznz1caxs.bkt.clouddn.com/grid-align-content-space-around.png)

```css
.container {
  align-content: space-between;	
}
```

![grid-align-content-space-between](http://oznz1caxs.bkt.clouddn.com/grid-align-content-space-between.png)

```css
.container {
  align-content: space-evenly;	
}
```

![grid-align-content-space-evenly](http://oznz1caxs.bkt.clouddn.com/grid-align-content-space-evenly.png)



##### grid-auto-columns&grid-auto-rows：指定所有自动生成的网格轨道（又名隐式网格轨道）的大小。 隐式网格轨道在您明确定位超出定义网格范围的行或列（通过`grid-template-rows`/`grid-template-columns`）时被创建。值：

- **轨道大小**：可以是固定的长度，或者是百分比，或者是网格中空闲的空间长度（使用fr单位）

```css
.container {
  grid-auto-columns: <track-size> ...;
  grid-auto-rows: <track-size> ...;
}
```

为了说明如何创建隐式网格轨道，如下：

```css
.container {
  grid-template-columns: 60px 60px;
  grid-template-rows: 90px 90px
}
```

![grid-auto](http://oznz1caxs.bkt.clouddn.com/grid-auto.png)

这创建了2X2的网格。但是你想象一下，如果你使用`gird-column`和`grid-row`来定位网格元素，如下：

```css
.item-a {
  grid-column: 1 / 2;
  grid-row: 2 / 3;
}
.item-b {
  grid-column: 5 / 6;
  grid-row: 2 / 3;
}
```

![implicit-tracks](http://oznz1caxs.bkt.clouddn.com/implicit-tracks.png)



我们定义.item-b从第5行开始到第6行结束，但是我们从来没有定义第5行或第6行。因为我们引用了不存在的行，所以创建宽度为0的隐式轨道来填充 在差距。 我们可以使用网格自动列和网格自动行来指定这些隐式轨道的宽度：

```css
.container {
  grid-auto-columns: 60px;
}
```

![implicit-tracks-with-widths](http://oznz1caxs.bkt.clouddn.com/implicit-tracks-with-widths.png)



##### grid-auto-flow：如果你有网格元素没有明确的放置在网格中，自动放置算法会自动放置项目。该属性控制自动布局算法的工作。值：

- **row**：告诉自动布局算法依次填充每行，根据需要添加新行。
- **column**：告诉自动布局算法依次填入每列，根据需要添加新列。
- **dense**：告诉自动布局算法，如果稍后出现较小的项目，则尝试在网格中更早地填充空洞。

```css
.container {
  grid-auto-flow: row | column | row dense | column dense
}
```

请注意， `dense`可能导致您的项目出现乱序。

例子：

```html
<section class="container">
  <div class="item-a">item-a</div>
  <div class="item-b">item-b</div>
  <div class="item-c">item-c</div>
  <div class="item-d">item-d</div>
  <div class="item-e">item-e</div>
</section>
```

定义一个有5列和2行的网格，并将`grid-auto-flow`设置为`row`（默认值）：

```css
.container {
  display: grid;
  grid-template-columns: 60px 60px 60px 60px 60px;
  grid-template-rows: 30px 30px;
  grid-auto-flow: row;
}
```

将物品放在网格上时，只能为其中的两个点指定点：

```css
.item-a {
  grid-column: 1;
  grid-row: 1 / 3;
}
.item-e {
  grid-column: 5;
  grid-row: 1 / 3;
}
```

因为我们将`grid-auto-flow`设置为`row`，所以我们的`grid`就像这样。 注意我们没有放置的三个项目（item-b，item-c和item-d）是如何流过可用的行的：

![grid-auto-flow-row](http://oznz1caxs.bkt.clouddn.com/grid-auto-flow-row.png)

如果我们将grid-auto-flow设置为column，则item-b，item-c和item-d向下流动列：

```css
.container {
  display: grid;
  grid-template-columns: 60px 60px 60px 60px 60px;
  grid-template-rows: 30px 30px;
  grid-auto-flow: column;
}
```

![grid-auto-flow-column](http://oznz1caxs.bkt.clouddn.com/grid-auto-flow-column.png)



##### grid：在单个声明中设置所有以下属性的简写,`grid-template-rows`，`grid-template-columns`，`grid-template-areas`，`grid-auto-rows`，`grid-auto-columns`和`grid-auto-flow`。 它也将`grid-column-gap`和`grid-row-gap`设置为它们的初始值，即使它们不能由此属性明确设置。值：

- **none**：将所有子属性设置为其初始值
- **grid-template-rows / grid-template-columns**：将`grid-template-rows`和`grid-template-columns`分别设置为指定值，将所有其他子属性设置为其初始值
- **grid-auto-flow [grid-auto-rows [ / grid-auto-columns] ] **: 接受所有与`grid-auto-flow`，`grid-auto-rows`和`grid-auto-columns`相同的值。 如果省略`grid-auto-columns`，则将其设置为为`grid-auto-rows`指定的值。 如果两者都被省略，则它们被设置为它们的初始值

```css
.container {
    grid: none | <grid-template-rows> / <grid-template-columns> | <grid-auto-flow> [<grid-auto-rows> [/ <grid-auto-columns>]];
}

```

以下两个代码块是等效的：

```css
.container {
  grid: 200px auto / 1fr auto 1fr;
}
.container {
  grid-template-rows: 200px auto;
  grid-template-columns: 1fr auto 1fr;
  grid-template-areas: none;
}
```

```
.container {
  grid: column 1fr / auto;
}
.container {
  grid-auto-flow: column;
  grid-auto-rows: 1fr;
  grid-auto-columns: auto;
}
```

它也接受一个更复杂但相当方便的语法来一次设置所有内容。 您可以指定网格模板区域，网格模板行和网格模板列，并将所有其他子属性设置为其初始值。 你在做什么是指定线内名称和轨道大小与他们各自的网格区域。 下面是最简单的例子来描述：

```css
.container {
  grid: [row1-start] "header header header" 1fr [row1-end]
        [row2-start] "footer footer footer" 25px [row2-end]
        / auto 50px auto;
}

.container {
  grid-template-areas: 
    "header header header"
    "footer footer footer";
  grid-template-rows: [row1-start] 1fr [row1-end row2-start] 25px [row2-end];
  grid-template-columns: auto 50px auto;    
}
```



#### 网格元素属性（作用于子元素）

##### grid-column-start&grid-column-end&grid-row-start&grid-row-end：通过引用特定的网格线来确定网格内网格的位置。` grid-column-start` / `grid-row-start`是项目开始的行，`grid-column-end `/ `grid-row-end`是项目结束的行。

- **line**：可以是一个数字来引用一个编号的网格线，或者一个名字来引用一个命名的网格线
- **span <number>**：该项目将跨越所提供的网格轨道数量
- **span <name>**：该项目将跨越，直到它与提供的名称命中下一行
- **auto**：表示自动放置，自动跨度或默认跨度

```css
.item {
  grid-column-start: <number> | <name> | span <number> | span <name> | auto
  grid-column-end: <number> | <name> | span <number> | span <name> | auto
  grid-row-start: <number> | <name> | span <number> | span <name> | auto
  grid-row-end: <number> | <name> | span <number> | span <name> | auto
}
```

例子：

```css
.item-a {
  grid-column-start: 2;
  grid-column-end: five;
  grid-row-start: row1-start
  grid-row-end: 3
}
```

![grid-start-end-a](http://oznz1caxs.bkt.clouddn.com/grid-start-end-a.png)

```css
.item-b {
  grid-column-start: 1;
  grid-column-end: span col4-start;
  grid-row-start: 2
  grid-row-end: span 2
}
```

![grid-start-end-b](http://oznz1caxs.bkt.clouddn.com/grid-start-end-b.png)

如果没有声明`grid-column-end` / `grid-row-end`，默认情况下，该项目将跨越1个轨道。

项目可以相互重叠。 可以使用z-index来控制它们的堆叠顺序。



##### grid-column & grid-row：`grid-column-start` + `grid-column-end`, 和` grid-row-start` + `grid-row-end`的缩写形式。值：

- **start-line / end-line**: 接受的值和复杂形式的一样，包括`span`

```css
.item {
  grid-column: <start-line> / <end-line> | <start-line> / span <value>;
  grid-row: <start-line> / <end-line> | <start-line> / span <value>;
}
```

例子：

```
.item-c {
  grid-column: 3 / span 2;
  grid-row: third-line / 4;
}
```

![grid-start-end-c](http://oznz1caxs.bkt.clouddn.com/grid-start-end-c.png)



##### grid-area: 为项目提供一个名称，以便可以通过使用`grid-template-areas`属性创建的模板来引用它。 另外，这个属性可以用作`grid-row-start` + `grid-column-start` + `grid-row-end` +` grid-column-end`的缩写。值：

- **name**：一个你选择的名字与`grid-template-areas`相匹配
- **row-start / column-start  / row-end /column-end**：可以是数字或者是网格线的名称

```css
.item {
  grid-area: <name> | <row-start> / <column-start> / <row-end> / <column-end>;
}
```

设置名称如下：

```css
.item-d {
  grid-area: header
}
```

作为`grid-row-start` + `grid-column-start` + `grid-row-end` +` grid-column-end`的缩写

```css
.item-d {
  grid-area: 1 / col4-start / last-line / 6
}
```

![grid-start-end-d](http://oznz1caxs.bkt.clouddn.com/grid-start-end-d.png)



##### justify-self：沿着行轴对齐网格内的内容（而不是沿列轴对齐的align-self）。 此值适用于单个网格项目内的内容。值：

- **start**：将内容对齐到网格区域的左端。
- **end**：将内容对齐到网格区域的右端。
- **center**：将内容对齐到网格区域的中间。
- **stretch**：填充网格区域的整个宽度（这是默认的）。

```css
.item {
  justify-self: start | end | center | stretch;
}
```

例子：

```css
.item-a {
  justify-self: start;
}
```

![justify-self-start](http://oznz1caxs.bkt.clouddn.com/grid-justify-self-start.png)

```css
.item-a {
  justify-self: end;
}
```

![grid-justify-self-end](http://oznz1caxs.bkt.clouddn.com/grid-justify-self-end.png)

```css
.item-a {
  justify-self: center;
}
```

![grid-justify-self-center](http://oznz1caxs.bkt.clouddn.com/grid-justify-self-center.png)

```css
.item-a {
  justify-self: stretch;
}
```

![grid-justify-self-stretch](http://oznz1caxs.bkt.clouddn.com/grid-justify-self-stretch.png)

要为网格中的所有项目设置对齐方式，也可以通过`justify-items`属性在网格容器上设置此行为。



##### align-self: 沿着列轴对齐网格内的内容（而不是沿行轴对齐的justify-self）。 此值适用于单个网格项目内的内容。值：

- **start**：将内容对齐到网格区域的顶部。
- **end**：将内容对齐到网格区域的底部。
- **center**：将内容对齐到网格区域的中间。
- **stretch**：填充网格区域的整个高度（这是默认的）。

```css
.item {
  align-self: start | end | center | stretch;
}
```

例子：

```css
.item-a {
  align-self: start;
}
```

![grid-align-self-start](http://oznz1caxs.bkt.clouddn.com/grid-align-self-start.png)

```css
.item-a {
  align-self: end;
}
```

![grid-align-self-end](http://oznz1caxs.bkt.clouddn.com/grid-align-self-end.png)

```css
.item-a {
  align-self: center;
}
```

![grid-align-self-center](http://oznz1caxs.bkt.clouddn.com/grid-align-self-center.png)

```css
.item-a {
  align-self: stretch;
}
```

![grid-align-self-stretch](http://oznz1caxs.bkt.clouddn.com/grid-align-self-stretch.png)

要对齐网格中的所有项目，也可以通过`align-items`属性在网格容器上设置此行为。