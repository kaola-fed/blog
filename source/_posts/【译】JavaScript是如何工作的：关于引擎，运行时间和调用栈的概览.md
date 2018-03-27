# JavaScript是如何工作的：关于引擎，运行时间和调用栈的概览

随着JavaScript越来越流行，很多团队在他们的前端，后端，混合应用和嵌
入式的技术栈中都扩展或正在扩展JavaScript。

这篇文章是*深入理解JavaScript及其工作原理*系列的第一篇，我们认为通过理解JavaScript构建模块以及其如何工作的将会帮助你写出更好的代码和应用。我们也会分享一些关于创建*SessionStack*时的经验，一个轻量级，强壮高性能且时刻保持竞争性的JavaScript应用。

正如[GitHut stats](http://githut.info/)展示的那样,JavaScript是目前最活跃，最多push的语言。
![Alt text](https://cdn-images-1.medium.com/max/1600/1*Zf4reZZJ9DCKsXf5CSXghg.png "title")

如果项目目前非常依赖JavaScript，这意味着开发者必须深入理解JavaScript的内部原理然后才能使用JavaScript去构建好的软件。

## 概览
几乎所有人都听过V8引擎这个概念，并且大多数人都知道JavaScript是单线程或者它使用的是回调队列。

在这篇文章里，我们会穿过那些概念并且解释JavaScript是如何运行的。通过知道这些细节，你将会更好的使用那些接口写出更棒的代码。

假如你是JavaScript新人，这边文章将会帮助你理解为什么JavaScript跟其他语言比较起来这么的“奇怪”。

如果你是一个有经验的JavaScript开发者了，那么希望这篇文章在JavaScript是如何运行这个方面能给你带来一点点新的思考。

## JavaScript引擎
现在很流行的一个JavaScript引擎就是Google的V8引擎，他被用在Chrome和Node.js里面。这里有一张非常简单的概览图：
![Alt text](https://cdn-images-1.medium.com/max/1200/1*OnH_DlbNAPvB9KLxUCyMsA.png "title")
引擎由两个主要组件构成:

- Memory Heap(内存堆) —— 内存分配的地方
- Call Stack(调用栈) —— 代码执行的地方

## JavaScript运行时间
很多开发者都在浏览器环境中使用过一些api（例如：setTimeout）。然而这些apis并不是JavaScript引擎提供的。

那么。他们从哪来的呢？

真实情况可能有点复杂。

![Alt text](https://cdn-images-1.medium.com/max/800/1*4lHHyfEhVB0LnQ3HlhSs8g.png "title")

这里除了引擎之外还有更多，例如由浏览器提供的Web APIs，就像DOM，AJAX，setTimeout等等。

然后，我们还有如此流行的 **event loop** 和 **callback queue**

## The Call Stack（调用栈）
JavaScript是一个单线程的编程语言，这意味着它有一个单独的调用栈，因此它只能在同一时间做一件事。

调用栈是一种描述我们程序进程的数据结构，如果我们进入了一个function，我们就把这个function放在stack的顶部，如果我们从一个function返回了，那么我们就删除stack的第一个stack frame。这就是stack做的事情。

让我们看个例子，请看下列代码：
``` 
function multiply(x, y) {
    return x * y;
}
function printSquare(x) {
    var s = multiply(x, x);
    console.log(s);
}
printSquare(5);
```
当引擎开始执行这段代码，Call Stack将会清空，并且开始如下步骤：
![Alt text](https://cdn-images-1.medium.com/max/800/1*Yp1KOt_UJ47HChmS9y7KXw.png "title")
每个进入Call Stack的都叫做 Stack Frame(栈帧)。

这就是当异常抛出的时候，call stack内部是如何记录步骤的 —— 它基本就是异常发生时Call Stack的状态。看下面的代码
```
function foo() {
    throw new Error('SessionStack will help you resolve crashes :)');
}
function bar() {
    foo();
}
function start() {
    bar();
}
start();
```
如果这段代码（假设在一个叫做foo.js的文件内）放在Chrome里执行，那么会产生如下的栈追踪。
![Alt text](https://cdn-images-1.medium.com/max/800/1*T-W_ihvl-9rG4dn18kP3Qw.png "title")

**"栈溢出"** —— 这发生在你达到了调用栈的最大调用个数，这很容易发生，尤其是在你用一些不经测试的递归代码时。看一下下面的代码：
```
function foo() {
    foo();
}

foo();
```
当引擎开始执行这段代码时，它以调用“foo”开始，然而这个函数是递归的，并且在没有任何终止可能的情况下开始调用自己。所以每执行一步，都会把foo这个function加到Call Stack的顶部。看起来就像这样：
![Alt text](https://cdn-images-1.medium.com/max/800/1*AycFMDy9tlDmNoc5LXd9-g.png "title")

然而，在某些时刻，当函数的调用数量超过了调用栈的实际大小时，浏览器会做抛出一个错误。
![Alt text](https://cdn-images-1.medium.com/max/800/1*e0nEd59RPKz9coyY8FX-uw.png "title")

在单线程上运行代码会很简单，你不必处理那些多线程造成的复杂场景 —— 例如，死锁。

但是在单线程上运行也有限制。因为JavaScript只有一个单一的调用栈，**当事情进展缓慢会发生什么？**

## 并发 & 时间循环
当调用栈中调用了需要大量时间处理的函数时会发生什么呢？例如,想象你想用JavaScript在浏览器中做一些复杂的图像转换。
你可能会问 —— 这是个问题吗？在stack里面执行一个会花费大量时间的函数，会导致浏览器被阻塞而做不了任何事。这意味着浏览器无法渲染页面，不能运行代码，它被卡住了。这将会严重影响你的应用的UI和使用体验。

并且这不是唯一的问题。一旦你的浏览器开始在调用栈内处理大量的任务，那么它就会停止响应很长一段时间。随后大多数浏览器会提示错误，询问你是否想要终止当前的页面。

![Alt text](https://cdn-images-1.medium.com/max/800/1*WlMXK3rs_scqKTRV41au7g.jpeg "title")

按照这种情况，这可不是好的用户体验，对吗？

那么我们怎么才能在执行耗时的代码的同时又不阻塞UI并且使浏览器保持相应呢？我们的解决方案是**异步回调**

这将会在下一篇文章详细解释[“How JavaScript actually works” tutorial: “Inside the V8 engine + 5 tips on how to write optimized code”](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e) .

作者：Alexander Zlatkov

编译：[mike](https://github.com/mikeCodemikeLife)

英文原文: [How JavaScript works: an overview of the engine, the runtime, and the call stack](
    https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf
)


