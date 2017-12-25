现有Web页面上，table一般用于统计数据、列表书记等信息的展示。但是当table包含大量的信息需要列出展示时，往往对table添加滚动条来展示更多的数据。突然想到要研究下table组件也是因为最近碰到table数据大量展示的问题，寻觅了一些比较好的解决办法，就目前来说，个人已知最好的应该就是[ant-design中的Table表格](https://ant.design/components/table-cn/)。它通过对表格前后列进行固定，中间滚动的交互方式可以很好的满足个人需求，这种交互对于大量表格数据和关键信息的展示非常友好，下面我们就来聊聊包含大量数据和列数的表格展示优化。

<!-- more -->

## 表格功能
首先需要对表格的大致功能有一个大概的了解。Ant-design上对表格的使用说明如下：
> 当有大量结构化的数据需要展现时；当需要对数据进行排序、搜索、分页、自定义操作等复杂行为时。  
通常来说，表格的主要用途就是数据的展示，然后还附带一些其他的功能来满足对数据的整理、筛选等。

## 表格设计
表格一般包括表头、表尾，内容对应具体的行列，每一个数据占据一个单元格，如下图所示。
![表格](https://haitao.nos.netease.com/96f45ebd-829c-4d1e-bf86-6713da11ecc1.png)
根据HTML中表格的标签属性，我们对表格进行拆分：
* 表头
* 表尾
* 行
* 列
* 单元格

对应常用的功能项包括：
- 选择等操作
- 筛选和排序
- 展开数据
- 表格行列合并
- 树形数据
- 固定表头、表尾
- 固定列
- 固定头和列
- 编辑
- …

## 大量数据的表格交互优化
对于包含大量数据和列数的表格，添加滚动条是一种常见的方法，用来提示表格包含更多的内容。但是，这种方法往往不够简洁，并且不容易展示关键的信息。
> 滚动条往往太过笨重，在视觉上喧宾夺主，因此，现代操作系统已经开始简化它的外观，当用户不与可滚动的元素交互时，滚动条就会被隐藏。（《CSS揭秘》34-滚动条提示）

对于大量数据的表格，更好的用户体验优化主要包含以下两点：
1. 更优雅的用户体验模式——以阴影来提示更多的内容
在css揭秘中有提及到：“Google Reader的用户体验设计师找到了一种非常优雅的方式来做出和滚动条提示类似的提示：当侧边栏的容器还有更多的内容时，一层淡淡的阴影会出现在容器的顶部或者底部”。
对于大量数据的表格，我们可以以类似的方式来提示用户表格包含更多的内容，需要滚动。
2. 关键信息展示——固定列&表头
对于表格信息来说，往往有些信息是重要的，需要一目了然的看清楚，而不是通过滚动才能看到用户需要的信息。将关键的表格列进行固定，而次要信息放置在可滚动查看的容器中，可以突出表格信息的重要性，同时，对于方便用户更好的在当前窗口进行交互，而不需要更多的操作。

对于以上两种交互的优化，[Ant Design - A UI Design Language](https://ant.design/components/table-cn/#components-table-demo-fixed-columns-header)给出了很好的示例。
![Ant-design-table](https://haitao.nos.netease.com/f6e3c353-952a-4b5d-a1dd-a248deaa741d.png)

## 优化的实现
对于固定头和列的表格数据交互方式，现有的组件库均有对应的实现，比较常见的对应有：
* React：[Ant Design - A UI Design Language](https://ant.design/components/table-cn/)
* Vue：[Element](http://element.eleme.io/#/zh-CN/component/table)，[iView - A high quality UI Toolkit based on Vue.js](https://www.iviewui.com/)
* Regular: [表格 - nek-ui](http://nek-ui.kaolafed.com/components/layout_KLTable_.html)

那么如何实现上述表格的交互优化呢，综合以上三种组件库对于table组件的实现方式来看，具体包括以下几个方面（以下文中示例代码使用vue实现）。
### 布局
在考虑表头和列进行固定时，我们首先需要考虑的是如何对表格进行布局来同时满足上下和左右的滚动。
![表格滚动示例](https://haitao.nos.netease.com/9cde2bdb-89b0-4cbe-91c9-b6367635a142.png)
如上图所示，蓝色框框部分将表格分为固定的表头和内容，内容区域可以上下滚动；红色框框部分将表格分类固定的列和中间列，中间部分可以左右滚动。从示意图可以发现，两部分有一部分内容交叉。
在布局上，主要实现方式为以下两种：
1. 如果依据左右滚动布局，那么整个表格分为左，中，右三部分，中间overflow左右滚动。那么，如果同时满足表头固定后，body上下滚动，需要js事件监听中间部分的上下滚动，同时对左右表格的body部分进行滚动。见[表格左右滚动布局示例](https://codepen.io/vivijin/pen/MEoaVX)
2. 如果依据上下滚动布局，那么，整个表格将分为两大部分，thead和tbody，内容部分溢出后滚动。那么，如果需要同时满足中间部分进行左右滚动，需要在tbody左右滚动时，同时控制thead的中间进行滚动，需要js监听tbody的scroll事件，对thead进行滚动。见[表格上下滚动布局示例](https://codepen.io/vivijin/pen/RLgWMW)
- - - -
这两种方式均需要通过js监听scroll事件，来满足另一部分的滚动事件，示例的demo比较简陋。对比这两种方式，可以说实现上都差不多，现有组件库的实现均为第一种方式。采用这一类布局的共同点和优势如下：
* 将表头和内容分开为两部分，便于控制表头固定。
* 通过布局满足上下或者左右的滚动，通过js控制另一方向的滚动，相对于完全通过js控制滚动更有优势。
* 需要特别注意的一点是，**中间内容的表格需要包含左右固定的两列，而不是完全拆分**，左右固定的列仅仅是通过固定布局覆盖现有的数据。这样做的优势是保证在中间部分滚动的时候，滚动条属于整个table表格而不是中间部分，给用户视觉上的整个表格滚动的交互体验，这样也保证了一个方向上的完全滚动。
* 左右滚动布局需要在body滚动的同时控制左侧固定列和右侧固定列同时滚动，相对于上下滚动布局仅仅需要控制表头内容左右滚动来说，减少一次计算控制。*本以为布局2在数据量大的情况下，相对布局一发生抖动的可能性更大，但是在5000行数据测试下（仅测试chrome），发现滚动时，两者相差不大。*
- - - -
### 固定行列对齐
由于表格的表头、列固定方法采用的是从table中拆分的方法，固定的表格部分和其余部分可能不一致。同时，表格的内容不固定的情况下，其布局也是比较难以预测的，浏览器往往会根据内容对表格列宽进行调整。

因此，我们在真正使用表格进行数据展示时，由于数据的不可控性，展示出来的效果可能和我们预计的不一样，如下图所示。

::表格行不对齐::
![表格行不对齐](https://haitao.nos.netease.com/da0df198-0eea-4b23-bc45-47de451dbbe8.png)
::表格列不对齐::
![表格列不对齐](https://haitao.nos.netease.com/06cb0d61-55e5-4290-a2e0-6d5a913755c6.png)
对于这两种情况，解决方式如下：
1. 列不对齐，给表格每一列固定宽度，通过定宽的方法来控制表格。
* 设定table的布局方法为table-layout: fixed。
	table-layout的默认值是auto，其行为模式被称作自动表格布局算法。将表格的布局方式设置为fixed不仅有利于页面的快速渲染，同时也让表格更可控。具体的规范见：[table-layout - CSS | MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/table-layout)

> 使用 “fixed” 布局方式时，整个表格可以在其首行被下载后就被解析和渲染。这样对于 “automatic” 自动布局方式来说可以加速渲染，但是其后的单元格内容并不会自适应当前列宽。任何一个包含溢出内容的单元格可以使用 overflow  属性控制是否允许内容溢出。

* 指定表格上每一列的宽度。设置宽度的方法有两种，其一是对每一个th指定width，另一种是使用col属性[<col>](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/col)。
*相对来说，col的方式和对每一个th设置宽度方式比，更优雅一点。我们还可以通过[<colgroup>](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/colgroup)来对表格中的列进行组合，以便对其进行格式化。不过需要注意的一点是，目前的浏览器对于col和colgroup，很多属性已经被废弃了，目前支持度比较全面的仅span和width这两个属性，分别用于规定列组应该横跨的列数和列的宽度。*
* 设定宽度和table-layout后，如果整个表格的宽度大于所有列设定的宽度，且所有列均设置宽度，那么会将多余的宽度按照设置的列宽度比例平均分配。如果只设置了部分列，那么平均分配到剩下未指定宽度的所有列中；如果整个表格宽度小于所有列的设定宽度，且所有列均设置宽度，那么表格会被列撑开。如果只设置了部分列，没有设置宽度的列会被压缩宽度。
2. 行不对齐，实现方式有三种。
* 如果表格中展示的数据不需要完全展示，可以指定表格行的高度，当表格数据溢出时，自动出现省略号来防止表格行左右不对齐的情况。这种方式适用于已知数据的，固定内容的表格数据展示。
* 对左右固定列的两个table，不仅仅包含固定列的数据，并且包含所有表格数据。这样表格行高依据于整个表格一行的数据，并且保持左、中、右数据的完全一致，不会出现左右行高不一致导致行不对齐。这种方式会导致html中拥有较多的冗余数据，不太利于语义化。
* 绘制时，计算一次中间body的表格行高度，根据中间行高来设定左右固定列的表格行高（Ant-design的实现方法）。这种方式减少了冗余数据，但是在初始js渲染时，可能需要更长的时间。

### 左右阴影提示
给table左右固定列均增加一个淡淡的阴影，可以很好的给用户以“这里还有更多数据”的提示，同时，阴影也让固定列的数据更有层次感。
例如通过给左右固定列增加box-shadow，可以增加阴影效果，实现css如下：
```
.table-fixed-left-scroll {
  box-shadow: 6px 0 6px -4px rgba(0,0,0,.2);
}
.table-fixed-right-scroll {
  box-shadow: -6px 0 6px -4px rgba(0,0,0,.2);
}
```
就现有的大部分组件库来说，仅仅做了阴影的实现，当我们滚动时，这条阴影会一直停留在相同的位置。相对来说，更好的交互效果是和Ant-design的table一样，阴影会随着我们的滚动而自动出现，这样给用户的提示也就更加友好，同时可以增加transition动画来让左右阴影出现的更为自然。
![自动阴影提示](https://haitao.nos.netease.com/d3c3e61c-868f-448d-a22a-1963c053b6f4.gif)
**实现方式**
监听body区域scroll事件，通过判断scrollLeft以及整个整个body区域是否被overflow的宽度，来判断左阴影和右阴影。同时需要注意的是：在窗口resize的时候，需要对左右阴影进行重新计算。为了减少scroll事件时整个区域被overflow的宽度的计算（计算clientWidth会增加一次浏览器的重排），在整个窗口onload和resize成功后，需要对窗口进行一次计算，并将计算结果缓存。
```
/* onscroll */
handleBodyScroll() {
      this.scrollValue = this.$refs.tableScroll.scrollLeft;
      this.hasRight = this.scrollValue - this.leftScroll < 0;
      this.hasLeft = this.scrollValue > 0;
},
/* onload & onresize */
setTableShadow() {
      this.leftScroll = this.$refs.tableContent.clientWidth - this.$refs.tableScroll.clientWidth;
      this.handleBodyScroll();
}
```

## 更进一步的优化尝试
对于需要固定表头和表列的table，往往数据量都比较大。我们可以对现有的实现方式做更进一步的优化尝试。

### 使用tranfrom来代替设置scrollLeft
考虑[浏览器重排和重绘](http://blog.csdn.net/q1056843325/article/details/53340308)，一般来说，scrollLeft会引起一次重排，而transfrom仅仅是一次重绘，同时我们可以使用transform3D使用GPU来加速浏览器的渲染。那么，是不是可以考虑在scroll事件中，使用transform来代替scrollLeft的设置呢？
在[React 实现一个漂亮的 Table | HYPERS 前端团队博客](http://blog.hypers.io/2017/09/03/rsuite-table/)这篇文章中，有提及到实现一个优化的table组件使用下面两点来优化onScroll 触发的频率和渲染的速度跟不上造成的抖动问题，他的解决办法如下：
> 用 transform: translate3D 代替 top 与 left ，因为 top/left 会导致回流，而 translate 只产生重绘，性能会更好，另外 translate3D 走的是 3D, 在手机浏览器器上会 GPU 加速。  

> onScroll 触发的频率和渲染的速度会存在跟不上的情况，所有这里最好是自己实现一个滚动条，在 Table Body 上监听 onWheel 事件，在滚动条上监听 onMouse* 事件。 在自己实现滚动条的时候需要注意的是，在 Mac 的 chrome 上，左右滑动的时候会触发浏览器的上一页和下一页功能，所以这里的事件冒泡要处理好（本来想找一个开源的滚动条轮子，发现有好多组件这个问题没有处理好，所以就自己写了）。

为了验证transform和scroll实际效率的对比，使用5000行数据对之前的demo进行了scroll抖动的测试，发现两种布局方式对应的效率提升并没有很明显。

::左右布局-scroll::

![左右布局-scroll](https://haitao.nos.netease.com/611838e5-e7c9-49a5-9178-0e03eb24b6d4.gif)

::左右布局-transform3D::

![左右布局-transform3D](https://haitao.nos.netease.com/be70c43d-5590-4685-bfd1-0d0097fc1af2.gif)

::上下布局-scroll::

![上下布局-scrol](https://haitao.nos.netease.com/bc91429d-6d7a-4cf0-b5ca-a0e1dca820a5.gif)

::上下布局-transform3D::

![上下布局-transform3D](https://haitao.nos.netease.com/43f0c901-0293-4939-955a-f072c2cc12dd.gif)

从以上四个图可以对比看出，tranfrom3D和scroll的性能差距并没有很大。我们还可以通过[jsperf ](https://jsperf.com/translate3d-vs-xy/131) 对两者进行两者的性能分析，同样得到**transfrom3D和scroll对滚动性能的影响并没有相差很大**这个结论。

### 减少scroll事件中的dom操作
本文中的Demo只是简单的示例，为了更好的对比table的scroll性能，通过对iview和element两个组件库的table均进行了5000行数据下table滚动的测试，发现两者均有一定的抖动（都有做防抖动处理，不够明显）。
::iview-table-scroll抖动::

![iview-table-scroll抖动](https://haitao.nos.netease.com/09403a82-f3be-41ba-b388-5438c911e1c5.gif)

::element-table-scroll抖动::

![element-table-scroll抖动](https://haitao.nos.netease.com/c5458508-786d-4d23-92f9-258ffc67f3aa.gif)

查看两个组件库的源码，发现两者均在scroll事件中执行了较多的逻辑判断，例如iview在scroll中进行了column是否存在的判断，时，每次渲染都对操作那一列进行了重新渲染计算，element在scroll时，对左右和上下的滚动都进行了判断计算，这些也都影响滚动的性能。由于onscroll的高频触发，**尽量减少scroll事件中对dom的计算和操作**，减少浏览器重排和重绘。

### 优化滚动事件

> 对于onscroll，onresize等这一类高频触发的事件，如果事件中涉及到大量的位置计算、DOM 操作、元素重绘等工作且这些工作无法在下一个 scroll 事件触发前完成，就会造成浏览器掉帧。加之用户鼠标滚动往往是连续的，就会持续触发 scroll 事件导致掉帧扩大、浏览器 CPU 使用率增加、用户体验受到影响。 

对用户来说，平滑的滚动往往能带来很好的交互效果，优化滚动事件往往有以下三种方法：
* 防抖（Debouncing）：把多个操作合并为一个，即在一定时间内，规定事件出发的次数。
* 节流（Throttling）：只允许操作在一定时间内执行一次，只有当上一次操作执行后过了你规定的时间间隔，才能进行下一次。这种方法往往运用于图片懒加载的的优化。
* 使用 rAF（requestAnimationFrame）触发滚动事件：在页面重绘之前，通知浏览器进行操作。
一般来说，现有的table组件中，一般采用第一种方式来对scroll事件进行处理，通过约束一定时间内发生的次数，来因为用户过快操作导致的scroll请求。

### 使用纯CSS实现阴影自动提示
在《CSS揭秘》一书的第34小节*滚动提示*中（具体文章可见：[Pure CSS scrolling shadows with background-attachment: local | Lea Verou](http://lea.verou.me/2012/04/background-attachment-local/)），对这种方法进行详细的阐述，其原理实现是通过设置两层背景来得到阴影，或者通过设置伪元素和定位来实现，方法实现如下：
* 两层背景：play.csssecrets.io/scrolling-hints
* 伪元素和定位：[Scrolling shadows](http://kizu.ru/en/fun/shadowscroll/)
对于table组件，实现css如下：
```
.tablebackground {
  background: linear-gradient(white 15px, hsla(0,0%,100%,1)) 100px 0 / 15px 300px,
              radial-gradient(at left, rgba(0,0,0,.2), transparent 80%) 100px 0 / 10px 200px,
              linear-gradient(to left, white 15px, hsla(0,0%,100%,1)) right / 110px 300px,
              radial-gradient(at right, rgba(0,0,0,.2), transparent 70%) 390px / 15px 300px;
  background-repeat: no-repeat;
  background-attachment: local, scroll, local, scroll;
}
```
**实现原理**
通过[background-attachment](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-attachment)这个属性，它包含fixed、local和scroll三种取值。
> fixed：此关键字表示背景相对于视口固定。即使一个元素拥有滚动机制，背景也不会随着元素的内容滚动。  

> local：此关键字表示背景相对于元素的内容固定。如果一个元素拥有滚动机制，背景将会随着元素的内容滚动， 并且背景的绘制区域和定位区域是相对于可滚动的区域而不是包含他们的边框。  

> scroll：此关键字表示背景相对于元素本身固定， 而不是随着它的内容滚动（对元素边框是有效的）。

具体实现为：通过构建两层背景，一层用于生成阴影，使用scroll属性，默认和内容保持在原位；另一层为一个用来遮挡阴影的白色矩形，使用local属性，相对于可滚动区域位置固定，这样它就会在滚动在最左侧或者右侧时盖住阴影，滚动时跟着滚动，露出阴影。
同时，使用线性渐变的遮罩层可以让滚动时阴影出现的更为平滑，具体的实现效果请看demo[纯css实现table阴影自动提示](https://codepen.io/vivijin/pen/RLgoXm)。

这种纯CSS的实现很好的避免了通过js监听事件来对dom节点进行操作，更好的优化了性能，但是这种写法依然存在一定的问题：
1. 我们在对table设定遮罩层时，是从左边固定的位置开始的，如果中间的table内容为整个table的宽度，那么需要指定阴影开始的位置，即需要知道左边固定列和右边固定列的宽度，比较适用于固定列宽度已知的table。
2. 可以考虑将中间的窗口大小仅仅设置为非固定列的table部分，这样不需要计算固定列的宽度，但是这样出来的浏览器自带滚动条不能涵盖整个table。

## 总结
对于table表头和列固定，在实现时需要注意以下几点：
1. 存在表头和固定列时，需要对table的布局进行重组，一般固定一个方向进行overflow，另一方向通过监听onscroll事件来同步滚动。
2. 需要固定表头和列时，特别注意table的每一列的宽度和每一行的高度，可以使用一些方法来减少因为宽窄屏幕和数据的不确定性导致的错位。
3. 增加左右固定列的自动阴影提示可以给用户更好的交互体验。实现方式为通过监听scroll事件，使用js来实现，或者通过纯CSS设置背景图来实现。相对来说纯CSS实现减少了dom操作，不过需要结合table的布局方式注意下背景阴影的位置。
4. 尽量减少scroll事件内对dom节点的操作，简化 scroll 内的操作。
5. 采用一定的防抖动或者节流等方法来更好的平滑滚动效果，提高页面性能。
6. 固定表头和列一般用于大数据的table展示，在写常用table组件时，需要注意table的通用的使用场景，不能因为考虑一些极端情况而因小失大，更多的考虑兼容表头和列的固定即可。

## 参考资料
- [Ant-desing](https://ant.design/index-cn)
- [iView](https://www.iviewui.com/)
- [Element](http://element.eleme.io/#/zh-CN)
- [React实现一个漂亮的Table](http://blog.hypers.io/2017/09/03/rsuite-table/)
- [Pure CSS scrolling shadows with background-attachment: local | Lea Verou](http://lea.verou.me/2012/04/background-attachment-local/)
- [How to make faster scroll effects?](https://gist.github.com/Warry/4254579)
- [高性能滚动 scroll 及页面渲染优化](http://blog.csdn.net/lankecms/article/details/51494030)
- [实例解析防抖动（Debouncing）和节流阀（Throttling）](http://www.open-open.com/lib/view/open1463886809124.html)

by [邓瑾](http://www.kaola.com)