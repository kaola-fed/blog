---
title: 「译」Promises/A+规范
date: 2017-07-31
---



[官网原文](https://promisesaplus.com/)

**一份针对健全、通用JavaScript promises对象的开放标准 — 由实现者制定，供实现者参考。**

一个 *`promise`* 对象代表一个异步操作的最终结果。与promise进行交互的主要方式是通过它的 **`then`** 方法，通过该方法注册回调函数，进而接受promise对象最终的值（value）或不能完成（fulfill）的原因（reason）。

<!-- more -->

本规范详细阐述了 **`then`** 方法的行为，给所有遵循本规范的promise实现提供了一份通用的实现基础。所以本规范可以认为是非常稳定的。尽管 Promises/A+ 组织可能偶尔会为了处理一些新发现的边界情况（corner cases）对本规范进行微小的向后兼容的修正，如果将要加入较大的不兼容的修正，我们一定会进行仔细详尽的考虑、讨论以及测试。

从历史上看，本规范是对先前 [Promises/A](http://wiki.commonjs.org/wiki/Promises/A) 规范条款的澄清明确，一方面对其扩展进而覆盖 *事实(de facto)* 行为（译者注：事实行为应该代表Promise/A中那些已经被广泛采用实现并且规范化的事实标准）；另一方面删除了其中没有规范化或者存在问题的规范条款。

最后，本规范的核心不会讨论如何创建（create）、完成（fulfill）或者拒绝（reject）promises对象，而是会专注于如何提供一个通用的 **`then`** 方法。其它相关规范的未来工作可能会涉及上面的三个主题。






## 1. 术语（Terminology）

**1.1** "promise" 是一个遵循本规范、并拥有 **`then`** 方法的对象或函数。

**1.2** "thenable" 是一个定义了 **`then`** 方法的对象或函数。

**1.3** "value" 是JavaScript中任意一种合法值(包括 **`undefined`**，"thenable"以及"promise")。

**1.4** "exception" 是一个使用 **`throw`** 语句抛出的值。

**1.5** "reason" 是一个指示了promise为何被拒绝（rejected）的值。

## 2. 要求（Requirements）

### 2.1 Promise 的状态

一个 Promise 必须是以下三种状态之一：等待态（pending）、完成态（fulfilled）或拒绝态（rejected）。

#### 2.1.1 处于等待态（pending）时，promise 对象：

- **2.1.1.1** 可以跳转至完成态或拒绝态

#### 2.1.2 处于执行态（fulfilled）时，promise 对象：

- **2.1.2.1** 不能跳转至任何其它状态

- **2.1.2.2** 必须拥有一个固定不可变的值(value)

#### 2.1.3 处于拒绝态（rejected），promise 对象：

- **2.1.3.1** 不能跳转至任何其它状态

- **2.1.3.2** 必须拥有一个固定不可变的原因(reason)

> 这里的“固定不可变”指的是同一个值（identity）（例如，可通过 **`===`** 的比较），但并不意味深层次的同一性（译者注：对于复杂深层次对象，深层次属性值的更改不代表promise的值或原因的变化）。

### 2.2 **`then`** 方法

一个 promise 对象必须提供一个 **`then`** 方法来访问其当前值、最终的值或原因。

promise 对象的 **`then`** 方法接受两个参数：

```javascript
promise.then(onFulfilled, onRejected)
```

#### 2.2.1 **`onFulfilled`** 和 **`onRejected`** 都是可选参数。

- **2.2.1.1** 如果 **`onFulfilled`** 不是函数，必须将其忽略

- **2.2.1.2** 如果 **`onRejected`** 不是函数，必须将其忽略

#### 2.2.2 如果 onFulfilled 是一个函数：

- **2.2.2.1** 当 promise 完成（fulfilled）后必须对其调用，并且 promise 最终的值（value）作为第一个参数；

- **2.2.2.2**  promise 未完成（fulfilled）前，不能对其调用

- **2.2.2.3**  调用次数不可超过一次

#### 2.2.3 如果 onRejected 是一个函数：

- **2.2.3.1** 当 promise 被拒绝（rejected）后必须对其调用，并且 promise 最终的原因（reason）作为第一个参数；

- **2.2.3.2** promise 未被拒绝（rejected）前，不能对其调用

- **2.2.3.3** 调用次数不可超过一次

#### 2.2.4 **`onFulfilled`** 和 **`onRejected`** 只有在[执行上下文（execution context）](https://es5.github.io/#x10.3)堆栈仅包含平台代码（platform code）时才可被调用。[[3.1](#3.1)]

#### 2.2.5 **`onFulfilled`** 和 **`onRejected`**  必须被作为函数调用（例如没有 **`this`** 值）。[[3.2](#3.2)]

#### 2.2.6 **`then`** 方法可以被同一个promise对象调用多次。

- **2.2.6.1** 如果/当 **`promise`** 执行完成后，所有 **`onFulfilled`** 回调函数，必须按照其最初调用 **`then`** 的顺序依次执行。

- **2.2.6.2** 如果/当 **`promise`** 执行被拒绝后，所有 **`onRejected`** 回调函数，必须按照其最初调用 **`then`** 的顺序依次执行。

#### 2.2.7 **`then`** 方法必须返回一个promise对象 [[3.3](#3.3)]

```javascript
promise2 = promise1.then(onFulfilled, onRejected);   
```

- **2.2.7.1** 如果 **`onFulfilled`** 或者 **`onRejected`** 返回一个值 **`x`** ，运行 *Promise Resolution Procedure* **`[[Resolve]](promise2, x)`**

- **2.2.7.2** 如果 **`onFulfilled`** 或者 **`onRejected`** 抛出一个异常 **`e`**，则 **`promise2`** 必须拒绝执行，并以 **`e`** 作为拒绝的原因。

- **2.2.7.3** 如果 **`onFulfilled`** 不是函数，并且 **`promise1`** 执行完成，**`promise2`** 必须成功完成并返回相同的值（value）。

- **2.2.7.4** 如果 **`onRejected`** 不是函数，并且 **`promise1`** 执行被拒绝，**`promise2`** 必须以相同的原因（reason）被拒绝执行。

### 2.3 Promise Resolution Procedure（PRP）

**`Promise Resolution Procedure`** 是一个抽象的操作，以一个 promise 对象和一个值（value）作为输入参数，我们表示为 **`[[Resolve]](promise, x)`**。如果 **`x`** 是一个 **`thenable`** 的对象（即一个拥有 **`then`** 方法的函数或对象），并且其行为和一个promise对象至少有些许相似，PRP就会尝试让 **`promise`** 接受 **`x`** 的状态；否则，其用 **`x`** 的值来执行完成 **`promise`**。

这种处理 **`thenables`** 的方式使得各种promise的实现可以互通，只要它们开放出一个与Promise/A+协议兼容的 **`then`** 方法即可。这也使得遵循 Promise/A+ 规范的实现可以 **`接受`** （译者注：这里的 **`接受`**, 原文使用了assimilate这个单词，原意为消化吸收，这里面指的就是可以接受其它非规范化实现的then方法）那些未遵循规范实现的合理的 **`then`** 方法。

通过以下步骤来运行 **`[[Resolve]](promise, x)`**：

- **2.3.1** 如果 **`promise`** 和 **`x`** 引用同一对象，以 **`TypeError`** 为原因拒绝执行 **`promise`**。


- **2.3.2** 如果 **`x`** 是一个promise对象，则promise采纳 **`x`** 的状态 [[3.4](#3.4)]：

  - **2.3.2.1** 如果 **`x`** 处于等待态（pending），**`promise`** 必须保持为等待态（pending）直到 **`x`** 完成（fulfilled）或 拒绝（rejected）。

  - **2.3.2.2** 如果/当 **`x`** 已经完成（fulfilled），使用相同的值执行完成 **`promise`**。

  - **2.3.2.3** 如果/当 **`x`** 被拒绝（rejected），使用相同的原因拒绝执行 **`promise`**。

- **2.3.3** 其它情况，如果 **`x`** 为一个对象或者函数：

  - **2.3.3.1** 把 **`then`** 赋值为 **`x.then`**[[3.5](#3.5)]

  - **2.3.3.2** 如果取 **`x.then`** 的值时抛出错误 **`e`**，则使用 **`e`** 为原因拒绝 **`promise`**。

  - **2.3.3.3** 如果 **`then`** 是一个函数，则以 **`x`** 作为 **`this`** 的值对其进行调用，并以 **`resolvePromise`** 作为第一个参数，**`x`** 作为第二个参数，其中:
  
    - **2.3.3.3.1** 如果/当以 **`y`** 值为参数调用 **`resolvePromise`** 时，则运行 **`[[Resolve]](promise, y)`**。
  
    - **2.3.3.3.2** 如果/当以 **`r`** 原因为参数调用 **`rejectPromise`** 时，则使用原因 **`r`** 拒绝执行 **`promise`**。
  
    - **2.3.3.3.3** 如果 **`resolvePromise`** 和 **`rejectPromise`** 都被调用，或者被使用相同的参数调用了多次，则首次调用将被优先采用，其它调用将被忽略。
  
    - **2.3.3.3.4** 如果调用 **`then`** 方法过程中抛出异常 **`e`**：
    
      - **2.3.3.3.4.1** 如果 **`resolvePromise`** 或 **`rejectPromise`** 都已经被调用，则忽略该异常。
    
      - **2.3.3.3.4.2** 否则使用 **`e`** 为原因拒绝执行 **`promise`**。

  - **2.3.3.4** 如果 **`then`** 不是函数，以 **`x`** 为值完成 **`promise`**。

- **2.3.4** 如果 **`x`** 不是对象或者函数，以 **`x`** 为值完成 **`promise`**。

如果一个promise对象被一个处于循环thenable链中的thenable对象解决（resolve），由于 **`[[Resolve]](promise, thenable)`** 的递归本质会使得其再次被调用，按照上面的算法，这种情况将导致无限递归。本规范鼓励实现者检测这种递归情况的出现，并使用带有一定信息的 **`TypeError`** 作为原因拒绝执行 **`promise`** [[3.6](#3.6)]，但规范不对此检测做强制要求。

## 3. 注释

- **3.1** 这里“平台代码”指的是引擎(engine)、环境(environment)以及 promise 的实现代码（implementation code）。实际上，这个要求确保了 **`onFulfilled`** 和 **`onRejected`** 方法能够异步执行，即在调用 **`then`** 方法的那个事件循环之后的新执行栈中异步执行。这个要求可以通过“宏任务（macro-task）”机制（例如 [**`setTimeout`**](https://html.spec.whatwg.org/multipage/webappapis.html#timers) 或者 [**`setImmediate`**](https://dvcs.w3.org/hg/webperf/raw-file/tip/specs/setImmediate/Overview.html#processingmodel)）或者“微任务（micro-task）”机制（例如 [**`MutationObserver`**](https://dom.spec.whatwg.org/#interface-mutationobserver) 或者 [**`process.nextTick`**](http://nodejs.org/api/process.html#process_process_nexttick_callback)）来实现。既然 promise 的实现代码本身也被认为是“平台代码”，所以其自身可能也会包含一种任务调度队列或者“trampoline”控制结构来对其处理程序进行调用。

- **3.2** 也就是说，**`this`** 的值，在严格模式（strict）下为 **`undefined`**；在非严格模式()下为全局对象（global object）。

- **3.3** 在满足所有要求的情况下，现实实现中可能允许 **`promise2 === promise1`**。每个实现都需要说明其是否允许出现 **`promise2 === promise1`**，以及在何种条件下出现。

- **3.4** 通常来说，只有当 **`x`** 是从当前实现中定义出来的，我们才知道它是一个真正的promise对象。这条规则允许使用特定实现中的方法去采用符合规范的promise对象的状态。

- **3.5** 这一步骤中，我们首先会存储一个指向 x.then 的引用，然后测试该调用，之后调用该引用，这一系列过程为了防止对 x.then 属性的多处访问。这些预防措施对于保证访问器属性的一致性至关重要，因为其值可能在多次取值期间发生变化。

- **3.6** 本规范的实现 *不应该* 任意限定thenable链的深度，并认定只要超过了该限定递归就是无限的。只有真正的循环递归才应导致一个 **`TypeError`**；如果一条无限长链上的thenable对象各不相同，那么无限递归下去则是正确的行为。

By [Hong](https://github.com/llwanghong/blog)