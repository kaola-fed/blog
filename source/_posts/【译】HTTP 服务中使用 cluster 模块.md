<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
> 原文地址：http://www.acuriousanimal.com/2017/08/18/using-cluster-module-with-http-servers.html

[cluster](https://nodejs.org/api/cluster.html) 模块能够使我们在多核系统中提高应用性能。无论是基于何种服务，我们都希望尽可能利用每台机器上的所有 CPU。它允许我们在一组工作进程之间对传入的请求进行负载均衡，并因此提高我们应用程序的吞吐量。

在之前的[文章](http://www.acuriousanimal.com/2017/08/12/understanding-the-nodejs-cluster-module.html)中，我介绍了集群模块，并展示了它的一些基本用法，以创建工作进程并将其与主进程通讯。 在这篇文章中，我们将看到如何在创建HTTP服务器时使用集群模块，无论是使用纯HTTP模块还是使用ExpressJS。

### HTTP 服务中使用 cluster 模块

让我们来看看如何创建一个基础HTTP服务来享受集群模块带来的利益。

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
}

function childProcess() {
  console.log(`Worker ${process.pid} started...`);

  http.createServer((req, res) => {
    res.writeHead(200);
    res.end('Hello World');
  }).listen(3000);
}
```

在 `masterProcess` 函数中，我们为每个 CPU 派生一个工作进程。另一方面，`childProcess` 只是在端口 3000 上创建一个 HTTP 服务，然后返回一个带有 200 状态码的 `Hello World` 文本字符串。

执行结果如下：

```shell
$ node app.js

Master 1859 is running
Forking process number 0...
Forking process number 1...
Forking process number 2...
Forking process number 3...
Worker 1860 started...
Worker 1862 started...
Worker 1863 started...
Worker 1861 started...
```

基本上我们的初始进程(主)会为每个 CPU 运行一个处理 HTTP 服务请求的新进程。正如你所看到，这可以提高你的服务器性能，因为本由一个进程处理100万条请求的任务，现在改由4个进程来处理了。

### cluster 模块如何与网络连接一起工作

上面的例子看上去简单，但却暗藏玄机。我们知道在任意系统中，一个进程可以使用一个端口与其他系统通信，这也意味着，给定的端口只能被改进程使用。那么问题来了，**为什么新建的工作进程能使用相同的端口呢？**

简单来说，是主进程监听了给定的端口，并负载均衡所有子进程之间的请求。按官方文档来说：

>工作进程由child_process.fork()方法创建，因此它们可以使用IPC和父进程通信，从而使各进程交替处理连接服务。
>
>cluster模块支持两种连接分发模式（将新连接安排给某一工作进程处理）。
>
>* 第一种方法（也是除Windows外所有平台的默认方法），是循环法。由主进程负责监听端口，接收新连接后再将连接循环分发给工作进程。在分发中使用了一些内置技巧防止工作进程任务过载。
>
>* 第二种方法是，主进程创建监听socket后发送给感兴趣的工作进程，由工作进程负责直接接收连接。
>
>**只要有存活的工作进程，服务器就可以继续处理连接。如果没有存活的工作进程，现有连接会丢失，新的连接也会被拒绝。**

### 集群模块负载均衡的其他选择

集群模块允许主进程接收请求并在所有工作进程之间进行负载平衡。 这是一种提高性能的方法，但并不是唯一的方法。

在文章 [cluster 模块，iptables以及Nginx之间的负载均衡性能比较](https://medium.com/@fermads/node-js-process-load-balancing-comparing-cluster-iptables-and-nginx-6746aaf38272) 中，你可以在集群模块，iptables和nginx反向代理之间找到性能比较。

### 小结
如今的性能在任何Web应用程序中都是强制性的，我们需要支持高吞吐量和快速服务数据。

集群模块是一个可行的解决方案，它允许我们有一个主进程，并为每个核心创建一个工作进程，以便它们运行一个HTTP服务器。 群集模块提供了两个很棒的功能：

* 通过创建IPC通道并允许使用 `process.send()` 发送消息来简化主进程和工作进程之间的通信
* 允许工作进程共享相同的端口。 这样做使得主进程接收请求并在工作者之间进行多路复用

by 贾克斯