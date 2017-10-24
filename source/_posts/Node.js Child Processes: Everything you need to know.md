---
title: Node.js Child Processes: Everything you need to know
date: 2017-10-26
---

这篇文章是上周在Medium上看到的，[原文链接](https://medium.freecodecamp.org/node-js-child-processes-everything-you-need-to-know-e69498fe970a)，主要介绍了如何使用`spawn`, `exec`, `execFile`, `fork`，这里做一下简单总结。

单线程，非阻塞IO能让Node.js在单个进程下工作的很好。但是随着应用不断扩大规模，单个进程对增加的工作量就显得不太够用了。虽然Node.js是单线程的，但我们可以使用多进程来处理问题。

Node.js天生就是为通过多个节点来构建分布式应用而设计的，这也是它叫做**Node**的原因。

## The Child Processes Module
通过`child_process`模块可以快速创建子进程，而且这些子进程可以通过事件系统方便地通信。`child_process`能让我们通过在子进程中执行命令的方式获取操作系统的能力。我们可以控制子进程的输入流，监听子进程的输出流。也可以控制向子进程命令中传递的参数，还可以对子进程的输出做任何操作。这些都是基于Node.js的流(Stream)进行的。接下来介绍下`child_process`模块中四种不同的创建子进程的方法。

## Spawned Child Processes
`spawn`函数在新进程中执行命令，并且可以向其传入任意参数。下面是在子进程中执行`pwd`的例子：

```
const { spawn } = require('child_process');
const child = spawn('pwd');
```
我们从`child_process`模块出结构出`spawn`函数，并将操作系统命令作为第一个参数传入。返回值`child`就是子进程实例，子进程实例都实现了`EventEmitter API`，所以我们可以直接在子进程实例上注册事件比如注册`exit`事件：

```
child.on('exit', function (code, signal) {
  console.log('child process exited with ' +
              `code ${code} and signal ${signal}`);
});
```

`code`, `signal`分别是进程退出时的退出码和信号，正常退出时分别是0和`undefined`。除此之外，还有`disconnect`, `close`, `message`事件。`message`事件是最重要的一个，它在子进程使用`process.send()`发送消息时触发。从而进行父子进程通信。

 每个子进程都有三个标准`stdio`流，分别是`stdin`, `stdout`, `stderr`。当流关闭时，使用它们的子进程会触发`close`事件。`close`和`exit`不同，因为多个子进程共享同一个`stdio`流，所以一个子进程退出流不一定关闭。

 因为所有的流都是事件发射器，所以可以在这些流上监听不同的事件，其中`stdin`是可写流，`stdout`, `stderr`是可读流。监听可读流的`data`事件，可以拿到命名的输出或者错误：

 ```
child.stdout.on('data', (data) => {
  console.log(`child stdout:\n${data}`);
});

child.stderr.on('data', (data) => {
  console.error(`child stderr:\n${data}`);
});
 ```

`spawn`的第二个参数是一个数组，表示执行命令的参数。例如我们想在当前目录执行`find`命令并使用`-type f`参数时，可以这样：

```
const child = spawn('find', ['.', '-type', 'f']);
```

`stdin`作为可写流，我们可以用它向命令输入，最近单的方式是通过`pipe()`函数，因为父进程的`stdin`也是可写流，我们可以将它的输入传入子进程的`stdin`：

```
const { spawn } = require('child_process');

const child = spawn('wc');

process.stdin.pipe(child.stdin)

child.stdout.on('data', (data) => {
  console.log(`child stdout:\n${data}`);
});
```

上面的例子执行了`wc`命令，我们将父进程的`stdin` pipe 到子进程的`stdin`，启动程序后我们会进入输入模式，输入一些东西后敲` Ctrl+D`，输入将被传入命令:

![](https://cdn-images-1.medium.com/max/2000/1*s9dQY9GdgkkIf9zC1BL6Bg.gif)

>tips: `Ctrl + D`不是发送信号，是表示一个特殊的二进制值，表示`EOF`，在shell中表示退出当前shell。

## Shell Syntax and the exec function
默认情况下，`spawn`不会创建shell去执行命令，这让它比创建shell来执行命令的`exec`稍微高效些。`exec`方法还有一个很大的不同，它会缓冲所有的输出然后一次性将输出传给回调函数，而不是使用流。

下面是通过`exec`执行`find | wc`命令的例子：

```
const { exec } = require('child_process');

exec('find . -type f | wc -l', (err, stdout, stderr) => {
  if (err) {
    console.error(`exec error: ${err}`);
    return;
  }

  console.log(`Number of files ${stdout}`);
});

```

因为`exec`创建shell执行命令，所以我们可以直接使用shell的语法，比如这里的`pipe`。`exec`的回调分别是`err`, `stdout`, `stderr`。

我们可以让衍生的子进程继承父进程的IO对象。只要使用`stdio: 'inherit'`参数就可以了：

```
const child = spawn('find . -type f | wc -l', {
  stdio: 'inherit',
  shell: true
});

```

这样，子进程就可以继承父进程的IO对象，这是子进程的data事件处理函数会在父进程的`process.stdout`上触发，脚本的输出会立刻输出。并且通过`shell: true`，我们可以在`spawn`中也使用shell语法。两全其美。

## The execFile function
如果你想启动一个不需要shell来执行的文件，`execFile`是个好选择，但是在Windows上，`.bat`, `.cmd`这种需要shell的文件就无法执行了，可以通过`exec`和`shell: true`的`spawn`来执行。

## The *Sync function
这些就是创建子进程函数的同步版本，不用多说。

## The fork() function
`fork()`方法是`spawn`的变体，专门用来衍生node进程，和`spawn`最大的不同就是父子进程的通信方式不同，我们可以在fork出的进程中使用`process.send`向父进程传递信息：

```
// parent.js
const { fork } = require('child_process');

const forked = fork('child.js');

forked.on('message', (msg) => {
  console.log('Message from child', msg);
});

forked.send({ hello: 'world' });

```

```
process.on('message', (msg) => {
  console.log('Message from parent:', msg);
});

let counter = 0;

setInterval(() => {
  process.send({ counter: counter++ });
}, 1000);

```

## 总结
- Node.js中通过`child_process`创建子进程，主要有`spawn`, `exec`, `execFile`, `spawn`。
- `spawn`默认不创建shell来执行命令，但可以通过`shell: true`使用shell模式。
- 父子进程通过事件系统通信，`exec`则会缓冲子进程的输出，一次性传入父进程回调中，所以在输出内容较大时，应该考虑用`spawn`。
- `fork`函数专门用来创建node进程，且父子进程是建立的`IPC通道`，所以可以使用`process.send(message)`传递信息，messaeg在内部会被`JSON.stringify`处理。
- 此外`exec`, `execFile`和`spawn`还提供了同步版本。