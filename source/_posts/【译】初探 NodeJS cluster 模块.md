<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
> 原文地址：http://www.acuriousanimal.com/2017/08/12/understanding-the-nodejs-cluster-module.html

NodeJS 是单进程应用，这意味着默认情况下它不会利用多核系统。如果你有一个8核的 CPU，并且通过 `$ node app.js` 来运行一个 Node 程序，那么它只会在一个进程中运行，其余的 CPU 将会浪费掉。

幸运的是NodeJS 提供了一个 [cluster](https://nodejs.org/api/cluster.html) 模块，它提供了一系列的函数和属性，来帮助我们创建能充分使用 CPU 的程序。不足为奇，cluster 模块中能最大化使用 CPU 正是通过衍生(fork)子进程的方式来达到的，类似于老 [fork()](http://www.includehelp.com/c-programs/c-fork-function-linux-example.aspx) 系统调用 Unix 系统。

### cluster 模块介绍

cluster 模块包含了一系列的函数和属性，来帮助我们衍生子进程并充分利用多核系统。
通过 cluster 模块，父进程衍生出来的子进程可通过 [IPC](https://en.wikipedia.org/wiki/Inter-process_communication) 通道来与父进程进行通信。**记住进程之间是不共享内存的，是相互独立的内存空间**

下面是引用了 NodeJS 官方文档来解释 cluster 模块

>Node.js在单个线程中运行单个实例。 为了利用多核系统，用户有时需要启动一个进程集群来处理负载任务。
>
>cluster 模块允许简单容易的创建共享服务器端口的子进程。
>
>工作进程由 `child_process.fork()` 方法创建，因此它们可以使用IPC和父进程通信，从而使各进程交替处理连接服务。`child_process.fork()` 方法是 `child_process.spawn()` 的一个特殊情况，专门用于衍生新的 Node.js 进程。 跟 `child_process.spawn()` 一样返回一个 `ChildProcess` 对象。 返回的 `ChildProcess` 会有一个额外的内置的通信通道，它允许消息在父进程和子进程之间来回传递。 详见 `subprocess.send()`。
>
>衍生的 Node.js 子进程与两者之间建立的 IPC 通信信道的异常是独立于父进程的。 每个进程都有自己的内存，使用自己的 V8 实例。 由于需要额外的资源分配，因此不推荐衍生大量的 Node.js 进程。

所以大部分的工作其实是由 [child_process](https://nodejs.org/api/child_process.html) 模块来完成的，它可以产生新的进程并且帮助他们进行通信，比如创建管道(pipes)。推荐阅读文章[你所不知道的Node.js子进程](https://medium.freecodecamp.org/node-js-child-processes-everything-you-need-to-know-e69498fe970a)。

### 简单用例

让我们来看一个简单的例子：

* 创建一个主进程，用于检索CPU的数量并为每个CPU分配一个子进程
* 每个子进程打印一条信息并退出

```js
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  masterProcess();
} else {
  childProcess();  
}

function masterProcess() {
  console.log(`Master ${process.pid} is running`);

  for (let i = 0; i < numCPUs; i++) {
    console.log(`Forking process number ${i}...`);
    cluster.fork();
  }

  process.exit();
}

function childProcess() {
  console.log(`Worker ${process.pid} started and finished`);

  process.exit();
}
```
保存代码为 `app.js` 并执行 `$ node app.js`。打印结果类似如下：

```js
$ node app.js

Master 8463 is running
Forking process number 0...
Forking process number 1...
Forking process number 2...
Forking process number 3...
Worker 8464 started and finished
Worker 8465 started and finished
Worker 8467 started and finished
Worker 8466 started and finished
```

#### 代码解释

当我们执行 `app.js` 的时候，操作系统会创建一个进程来运行我们的代码。我们引入 `cluster` 模块，并通过 `isMaster` 属性来判断是不是主进程。因为是第一个进程，所以 `isMaster` 值为 `true`，接着执行 `masterProcess` 函数。在 `masterProcess` 函数中根据 CPU 个数，从当前进程中循环 `fork` 出相同的子进程数。

通过 `fork` 方法创建的进程，就像是命令行中执行 `$node app.js` 一样，也就是说现在有多个进程在跑 `app.js` 的代码。

当每个子进程被创建和执行的时候，做的是跟第一次父进程一样的事，引入 `cluster` 模块执行 `if` 判断，不同的是现在 `isMaster` 为 `false`，所以他们执行 `childProcess` 函数。

> 注：NodeJS 还提供了 [Child Processes](https://nodejs.org/api/child_process.html) 模块，可以简化与其他进程的创建和通信。例如，我们执行一条 `ls -l` 的终端指令，让另外一个进程的标准输出通过管道(pipe)连接，作为它的标准输入。
>
> 译者注：例如 `ls -l /etc/ | wc -l`

### 父子进程通信

子进程一旦被创建，主进程和子进程之间的 IPC 通道也会被创建，我们可以通过 `send()` 方法来进行通信，`send()` 方法接受一个 object 对象作为参数。因为他们是不同的进程（不是线程），所以不能通过内存共享的方式来进行通信。

在主进程中，我们可以使用进程引用（即 `someChild.send({...})`）向子进程发送消息，并且在子进程内，我们可以简单地使用当前进程引用向主进程发送消息 `process.send()`。

修改 `childProcess` 函数如下：

```js
function childProcess() {
  console.log(`Worker ${process.pid} started`);

  process.on('message', function(message) {
    console.log(`Worker ${process.pid} recevies message '${JSON.stringify(message)}'`);
  });

  console.log(`Worker ${process.pid} sends message to master...`);
  process.send({ msg: `Message from worker ${process.pid}` });

  console.log(`Worker ${process.pid} finished`);
}
```
我们通过 `process.on('message', handler)` 来监听 `message` 事件，然后通过 `process.send({...})` 来发送消息。这里的消息是一个 Object 对象。

```js
let workers = [];

function masterProcess() {
  console.log(`Master ${process.pid} is running`);

  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    console.log(`Forking process number ${i}...`);

    const worker = cluster.fork();
    workers.push(worker);

    // Listen for messages from worker
    worker.on('message', function(message) {
      console.log(`Master ${process.pid} recevies message '${JSON.stringify(message)}' from worker ${worker.process.pid}`);
    });
  }

  // Send message to the workers
  workers.forEach(function(worker) {
    console.log(`Master ${process.pid} sends message to worker ${worker.process.pid}...`);
    worker.send({ msg: `Message from master ${process.pid}` });    
  }, this);
}
```
在 `masterProcess` 函数中，我们通过循环创建了 CPU 数等量的子进程，其中 `cluster.fork()` 返回的是一个子进程对象，我们把它存储在 `workers` 数组中，同时监听由子进程发来的信息。另外我们也通过 `worker.send({...})` 从主进程往对应子进程发送消息。

打印结果如下：

```js
$ node app.js

Master 4045 is running
Forking process number 0...
Forking process number 1...
Master 4045 sends message to worker 4046...
Master 4045 sends message to worker 4047...
Worker 4047 started
Worker 4047 sends message to master...
Worker 4047 finished
Master 4045 recevies message '{"msg":"Message from worker 4047"}' from worker 4047
Worker 4047 recevies message '{"msg":"Message from master 4045"}'
Worker 4046 started
Worker 4046 sends message to master...
Worker 4046 finished
Master 4045 recevies message '{"msg":"Message from worker 4046"}' from worker 4046
Worker 4046 recevies message '{"msg":"Message from master 4045"}'
```

### 小结

[cluster](https://nodejs.org/api/cluster.html) 模块为 NodeJS 提供了充分利用 CPU 所需的功能，不仅弥补了 [child process](https://nodejs.org/api/child_process.html) 模块的不足，如提供了大量工具来处理进程：启动，停止和管道输入/输出等，而且帮助我们能轻松创建子进程，通过创建 IPC 通道来进行父子进程间的通信。尽管在这篇文章中没能体现所有功能，但它对应用带来性能上的提升是不争的事实。下篇文章我将介绍 HTTP 服务中 cluster 模块的作用。

by 贾克斯