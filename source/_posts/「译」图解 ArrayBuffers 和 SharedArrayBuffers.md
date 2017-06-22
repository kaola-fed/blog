---
title:      「译」图解 ArrayBuffers 和 SharedArrayBuffers
date:       2017-06-21
---

翻译自：[A cartoon intro to ArrayBuffers and SharedArrayBuffers](https://hacks.mozilla.org/2017/06/a-cartoon-intro-to-arraybuffers-and-sharedarraybuffers/)

这是图解 SharedArrayBuffers 系列的第二篇：
 1. [内存管理碰撞课程][1]
 2. 图解 ArrayBuffers 和 SharedArrayBuffers
 3. [用 Atomics 避免 SharedArrayBuffers 竞争条件][2]

<!-- more -->

[上一篇文章][3]中，我解释了 JavaScript 这类自动管理内存的语言是如何处理内存的，同样也解释了类似 C 语言这种手动管理内存的语言

为什么这对于我们讨论的 [ArrayBuffers][4] 和 [SharedArrayBuffers][5] 如此重要？

因为即使你使用的是 JavaScript 这种自动管理内存的语言，ArrayBuffers 也提供了一种手动处理数据的途径

为什么你会有这样的需求呢？

正如[上篇文章][6]所说，这里有个权衡，自动管理内存对开发者是友好的，但是这会增加机器负担，甚至会有性能问题

![][7]

例如，JS 里创建一个变量，引擎会去猜测变量的类型以及内存里如何表示。因为有了类型猜测，JS 引擎通常会比真实需要预留更多的空间。根据变量不同，内存分配可能会是真实需求的 2-8 倍，这导致了内存浪费

而且，某些创建和使用 JS 对象的场景会让垃圾回收变得很困难。如果你是手动维护的内存，可以根据实际使用需求来决定分配和释放内存的策略

很多时候，这不是什么大不了的事。大多数场景并不会对性能要求那么苛刻，反而更多地担心管理内存的麻烦。而且一般情况下，手动管理内存可能更慢

但是对于底层需要极致优化的场景，ArrayBuffers 和 SharedArrayBuffers 为你提供了可能

![][8]

## ArrayBuffer 是如何工作的

ArrayBuffer 跟其它 JavaScript 数组差不多，但是不是所有 JavaScript 类型都可以放进去，比如对象、字符串。你唯一可以放进去的只有字节（可以用数字表示）

![][9]

需要澄清的一点是，你事实上不是直接把这个字节到 ArrayBuffer 里就行了，ArrayBuffer 并不知道字节有多长，该用多少位去存

ArrayBuffer 仅仅是一个个 0/1 组成的串，它不知道第一个元素和第二个元素的分割点

![][10]

为了提供必要的上下文信息，把 ArrayBuffer 分块，我们需要把它包裹到视图里，这些数据的视图可以通过带类型的数组添加，已经支持很多种类型的数组了

例如，你可以用一个 Int8 类型的数组把 0/1 串分割成 8 位一组的序列

![][11]

或者你可以用一个无符的 Int16 类型数组，把它分割成 16 位一组的序列，可以把它当作无符整型处理

![][12]

甚至你可以在同一个基础 buffer 上同时处理多种视图，不同视图在相同操作下会返回不同的结果

例如，如果我们从某个 ArrayBuffer 的 Int8 视图得到第 0 和第 1 个元素的值，在 Uint16 视图下，第 0 个元素与其有相同二进制位值，但是得到的值也会不一样

![][13]

这种方式下，ArrayBuffer 几乎是扮演原始内存角色了，它模拟内存的各种跟 C 语言里类似的操作

你可能纳闷了，为什么不让开发者直接操纵内存而是采用这个抽象层。因为直接操作内存会有安全风险，这个以后的文章会讲

## 什么是 SharedArrayBuffer

为了说明白 SharedArrayBuffers，我需要稍微解释下并行运行代码和 JavaScript 的关系

为了更快运行代码或者更更快响应用户事件，你可能会让代码并行运行，为了做到这点，你需要分割工作

一个典型的应用中，所有的工作都由一个单独的主线程处理，这点我之前提到过……这个主线程就像一个全栈工程师，掌管着 JavaScript、DOM 和 视图

任何能够从主线程负载减少工作的方法都对代码运行效率有帮助，某些情况下，ArrayBuffers 可以减少大量应该由主线程做的工作

![][14]

但是也有些时候减少主线程负载是远远不够的，有时你需要增援，你需要分割你的任务

大多数语言里，这种分割工作的方法可以使用多线程实现，这就像很多人同时在一个项目里工作。如果你可以完美地把任务分割为多个独立的部分，你可以分给不同的线程，然后，这些线程就同时各种独立执行这些任务

在 JavaScript 里，你可以借助 web worker 做这种事，这些 [web workers][15] 跟其它语言的线程还是有些区别的，默认它们不能共享内存

![][16]

这意味着如果你想分配你的任务给别的线程，你需要完整把任务复制过去，这可以通过 [postMessage][17] 实现

postMessage 把你传给它的任何对象都序列化，发送到其它 web worker，然后那边接收后反序列化并放进内存

![][18]

这个过程是非常慢的

某些类型数据（如 ArrayBuffers）你可以通过移动内存的方式实现，这意味着把某个特定区域的内存移过去后其它 web worker 就可以直接访问了

但是，之前的 web worker 就无法访问了

![][19]

对于某些场景这是实用的，但是也有很多场景对性能要求高，你只能使用共享的内存

而这就是 SharedArrayBuffers 为你提供的

![][20]

有了 SharedArrayBuffer 后，多个 web worker 就可以同时读写同一块内存了

你再也不需要 postMessage 伴有时延的通信了，多个 web worker 对数据访问都没有时延了

当然，这种同时访问也有风险，会产生竞争条件

![][21]

这个下一篇文章会细说

## SharedArrayBuffers 支持情况

所有主流浏览器都将会支持 SharedArrayBuffers

![][22]

Safari 10.1 已经支持了，Firefox 和 Chrome 也会很快支持并发布，Edge 会在他们秋季 Windows 更新的时候发布

即使所有主流浏览器都支持了，我们也不希望开发者直接使用它们，事实上，我们是反对的。你应该只使用更高级的封装好的抽象层接口

我们期盼的是 JavaScript 库开发者可以提供更简单安全的方法来使用 SharedArrayBuffers

而且，一旦 SharedArrayBuffers 内置到平台中，WebAssembly 可以通过它实现多线程，到那时候你就可以使用类似 Rust 的多线程语言轻松玩转多线程了

[下一篇文章][23]我们会介绍一个为避免竞争条件的库提供基础操作的工具（[Atomics][24]）

![][25]


By [Cody](https://github.com/int64ago)


  [1]: https://github.com/kaola-fed/blog/issues/70
  [2]: https://github.com/kaola-fed/blog/issues/72
  [3]: https://github.com/kaola-fed/blog/issues/70
  [4]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
  [5]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer
  [6]: https://github.com/kaola-fed/blog/issues/70
  [7]: https://cdn.int64ago.org/egh86ar2.png
  [8]: https://cdn.int64ago.org/howd3ymn.png
  [9]: https://cdn.int64ago.org/aki3zrxf.png
  [10]: https://cdn.int64ago.org/u34s6ku.png
  [11]: https://cdn.int64ago.org/eeju5fw6.png
  [12]: https://cdn.int64ago.org/h56rhd5j.png
  [13]: https://cdn.int64ago.org/llwasvia.png
  [14]: https://cdn.int64ago.org/qkodaaqi.png
  [15]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers
  [16]: https://cdn.int64ago.org/e68b0vad.png
  [17]: https://developer.mozilla.org/en-US/docs/Web/API/Worker/postMessage
  [18]: https://cdn.int64ago.org/blehepo.png
  [19]: https://cdn.int64ago.org/ngu33a1b.png
  [20]: https://cdn.int64ago.org/avi9ufo9.png
  [21]: https://cdn.int64ago.org/bp8oj22n.png
  [22]: https://cdn.int64ago.org/qx8jhl1d.png
  [23]: https://github.com/kaola-fed/blog/issues/72
  [24]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics
  [25]: https://cdn.int64ago.org/l5qu7huq.png