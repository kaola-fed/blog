---
title: 记Chrome的性能分析工具实践
date: 2017-12-13
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

### 业务场景

事情的起因是我们WMS系统内有一个批量打印的功能，今天仓库反应第一次打印的速度大概是2s，但是之后每次都越来越慢，到后面页面基本就直接卡死了。

从这个表现来看，这个问题基本可以定位成性能问题，而不是可以被try...catch到的异常。

想到的解决方案有两种：

1. 一行行review这部分相关的代码，console+debugger来逐步排查问题。
2. 使用Chrome自带的性能分析工具来定位问题。

一直没找到什么实践机会来使用这部分功能来定位问题，这次有了真实的业务场景，果断希望能用第二种方案来准确定位到问题。

### 内存泄露？

从表现上来看，感觉像是每次批量打印的时候导致了一部分内存泄漏，然后内存占比越来越高导致浏览器卡死。

这时候就需要用Chrome的Memory功能来统计下内存占比。在Memory下选择*Take heap snapshop*，分析完成之后可以看到当前页面的各种对象的内存占比。当前我关注的不用那么细，只需要关注到页面的总内存，我在每次批量打印完成后都重新统计了页面占的内存，可以发现：每次内存占用有轻微上升，但肯定不会导致页面卡死。至此，可以排除掉是内存泄漏导致的问题。

<img src="http://oksj4hknl.bkt.clouddn.com/memory.jpeg">

### 使用Timeline分析

排除到内存泄漏的问题后，我能想到的页面被卡住的原因是JS脚本的执行时间过长。因为浏览器的渲染是单线程的，如果当前浏览器在进行JS脚本计算，那么在这个过程中UI线程是没法同时进行渲染改变的，所以看起来就是页面卡顿了。

这时候就需要用Chrome的Timeline分析工具（新版本Chrome里改成Performance）来查看每一个函数调用的时长，来定位出问题的具体函数。

点击*record*按钮之后，然后多次调用批量打印功能，结束录制。得到了如图的分析结果：

<img src="http://oksj4hknl.bkt.clouddn.com/timeline1.jpeg">

从图中可以明显看出第一次打印到第三次打印耗时增加很多，其中黄色部分代表脚本执行时间，紫色部分代表渲染时间。增加的时间部分主要是脚本执行时间导致的，这时候需要定位具体是哪个函数导致的脚本执行时间增加。

这时候可以查看火焰图部分，横坐标代表了消耗时间，纵坐标是调用栈关系，上面的栈调用下面的栈。一直从上往下找到最内层的函数调用，发现了导致耗时增加的函数是`JsBarcode`，一个生成条形码的函数。

<img src="http://oksj4hknl.bkt.clouddn.com/jabarcode.jpeg">

为了验证这个结论，在代码里先注释掉这个函数，重新执行Timeline分析。得到如图的分析结果：

<img src="http://oksj4hknl.bkt.clouddn.com/timeline2.jpeg">

可以看到因为脚本执行导致的时间增加问题已经解决了，但是渲染的时间还是随着每一次的打印都会增加，这时候开始分析渲染的问题。查看渲染部分详情可以看到右上角的一个三角形，这个三角形代表这里存在异常，并且Chrome给出了相应的警告（Forced reflow）：

<img src="http://oksj4hknl.bkt.clouddn.com/sanjiaoxing.jpeg">

也就是说这里强制重绘了界面，但是至此还是没理解为什么会重新绘制页面，这里强大的Chrome直接给出了影响渲染的代码片段，scrollbar-width这个文件是element-ui的一个工具函数。

这时候查看element-ui源码对这个文件的引用情况，一层层往上定位发现在当前页面使用到的table组件和message-box组件里都引用到了这个文件。table组件会在列表数据刷新的时候调用到这个函数，而message-box会在弹出的时候调用到这个函数。

为了验证这个猜想，注释掉列表更新和弹框的代码，重新使用Timeline分析，发现刚才的三角形不见了，这时候页面被重新绘制的问题也找到了。

<img src="http://oksj4hknl.bkt.clouddn.com/timeline3.jpeg">

至此，导致增加打印时长的两个问题 1. 脚本执行 2. 页面渲染 都已经被定位到了，但是导致问题的业务代码还没改呢！也就是为什么绘制条形码的函数和重绘页面的时间会越来越长呢？

### 解决问题

问题已经分析到这了，很轻易想到了这部分功能里唯一一句相关的DOM操作代码，在每次打印一张快递单的时候都会appendChild一段DOM到body上（为了绘制二维码以及转canvas导出图片），但是每次appendChild之后并没有去remove掉这段冗余的DOM（逃。

回滚debugger的时候的修改代码，添加removeChild操作。重新执行Timeline分析，可以看到三次打印的耗时已经一致了。明天让QA小哥哥测试一下可以上线了～

<img src="http://oksj4hknl.bkt.clouddn.com/yabi.jpeg">

### 总结

最终定位到错误比较低级，不过这次实践基本是从Chrome强大的性能分析工具定位到了问题，虽然如果采用方案1去review code来排查最终也能定位到问题。但是可以想象，当业务代码足够复杂，函数调用层级很深的时候，去review code来排查的效率就会远不如利用Timeline分析的效率高了。

今天听了云音乐的校园十佳，写了一篇博客，忽然想吟诗一首，苟利国家生死以（逃

### 参考资料
[developers.google.com](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance/timeline-tool?hl=zh-cn)
[全新Chrome Devtool Performance使用指南](https://zhuanlan.zhihu.com/p/29879682)

