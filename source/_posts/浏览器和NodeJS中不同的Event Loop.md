> 异步处理机制 Event Loop（下文中简称EL或事件轮询） 大概是每个前端的必修知识。写成文章，一是为系统地总结学习 Node.js 的 `Events`、`Timer` 时对于事件轮询机制的追问，而是希望能对处于迷惑阶段的同学有所帮助。如果你和我一样，对于task、microtask、Node事件循环的phase、`process.nextTick()` 执行时机等存在疑惑的话，那么这篇文章可以提供一点参考，如果能解答你的疑惑，不胜荣幸，致谢参考文档。
>
> 本文内容旨在厘清浏览器（browsing context）和Node环境中不同的 Event Loop。

<h4 id="cata">目录</h4>

> - [不同环境中，这段代码的运行结果相同吗？](#example-code)
> - [记住，JS是单线程的](#single-thread)
> - #### [浏览器中的事件轮询](#browsing-context)
>    - ##### [Event Loop在HTML规范中的定义](#defination-b)
>    - ##### [图解Event Loop](#elimg-b)
>    - ##### [task任务](#task)
>    - ##### [microtask微任务](#microtask)
>    - ##### [Event Loop 在浏览器中的循环过程](#el-process-b)
>    - ##### [文章开头的代码解析](#code-analysis-b)
> - #### [Node中的事件轮询](#node)
>    - ##### [事件轮询中的几个阶段](#phase)
>    - ##### [Event Loop 在Node的循环过程](#el-process-n)
>    - ##### [setTimeout 和 setImmediate 的区别](#timeout&immediate)
>    - ##### [process.nextTick()](#nexttick)
>    - ##### [process.nextTick() 和 setImmediate()](#nexttick&immediate)
>    - ##### [文章开头的代码解析](#code-analysis-n)
> - #### [拓展](#expand)
>    - ##### [web worker是什么](#webworker)
>    - ##### [MutationObserver是什么](#mutationobserver)
>    - ##### [执行栈](#callstack)
> - #### [参考](#ref)

<h4 id="example-code">在文章开始之前，我们先来做道题目</h4>
> （请先尝试回答下面代码的运行结果，之后再在浏览器和node下验证。如果答案一致，那么可以关闭当前文档了hhh）


下文中对这段代码会再做分析
```js
setTimeout(() => console.log('setTimeout1'), 0);
setTimeout(() => {
    console.log('setTimeout2');
    Promise.resolve().then(() => {
        console.log('promise3');
        Promise.resolve().then(() => {
            console.log('promise4');
        })
        console.log(5)
    })
    setTimeout(() => console.log('setTimeout4'), 0);
}, 0);
setTimeout(() => console.log('setTimeout3'), 0);
Promise.resolve().then(() => {
    console.log('promise1');
})
```
答案戳
- [浏览器](#code-analysis-b-a)
- [Node](#code-analysis-n-a)

<h4 id="single-thread">js是单线程的，EL机制实现异步</h4>

> 轮询发生的前提：所有代码皆在主线程调用栈完成执行
> 
> 轮询发生的时机：当主线程任务清空后，轮询任务队列中的任务
> 
> 我们要讨论的是，轮询机制在浏览器和Node中的区别




<h2 id="browsing-context">browsing contexts</h2>

<h4 id="defination-b">EL在HTML规范中的定义</h4>
> To coordinate events, user interaction, scripts, rendering, networking, and so forth, user agents must use event loops as described in this section. There are two kinds of event loops: those for browsing contexts, and those for workers.

为了协调事件、用户交互、脚本、UI渲染、网络请求等行为，用户引擎必须使用Event Loop。EL包含两类：基于browsing contexts，基于worker。二者独立。

本文讨论的浏览器中的EL基于browsing contexts

<h4 id="elimg-b">图解Event Loop</h4>

![eventloop](https://raw.githubusercontent.com/aooy/aooy.github.io/master/blog/issues5/img/eventLoop.jpg)

- 同步任务直接进入 ++主执行栈（call stack）++ 中执行
- 等待主执行栈中任务执行完毕，由EL将异步任务推入主执行栈中执行


<h4 id="task">task</h4>
一个EL中有一个或多个task队列，来自不同任务源的task会放入不同的task队列中，比如，用户代理会为鼠标键盘事件分配一个task队列，为其他的事件分配另外的队列。

典型的任务源有以下几种（[Generic task sources](https://www.w3.org/TR/html5/webappapis.html#generic-task-sources)）：

- DOM操作任务源：响应DOM操作
- 用户交互任务源：对用户交互作出反应，例如键盘或鼠标输入。响应用户操作的事件（例如click）必须使用task队列
- 网络任务源：响应网络活动
- history traversal任务源：当调用history.back()等类似的api时，将任务插进task队列


    task在网上也被成为`macrotask` 可能是为了和 `microtask` 做对照。但是规范中并不是这么描述任务的。

除了上述task来源，常见的来源还有 数据库操作、`setTimeout/setInterval`等，可以概括为以下几种

- script代码
- setTimeout/setInterval
- I/O
- UI交互
- setImmediate(nodejs环境中)
    

<h4 id="microtask">Microtask</h4> 
一个EL中只有一个microtask队列，通常下面几种任务被认为是microtask

- promise（`promise`的`then`和`catch`才是microtask，本身其内部的代码并不是）
- MutationObserver
- process.nextTick(nodejs环境中)

<h4 id="el-process-b">EL循环过程</h4>

一个EL只要存在，就会不断执行下边的步骤：

> 1. 在所有task队列中选择一个最早进队列的task，用户代理可以选择任何task队列，如果没有可选的任务，则跳到Microtasks步骤
> 2. 将上边选择的task设置为正在运行的task
> 3. **Run**: 运行被选择的task
> 4. 将event loop的 `currently running task` 置为 `null`
> 5. 从task队列里移除前边Run里运行的task
> 6. **Microtasks**: 执行microtasks任务检查点。（也就是执行microtasks队列里的任务）
> 7. 更新渲染
> 8. *如果这是一个worker event loop，但是task队列中没有任务，并且WorkerGlobalScope对象的closing标识为true，则销毁EL，中止这些步骤，然后 run a worker*
> 9. 返回到第1步

简化一下上面的步骤，可以用下面的伪代码描述EL循环过程：

> 一个宏任务，所有微任务（，更新渲染），一个宏任务，所有微任务（，更新渲染）......
>
> 执行完microtask队列里的任务，有可能会渲染更新。在一帧以内的多次dom变动浏览器不会立即响应，而是会积攒变动以最高60HZ的频率更新视图
    
```js
while (true) {
    宏任务队列.shift()
    微任务队列全部任务()
}
```
![image](https://mengera88.github.io/images/eventqueue2.png)

<h4 id="code-analysis-b">掌握了吗？在浏览器中运行文章开头的代码</h4>
```js
setTimeout(() => console.log('setTimeout1'), 0);    // 1#
setTimeout(() => {                  // 2#
    console.log('setTimeout2');     // 2-1#
    Promise.resolve().then(() => {  // 2-2#
        console.log('promise2');        // 2-2-1#
        Promise.resolve().then(() => {  // 2-2-2#
            console.log('promise3');
        })
        console.log(5)                  // 2-2-3#
    })
    setTimeout(() => console.log('setTimeout4'), 0);    // 2-3#
}, 0);
setTimeout(() => console.log('setTimeout3'), 0);    // 3#
Promise.resolve().then(() => {  // 4#
    console.log('promise1');
})
```

<p id="code-analysis-b-a">运行结果：</p>

```js
promise1
setTimeout1
setTimeout2
promise2
5
promise3
setTimeout3
setTimeout4
```
1. 一个task：运行代码块，从头扫到尾，没有同步执行的代码，**1# 2# 3#** 回调放入task队列，**4#** 回调放入microtask队列，结束 `[task:1# 2# 3#][micro: 4#]`
2. 所有microtasks：执行 **4#** 回调，输出 ==`promise1`==，microtask队列清空，结束
3. 一个task：执行 **1#** 回调，输出 ==`setTimeout1`==，结束 `[task: 2# 3#]`
4. ~~所有microtasks：microtask队列为空，下一tick~~
5. 一个task：执行 **2#** 回调内的代码 => 执行 **2-1#** 输出 ==`setTimeout2`==；**2-2#**回调 放入microtask队列，**2-3#** 放入task队列 `[task: 3# 2-3#][micro: 2-2#]`
6. 所有microtasks：执行 **2-2#** 回调 => 执行 **2-2-1#**，输出 ==promise2==；**2-2-2#** 放入 microtask队列；执行 **2-2-3#**，输出 ==5==；microtask队列中还有任务 **2-2-2#** 回调，执行，输出 ==promise3== `[task: 3# 2-3#]`
7. 一个task：执行 **3#**，输出 ==setTimeout3== `[task: 3# 2-3#]`
8. ~~所有microtasks：microtask队列为空，下一tick~~
9. 一个task：执行 **2-3#**，输出 ==setTimeout4==

<h2 id="node">node</h2>

> Node中的EL由 **libuv库**实现

个人理解，它与浏览器中的轮询机制（一个task，所有microtasks；一个task，所有microtasks…）最大的不同是，node轮询有phase（阶段）的概念，不同的任务在不同阶段执行，进入下一阶段之前执行process.nextTick() 和 microtasks。

<h3 id="phase">Node事件轮询中的几个阶段</h3>

```js
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

> 每个阶段都有一个回调函数FIFO队列。EL进入一个阶段会执行里面所有的操作，然后执行回调函数，直到队列消耗尽，或是回调函数执行数量达到最大限制，进入下一个阶段
>
> 阶段里的执行队列：
> - Timers Queue `setTimeout()` `setInterval()`设定的回调函数
> - I/O Queue 几乎所有的回调，除了timers、close callbacks、check阶段的回调
> - Check Queue `setImmediate()` 设定的回调函数
> - Close Queue 比如 `socket.on('close', ...)`

##### timers

timers指定阈值之后执行`setTimeout()` `setInterval()` 设定的回调函数，但该阈值不是执行回调的确切时间，只是最短的间隔时间，node内核调度机制和其他的回调函数会推迟它的执行

由poll阶段来控制什么时候执行timers

##### I/O callbacks
##### ~~idle, prepare内部使用，忽略~~
##### poll

获取新的I/O事件，node会在适当的情况下阻塞在这里

为防止poll phase 耗尽 event loop，libuv 也有一个最大值（基于系统），会在超过最大值之后停止轮询更多的事件

执行两种函数
1. 为到时的timers执行脚本，然后
2. 执行poll队列中的事件

如果到了poll阶段，没有timers被调度，下面的情况会发生
1. poll队列不为空

    - 迭代执行回调队列中的函数，直到清空或者到达系统设定的时间限制
    
2. poll队列为空

    - 如果代码已经被`setImmediate`调度，EL立即结束poll phase进入check phase执行被调度的脚本
    - 没有被`setImmediate`调用，则等待timers回调函数被加到poll队列中，加进去就回到timers phase执行回调
    
    
##### check

- 一旦poll队列闲置下来或者是代码被`setImmediate`调度，EL会马上进入check phase
- check是特殊的timer

##### close callbacks

- 如果socket突然中断，`close`事件会在这个阶段被触发
- 或者 `close` 事件是被 `process.nextTick()` 触发

```js
var fs = require('fs');
function someAsyncOperation (callback) {
    // 假设用了95ms
    fs.readFile('/path/to/file', callback);
}

var timeoutScheduled = Date.now();
setTimeout(function () {
    var delay = Date.now() - timeoutScheduled;
    console.log(delay + "ms have passed since I was scheduled");
}, 100);

someAsyncOperation(function () {
    var startCallback = Date.now();
    while (Date.now() - startCallback < 10) {
        ;
    }
});

// log: 105ms have passed since I was scheduled
```
1. timers：至少需要100ms `setTimeout`的回调才会被执行，所以进入下阶段
2. I/O callbacks：没有回调队列
3. poll：当前队列还是空的，会等待一定时间，等最近的一个timer回调，需要等100ms，但是当95ms的时候 `readFile` 完成，此时回调被加入了poll的队列，执行10ms
4. timer的阈值时间100ms到了，回到timers阶段执行回调函数，打log


<h3 id="el-process-n">循环过程</h3>

> 对于循环开始之前的process.nextTick() microtasks会不会被处理，和小组的同学们有过讨论。在node官方文档中我们看到这样的定义
> 
> When Node.js starts, it initializes the event loop, processes the provided input script (or drops into the REPL, which is not covered in this document) which may make async API calls, schedule timers, or call process.nextTick(), then begins processing the event loop.
>
> 所以开始循环之前，process.nextTick()/microtasks 是会被先清掉的

> 循环开始之前

1. 同步任务
2. 发出异步请求
3. 规划定时器生效时间
4. 执行`process.nextTick()`

> 开始循环

1. 清空当前循环内的 Timers Queue，清空NextTick Queue，清空Microtask Queue
2. 清空当前循环内的 I/O Queue，清空NextTick Queue，清空Microtask Queue
3. 清空当前循环内的 Check Queue，清空NextTick Queue，清空Microtask Queue
4. 清空当前循环内的 Close Queue，清空NextTick Queue，清空Microtask Queue
5. 进入下一轮循环

优先级：`nextTick` > `microtask`  |  `setTimeout/setInterval` > `setImmediate`

> 伪代码

```js
while (true) {
  loop.forEach((阶段) => {
    阶段全部任务()
    nextTick全部任务()
    microTask全部任务()
  })
  loop = loop.next
}
```

<h4 id="timeout&immediate">setTimeout 和 setImmediate 的区别</h4>

- `setImmediate` 一旦当前poll阶段结束就执行一次脚本
- `setTimeout` 设定一个最短的调度该脚本的时间阈值
- 不在同一个I/O cycle中的时候，回调的调度顺序是不被保证的

    ```js
    // timeout_vs_immediate.js
    setTimeout(() => {
        console.log('timeout');
    }, 0);
    
    setImmediate(() => {
        console.log('immediate');
    });
    
    // terminal
    $ node timeout_vs_immediate.js
    timeout
    immediate
    
    $ node timeout_vs_immediate.js
    immediate
    timeout
    ```
- 在同一个I/O cycle中，`immediate` 总比 `timeout` 更早被调度

    ```js
    // timeout_vs_immediate.js
    const fs = require('fs');
    
    fs.readFile(__filename, () => {
        setTimeout(() => {
            console.log('timeout');
        }, 0);
        setImmediate(() => {
            console.log('immediate');
        });
    });
    
    // terminal
    $ node timeout_vs_immediate.js
    immediate
    timeout
    
    $ node timeout_vs_immediate.js
    immediate
    timeout
    ```

<h4 id="nexttick">process.nextTick()</h4>

`process.nextTick()` 不是Node的EL中的一部分（虽然它也是异步API），但是，任意阶段的操作结束之后 `nextTickQueue` 就会被处理。通过`process.nextTick()`触发的回调也会在进入下一阶段前被执行结束，这会允许用户递归调用 `process.nextTick()` 造成I/O被榨干，使EL不能进入poll阶段

> 文档中解释了为何允许这种情况存在

NodeJS的设计哲学是：**API应始终是异步的**，即使不必要的情况下也该如此

(这是[官方文档](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)的例子)，最新的API允许在 `process.nextTick()` 里在 `callback` 之后传递任意参数，这些参数会传给 `callback` 就不需要嵌套函数了
```js
function apiCall(arg, callback) {
  if (typeof arg !== 'string')
    return process.nextTick(callback, new TypeError('argument should be string'));
}
```
> [引用译文](http://blog.csdn.net/juhaotian/article/details/78997587)
> 
> 我们所做的是传递一个异常给用户, 但只有在我们允许执行剩余的用户代码时才会传递这个异常. 通过使用 `process.nextTick()`, 可以确保 `apiCall()` 总是在剩余的用户代码之后并且在事件循环被允许进入下一阶段之前执行 `callback` 函数. 为了实现这一点, JS调用栈被允许进行栈展开(译者注: `stack unwinding`, 抛出异常时，将暂停当前函数的执行，开始查找匹配的 `catch` 子句。首先检查 `throw` 本身是否在 `try` 块内部，如果是，检查与该 `try` 相关的 `catch` 子句，看是否可以处理该异常。如果不能处理，就退出当前函数，并且释放当前函数的内存并销毁局部对象，继续到上层的调用函数中查找，直到找到一个可以处理该异常的 `catch`。这个过程称为栈展开, 即 `stack unwinding`。当处理该异常的 `catch` 结束之后，紧接着该 `catch` 之后的点继续执行), 然后立即执行提供的回调, 这个回调允许开发者递归调用 `process.nextTick()`, 而不会触发 `RangeError: Maximum call stack size exceeded from v8` 这个异常.


另一个例子：`.listen` 传入端口号的时候，其回调函数会被立即执行，但是此时回调函数还未设置；为了避免这种情况，`listen`事件会被加入 `nextTickQueue`，这样就能使代码执行完成，允许用户添加需要的事件处理函数
```js
const server = net.createServer(() => {}).listen(8080);
server.on('listening', () => {});
```

<h4 id="nexttick&immediate">process.nextTick() 和 setImmediate() </h4>
> 官方推荐使用 `setImmediate()`，因为更容易推理，也兼容更多的环境，例如浏览器环境

- `process.nextTick()` 在当前循环阶段结束之前触发
- `setImmediate()` 在下一个事件循环中的check阶段触发

<h4 id="code-analysis-n">掌握了吗：在浏览器中运行文章开头的代码</h4>
```js
setTimeout(() => console.log('setTimeout1'), 0);    // 1#
setTimeout(() => {                  // 2#
    console.log('setTimeout2');     // 2-1#
    Promise.resolve().then(() => {  // 2-2#
        console.log('promise2');        // 2-2-1#
        Promise.resolve().then(() => {  // 2-2-2#
            console.log('promise3');
        })
        console.log(5)                  // 2-2-3#
    })
    setTimeout(() => console.log('setTimeout4'), 0);    // 2-3#
}, 0);
setTimeout(() => console.log('setTimeout3'), 0);    // 3#
Promise.resolve().then(() => {  // 4#
    console.log('promise1');
})
```

<p id="code-analysis-n-a">运行结果</p>

```js
promise1
setTimeout1
setTimeout2
setTimeout3
promise2
5
promise3
setTimeout4
```
过程分析：
1. 同步代码执行结束，开始Event Loop之前，需要清掉遗留的 nextTick队列和microtask队列中的任务，执行 **4#**回调，输出 ==promise1==
2. 开始第一轮EL

    1. timer队列中有 **1# 2# 3#** 回调，执行并输出 ==setTimeout1、setTimeout2、setTimeout3==
        - 执行 **2#** 回调 =>  **2-2#** 回调 放入microtask队列，**2-3#** 回调放入下一轮的 timer队列
        - 进入下一阶段之前，清除当前nextTick队列，为空，清除microtask队列
        - 执行 **2-2#** 回调 => 执行 **2-2-1#**，输出 ==promise2==；**2-2-2#** 回调放入 microtask队列；执行 **2-2-3#**，输出 ==5== ；
        - microtask队列中有 **2-2-2#** 回调，执行，输出 ==promise3==
        - microtask队列空了，进入下一阶段
    2. I/O Check Close阶段的队列均为空，此次轮询结束
3. 第二轮EL

    1. timer队列中有 **2-3#** 回调，执行，输出 ==setTimeout4==



<h2 id="expand">拓展</h2>

<h4 id="webworker"> web worker </h4>
[【转向Javascript系列】深入理解Web Worker](http://www.alloyteam.com/2015/11/deep-in-web-worker/)
[Web Worker浅识](http://note.youdao.com/noteshare?id=eb287f54753e456315c28cc9f1b17741)

<h4 id="mutationobserver">MutationObserver</h4>

[MutationObserver - Web API 接口 | MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)

<h4 id="callstack">执行栈</h4>
javaScript是单线程，也就是说只有一个主线程，主线程有一个栈，每一个函数执行的时候，都会生成新的execution context（执行上下文），执行上下文会包含一些当前函数的参数、局部变量之类的信息，它会被推入栈中， running execution context（正在执行的上下文）始终处于栈的顶部。当函数执行完后，它的执行上下文会从栈弹出。

![call-stack](https://raw.githubusercontent.com/aooy/aooy.github.io/master/blog/issues5/img/ec.jpg)


简单的例子
```js
function bar() {
console.log('bar');
}

function foo() {
console.log('foo');
bar();
}

foo();
```
执行栈的变化
![call-stack-change](https://raw.githubusercontent.com/aooy/aooy.github.io/master/blog/issues5/img/ec2.jpg)


<h2 id="ref">参考</h3>
- [HTML 5.2: 7. Web application APIs](https://www.w3.org/TR/html5/webappapis.html#event-loops)
- [你不得不知的Event Loop](https://juejin.im/post/5aa88faf6fb9a028ca52a96f)
- [浏览器和Node不同的事件循环（Event Loop）](https://juejin.im/post/5aa5dcabf265da239c7afe1e)
- [JavaScript 运行机制详解：再谈Event Loop](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)
- [Web Worker 是什么鬼？](http://www.cnblogs.com/zichi/p/4954328.html)
- [node的事件机制](https://segmentfault.com/a/1190000008473830)
- [The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [NodeJS官方文档中文版之《事件循环, 定时器和process.nextTick()》](http://blog.csdn.net/juhaotian/article/details/78997587)
- [从event loop规范探究javaScript异步及浏览器更新渲染时机](https://github.com/aooy/blog/issues/5)