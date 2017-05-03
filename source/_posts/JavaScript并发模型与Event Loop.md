---
title: JavaScript并发模型与Event Loop
date: 2017-04-21
---

## 并发模型可视化描述
![model.svg](https://mdn.mozillademos.org/files/4617/default.svg)

如上图所示，Javascript执行引擎的主线程运行的时候，产生堆（heap）和栈（stack），程序中代码依次进入栈中等待执行，若执行时遇到异步方法，该异步方法会被添加到用于回调的队列（queue）中【即JavaScript执行引擎的主线程拥有一个执行栈/堆和一个任务队列】。

<!-- more -->

> 栈（stack） : 函数调用会形成了一个堆栈帧
> 堆（heap） : 对象被分配在一个堆中，一个用以表示一个内存中大的未被组织的区域。
> 队列（queue） ： 一个 JavaScript 运行时包含了一个待处理的消息队列。每一个消息都与一个函数相关联。当栈为空时，则从队列中取出一个消息进行处理。这个处理过程包含了调用与这个消息相关联的函数（以及因而创建了一个初始堆栈帧）。当栈再次为空的时候，也就意味着该消息处理结束。 

### 为了更清晰地描述Event Loop，参考下图的描述：
![model.png](https://cdn.int64ago.org/6p10znqn.png)

**首先，我们对图中的一些名词稍加解释**：

1.  queue : 如上文的解释，值得注意的是，除了IO设备的事件(如load)会被添加到queue中，用户操作产生 的事件（如click,touchmove）同样也会被添加到queue中。队列中的这些事件会在主线程的执行栈被清空时被依次读取（队列先进先出，即先被压入队列中的事件会被先执行）。
2. callback : 被主线程挂起来的代码，等主线程执行队列中的事件时，事件对应的callback代码就会被执行

【注：因为主线程从"任务队列"中读取事件的过程是循环不断的，因此这种运行机制又称为Event Loop（事件循环）】

**下面我们通过setTimeout来看看单线程的JavaScript执行引擎是如何来执行该方法的。**
1. JavaScript执行引擎主线程运行，产生heap和stack
2. 从上往下执行同步代码,log(1)被压入执行栈，因为log是webkit内核支持的普通方法而非WebAPIs的方法，因此立即出栈被引擎执行，输出1
3. JavaScript执行引擎继续往下，遇到setTimeout()t异步方法（如图，setTimeout属于WebAPIs），将setTimeout(callback,5000)添加到执行栈
4. 因为setTimeout()属于WebAPIs中的方法，JavaScript执行引擎在将setTimeout()出栈执行时，注册setTimeout()延时方法交由浏览器内核其他模块（以webkit为例，是webcore模块）处理
5. 继续运行setTimeout()下面的log(3)代码，原理同步骤2
6. 当延时方法到达触发条件，即到达设置的延时时间时（5秒后），**该延时方法就会被添加至任务队列里**。这一过程由浏览器内核其他模块处理，与执行引擎主线程**独立**
7. JavaScript执行引擎在主线程方法执行完毕，到达空闲状态时，会从任务队列中顺序获取任务来执行。
8. 将队列的第一个回调函数重新压入执行栈，执行回调函数中的代码log(2)，原理同步骤2，回调函数的代码执行完毕，清空执行栈
9. JavaScript执行引擎继续轮循队列，直到队列为空
10. 执行完毕

```javascript
    console.log(1);
    setTimeout(function() {
        console.log(2);
    },5000);
    console.log(3);

    //输出结果：
    //1
    //3
    //2
```

## Macrotasks 和 Microtasks

基本上，一个完整的事件循环模型就讲完了。现在我们来重点关注一下队列。
异步任务分为两种：Macrotasks 和 Microtasks。

 - Macrotasks: setTimeout, setInterval, setImmediate, I/O, UI rendering
 - Microtasks: process.nextTick, Promises, Object.observe(废弃), MutationObserver

Macrotasks 和 Microtasks有什么区别呢？我们以setTimeout和Promises来举例。

```javascript
    console.log('1');
    setTimeout(function() {
      console.log('2');
    }, 0);
    Promise.resolve().then(function() {
      console.log('3');
    }).then(function() {
      console.log('4');
    });
    console.log('5');
    //输出结果：
    //1
    //5
    //3
    //4
    //2
```
原因是Promise中的then方法的函数会被推入 microtasks 队列，而setTimeout的任务会被推入 macrotasks 队列。在每一次事件循环中，macrotask 只会提取一个执行，而 microtask 会一直提取，直到 microtasks 队列清空。
结论如下：
1. microtask会优先macrotask执行
2. microtasks会被循环提取到执行引擎主线程的执行栈，直到microtasks任务队列清空，才会执行macrotask

【注：一般情况下，macrotask queues 我们会直接称为 task queues，只有 microtask queues 才会特别指明。】

## 【参考链接】
[JavaScript 运行机制详解：再谈Event Loop][1]
[并发模型与Event Loop][2]
[【转向Javascript系列】从setTimeout说事件循环模型][3]
[异步 JavaScript 之理解 macrotask 和 microtask][4] 


[1]: http://www.ruanyifeng.com/blog/2014/10/event-loop.html
[2]: https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/EventLoop
[3]: http://www.alloyteam.com/2015/10/turning-to-javascript-series-from-settimeout-said-the-event-loop-model/
[4]: https://blog.keifergu.me/2017/03/23/difference-between-javascript-macrotask-and-microtask/