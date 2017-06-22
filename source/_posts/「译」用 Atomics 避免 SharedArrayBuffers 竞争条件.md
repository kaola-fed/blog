---
title:      「译」用 Atomics 避免 SharedArrayBuffers 竞争条件
date:       2017-06-21
---

翻译自：[Avoiding race conditions in SharedArrayBuffers with Atomics](https://hacks.mozilla.org/2017/06/avoiding-race-conditions-in-sharedarraybuffers-with-atomics/)

这是图解 SharedArrayBuffers 系列的第三篇：
 1. [内存管理碰撞课程][1]
 2. [图解 ArrayBuffers 和 SharedArrayBuffers][2]
 3. 用 Atomics 避免 SharedArrayBuffers 竞争条件

<!-- more -->

> 译者注：文中会多次出现“线程（threads）”，这个翻译其实并不准确，但不会妨碍理解

[上篇文章][3]我介绍了什么情况下使用 SharedArrayBuffers 会导致竞争条件，这让使用 SharedArrayBuffers 变得很困难，我们并不希望应用开发者直接就这么使用 SharedArrayBuffers

但是在多线程编程方面经验丰富的库开发者可以使用这些底层 API 创造出高级的工具，应用开发者可以直接使用这些工具而不用去直接接触 SharedArrayBuffers 和 Atomics

![][4]

即使你工作中不需要直接接触 SharedArrayBuffers 和 Atomics，我觉得去理解它的工作原理也是很有意思的。因此，在这篇文章里我会解释下哪些竞争条件会产生，以及 Atomics 是如何解决这些问题的

但是，首先，什么是竞争条件呢？

![][5]

## 竞争条件：之前看过的例子

如果有两个线程使用同一个变量，那么就有可能产生竞争条件，这是最简单的情况。再具体点，假设一个线程要加载一个文件，而另一个线程要检查这个文件是否存在（译者注：这里应该是检查并设置存在标志位），它们会使用到同一个变量 fileExists 去通信

初始的时候，fileExists 被设置为 false

![][6]

一旦线程 2 先运行，文件就会被加载

![][7]

但是如果线程 1 先运行，就会向用户抛一个错误，说文件不存在

![][8]

但是这不是问题的关键，文件存在与否问题不大，真正的问题在于竞争条件

即使在单线程代码里，许多 JavaScript 开发者也会遇到这类竞争条件，你不需要理解多线程就能搞明白为什么会竞争

然而，有些竞争条件在单线程里就没法发生，只可能在有内存共享的多线程里发生

## 不同类型的竞争条件以及 Atomics 是如何解决的

现在说点多线程里不同类型的竞争条件，看看如何用 Atomics 解决的。这个并没有覆盖所有情况，但是却会给你提供一些思路去理解为什么 Atomics 的 API 会提供这些方法

开始之前，需要再次重申：你不应该直接使用 Atomics！写多线的代码本来就是个很苦难的事情，你应该直接使用可靠的库去处理多线程中共享内存问题

![][9]

## 单个运算的竞争条件

假设有两个线程同时增加某个变量的值，你可能认为，无论哪个线程先运行，最终的结果是一样的

![][10]

在代码里，即使增加一个变量这种操作看起来像是一个操作，但如果看到编译后的代码，会发现并不是

从 CPU 层面看，增加一个变量值需要三条指令，这是因为计算机同时有长期存储器和短期存储器（这个在其它文章里会说）

![][11]

所有的线程共享同一个长期存储器（内存），但是短期存储器（寄存器）并不是共享的

每个线程需要把值先从内存搬到寄存器，之后就可以在寄存器上进行计算了，再然后会把计算后的值写回内存

![][12]

如果线程 1 的所有的操作都先执行，之后执行所有线程 2 的操作，最终会得到我们的预期的结果

![][13]

但是，如果它们间隔着执行，从线程 2 的里移到寄存器的值就无法与内存的值同步了，这意味着线程 2 会无法用到线程 1 的计算结果。相反，它线程 2 会用覆盖掉线程 1 写回内存的值

![][14]

原子操作做的一件事就是在多线程中让计算机按照人所想的单操作方式工作

这就是为什么被叫做原子操作，因为它可以让一个包含多条指令（指令可以暂停和恢复）的操作执行起来像是一下子就完了，就好像一条指令，类似一个不可分割的原子

![][15]

使用原子操作会让加法变得有点不一样

![][16]

现在，我们可以使用 `Atomics.add` 了，加法执行过程中不会因为多线程而被打乱。一个线程在执行完原子操作前会阻止其它线程执行，之后其它线程才会执行自己的原子操作

![][17]

Atomics 中帮助避免竞争的方法有：

 - [Atomics.add][18]
 - [Atomics.sub][19]
 - [Atomics.and][20]
 - [Atomics.or][21]
 - [Atomics.xor][22]
 - [Atomics.exchange][23]

你会发现这个列表数量很有限，甚至没有除法和乘法。不过，库的开发者会提供类似这些常见原子操作的

库的开发者会借助 [Atomics.compareExchange][24] 从 SharedArrayBuffer 拿到值，应用相应的操作，然后只有在自上次检查到现在没有其它线程更新的情况下才会去写回。如果期间有其它线程更新了，则会先拿到新的值重新运算一次

## 多运算的竞争条件

这些 Atomic 运算符成功避免了“单运算”中的竞争条件。但是，有时你会同时改变一个对象上的多个值（使用多个运算），在此期间，你并不希望有其它的任务也在修改这个对象。简单说，就是在你修改这个对象期间，这个对象是处于禁闭状态，其它线程不可以访问

Atomics 没有提供任何方法去做这个事，但是却为库开发者提供了相应的方案，库开发者可以通过锁来达到目的

![][25]

如果代码想使用某个被锁住的数据，首先它需要去请求锁，之后它会用这个锁把其它线程锁在外面，只有它可以访问和更新这块数据

库开发者会通过使用 [Atomics.wait][26] 和 [Atomics.wake][27]，以及可选的 [Atomics.compareExchange][28] 和 [Atomics.store][29] 创建一个锁。想了解更多可以看下这篇文章 [简单锁的实现][30]

这种情况下，线程 2 会请求到锁，并把值设置为 true，这意味着直到线程 2 交出锁前，线程 1 是无法访问的

![][31]

如果线程 1 想要访问这块数据，它会试图请求锁。但是因为锁处于被使用状态，它无法拿到，它于是只能出于等待状态直到锁可用

![][32]

一旦线程 2 结束了，它会调用 unlock，锁会通知其它等待的线程自己空出来啦

![][33]

那个线程就会拿起锁，锁住数据供自己使用

![][34]

实现一个锁可能需要依赖很多 Atomics 的方法，但是用的最多的是下面两个：

 - [Atomics.wait][35]
 - [Atomics.wake][36]

## 指令重排导致的竞争条件

这里还有第三种同步问题需要用 Atomics 处理，这类问题可能会很神奇

你可能感觉不到，你写的代码很可能根本没按你期望的顺序执行，因为编译器和 CPU 会尝试重排指令使得代码更快地运行

比如，你写了一些代码去计算总和，你想的是计算完了要设置一个标记

![][37]

编译的时候需要决定每个变量该用哪个寄存器，之后就可以把代码翻译成机器的指令了

![][38]

目前为止，一切都在掌握中

如果你对计算机芯片级的原理不理解的话，可能你没发现到第 2 行需要等待下才能执行

大多数的计算机会把一个指令拆分为多个步骤，这使得 CPU 可以被充分利用

下面是一个指令执行步骤的例子：

1. 从内存里拿到下一个指令
2. 指令解码，从寄存器拿值
3. 执行指令
4. 结果写回寄存器

![][39]
![][40]
![][41]
![][42]

这就是指令如何像流水线工人一样工作，理想的情况是第二个指令会紧紧地跟着第一个指令，当第一个指令进行到步骤 2 的时候，第二个指令进行步骤 1

问题是，指令 1 和指令 2 存在依赖

![][43]

CPU 需要一直等待直到指令 1 更新了寄存器里的 `subTotal`，但是这就使执行变慢了

为了让这一切更加高效，很多编译器和 CPU 会记录好代码，找到不依赖 `subTotal` 或 `total` 的指令，然后移到两个指令之间

![][44]

这会让指令执行保持着一个很稳定的流水线

因为第三行不依赖任何前两行的值，编译器和 CPU 认为它是安全的。在单线程里运行时，直到运行完不会有其它代码看到这些

但是当有另一个 CPU 上的线程也在同时运行，情况就不妙了。其它线程不需要一直等到函数执行完毕，只要值写到内存里它就可以看到，因此，它会认为 `isDone` 是在 `total` 前设置的

如果你用 `isDone` 作为 `total` 被计算好用于其它线程的标记，这里就会产生竞争条件

Atomics 试图去解决这些问题，使用 Atomic 的时候就像在代码块上加了个围栏

Atomic 操作相互之间不会重排，其它操作也不会移动到它们的周围。其中，有两个经常用到的操作：

 - [Atomics.load][45]
 - [Atomics.store][46]

`Atomics.store` 之前的代码可以保证在 `Atomics.store` 之前运行完并把值写回内存。即使非原子指令相互之间重排了，也不会移到 `Atomics.store` 的下面

所有 `Atomics.load` 后面的变量可以保证只会在 `Atomics.load` 后面取得值。即使非原子指令重排了，也不会有指令会移到 `Atomics.load` 上面

![][47]

*提示：这里我写的一个 while 循环使用了自旋锁，很低效。如果它在主线程上运行的话，会让你的应用程序有无响应一段时间，你不应该在实际代码里用*

再次提醒，这些方法不建议直接在应用程序里使用，库开发者会用这些创造锁供使用

## 总结

有内存共享的多线程编程是很困难的，有太多竞争条件的陷进等着你往里跳

![][48]

这就是为什么你不会喜欢直接在应用程序里使用 SharedArrayBuffers 和 Atomics。相反，你应该使用一个由多线程方面经验丰富的开发者开发的可靠的库，他肯定会内存模型研究很透彻

SharedArrayBuffer 和 Atomics 才出来没多久，这样的库还没有呢，但是新的 API 已经足够去构建这些

By [Cody](https://github.com/int64ago)

  [1]: https://github.com/kaola-fed/blog/issues/70
  [2]: https://github.com/kaola-fed/blog/issues/71
  [3]: https://github.com/kaola-fed/blog/issues/71
  [4]: https://cdn.int64ago.org/1wv03eqe.png
  [5]: https://cdn.int64ago.org/d6jew52f.png
  [6]: https://cdn.int64ago.org/vpxkwldp.png
  [7]: https://cdn.int64ago.org/6a6p3wh.png
  [8]: https://cdn.int64ago.org/xfs6t7vm.png
  [9]: https://cdn.int64ago.org/1f9e9i2a.png
  [10]: https://cdn.int64ago.org/l7cfs46e.png
  [11]: https://cdn.int64ago.org/xcawm0s.png
  [12]: https://cdn.int64ago.org/bz6bbjju2.png
  [13]: https://cdn.int64ago.org/38lgf34p.png
  [14]: https://cdn.int64ago.org/r578i5n4.png
  [15]: https://cdn.int64ago.org/0zyrbyww.png
  [16]: https://cdn.int64ago.org/9p697sac.png
  [17]: https://cdn.int64ago.org/5e3o7ki9.png
  [18]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/add
  [19]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/sub
  [20]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/and
  [21]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/or
  [22]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/xor
  [23]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/exchange
  [24]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/compareExchange
  [25]: https://cdn.int64ago.org/jwatjmek.png
  [26]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait
  [27]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wake
  [28]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/compareExchange
  [29]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/store
  [30]: https://github.com/lars-t-hansen/js-lock-and-condition
  [31]: https://cdn.int64ago.org/kw8xhrw.png
  [32]: https://cdn.int64ago.org/ats5rtul.png
  [33]: https://cdn.int64ago.org/7r17txa.png
  [34]: https://cdn.int64ago.org/8cxk52wm.png
  [35]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait
  [36]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wake
  [37]: https://cdn.int64ago.org/hyaeeyq7.png
  [38]: https://cdn.int64ago.org/wc8dgia1.png
  [39]: https://cdn.int64ago.org/wem2vu9c.png
  [40]: https://cdn.int64ago.org/0ga09ftf.png
  [41]: https://cdn.int64ago.org/9ghuenzb.png
  [42]: https://cdn.int64ago.org/igxc4v0l.png
  [43]: https://cdn.int64ago.org/3ooo5g4a.png
  [44]: https://cdn.int64ago.org/g0b68hbr.png
  [45]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/load
  [46]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/store
  [47]: https://cdn.int64ago.org/1krxycg.png
  [48]: https://cdn.int64ago.org/at3pxv4c.png