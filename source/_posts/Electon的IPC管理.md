---
title: Electon的IPC管理
date: 2018-01-29
---

## Electon中的IPC

在Electron中，App的生命周期和页面渲染分别由主进程和渲染进程控制，想要通过页面操作主进程的一些函数并获取数据，就需要通过[IPC](https://zh.wikipedia.org/zh-hans/%E8%A1%8C%E7%A8%8B%E9%96%93%E9%80%9A%E8%A8%8A)进行通信。

Electron提供了[`ipcMain`](https://electronjs.org/docs/api/ipc-main)和[`ipcRenderer`](https://electronjs.org/docs/api/ipc-renderer)两个模块，分别用于主进程和渲染进程的事件监听与发送。在主进程使用`ipcMain`进行事件监听后，就可以在渲染进程中触发事件；同样，渲染进程中也可以监听主进程发来的事件。

下面是官方文档中的例子：

```js
// 主进程中
const {ipcMain} = require('electron');
ipcMain.on('asynchronous-message', (event, arg) => {
  console.log(arg);  // prints "ping"
  event.sender.send('asynchronous-reply', 'pong');  // 异步返回
});

ipcMain.on('synchronous-message', (event, arg) => {
  console.log(arg);  // prints "ping"
  event.returnValue = 'pong';  // 同步返回
});

```

```js
// 渲染进程中
const {ipcRenderer} = require('electron');
// 获取同步返回值
console.log(ipcRenderer.sendSync('synchronous-message', 'ping')); // prints "pong"

// 获取异步返回值
ipcRenderer.on('asynchronous-reply', (event, arg) => {
  console.log(arg); // prints "pong"
});
ipcRenderer.send('asynchronous-message', 'ping');
```

可以看出，Electron提供了同步和异步两种方式来获取数据，那么想象中的使用方式，就是对直接可以获取数据的函数使用，需要等待或者多步操作的就使用异步的形式来获取。然而事实上这么做却可能导致一些问题。

## 同步获取的问题

首先来看看同步消息的通信过程。

```js
// 主进程
ipcMain.on('synchronous-message', (event, arg) => {
  // 各种操作...
  
  event.returnValue = 'pong';
});
```
```js
// 渲染进程
const value = ipcRenderer.sendSync('synchronous-message', 'ping');
```

要获取同步消息的返回值，就必须在主进程中给`event.returnValue`赋值，这时渲染进程中的`ipcRenderer.sendSync`函数才会产生返回值，并结束函数。

但在`returnValue`被赋值之前，渲染进程就会被**阻塞**，整个UI都会卡在那。这就是同步获取数据最大的问题，一旦出现了任何意外导致`returnValue`赋值失败，就会将界面卡住，用户将完全无法操作。即使将`sendSync`放入Promis中，也无法阻止UI被阻塞的悲惨命运。所以说，Electron的同步消息函数是一个非常糟糕的API。

## 使用异步通信的问题

当然，完全可以全靠异步的形式进行通信。但同样也会出现一些问题。

首先，使用异步的通信时，需要在主进程和渲染进程中都要定义事件并监听，也就是说，一个事件需要有两个事件名才能完成一次操作（当然用同一个事件名也行，只是个人感觉这样没那么直观）。

异步通信的例子：

```js
// 主进程
ipcMain.on('event', (e, data) => e.sender.send('event-response', data));
```

```js
// 渲染进程
const foo = (data) => new Promise((resolve) => {
  ipcRenderer.send('event', data);
  ipcRenderer.on('event-response', (e, data) => resolve(data));
}
```

定义事件名只是个小问题，完全可以通过代码规范的约定来解决。实际上这样的方式可能还会出现一个问题，就是一个事件可能会同时有多个监听者，这是就无法确认每一次触发的事件应该只交给哪个监听器进行处理，并且每次有响应时，渲染进程中的每个监听器都会被触发。

## 定义协议

为了解决请求和响应匹配的问题，就需要定义一个简单的通信协议。

首先可以将通信分为两种方式，一种是单次请求并获取相应的，一种是需要持续坚挺的。

### 单次响应

对于单次响应的通信，可以确定一个请求对应一个响应。当响应到达的时候，也只触发对应的那一个监听器，这就需要在通信中增加一个唯一标识符。

在请求时，先在渲染进程计算出一个id，并以`{id, data}`的形式，将事件`event`发送给主进程，主进程收到后记录下id，然后在处理完成后，触发一个`${event}_res_${id}`的事件，客户端单次监听这个事件并处理。

```js
// 主进程
const listen = (eventName, handler) => {
  ipcMain.on(eventName, async (e, request) => {
    const { id, data } = request;
    const response = { code: 200 };
    try {
      response.data = await handler(data);
    } catch (err) {
      response.code = err.code || 500;
      response.data = { message: err.message || 'Main process error.' };
    }
    e.sender.send(`${eventName}_res_${id}`, response);
  });
};
```

```js
// 渲染进程
const sendEvent = (eventName, options = {}) => {
  const { data } = options;

  const id = uuid.v1();
  const responseEvent = `${eventName}_res_${id}`;

  return new Promise((resolve, reject) => {
    // 这里使用 once 监听，响应到达后就销毁
    ipcRenderer.once(responseEvent, (event, response) => {
      if (response.code === 200) {
        resolve(response.data);
      } else {
        reject(response.data);
      }
    });
    // 监听建立之后再发送事件，稳一点
    ipcRenderer.send(eventName, { id, data });
  });
};
```

整个流程看起来就是这样的
```
-------Render-------------------------Main-------
         |                              |
listen() |                              |
         |-- emit(event, {id, data}) -->|
         |                              | handle event
         |<---- emit(resEvent, data) ---|
         |                              |
```

这样，一次IPC通信在外面看上去就像一个异步网络请求一样了。这里还可以约定一些数据格式或者事件名格式之类的，都是小事，这里不讨论。

如果再进行其他一次封装和判断，就可以在不同环境使用不同的函数，来轻松的在Electron和Web环境下进行切换。

### 持续响应

需要持续监听的事件有点类似WebSocket，流程大概就是，渲染进程通知主进程开始任务，并持续返回带有特定id的事件，渲染进程进行监听。在停止时，清理两个线程中的监听器。

看上去就像这样：

```
--------Render----------------------------Main--------
          |                                 |
          |-- connect(event, {id, data}) -->|
listeners |                                 | register
          |------------- start ------------>|
          |                                 | start handler
          |<----- emit(idEvent, data) ------|
          |<-----          ...         -----|
          |<----- emit(idEvent, data) ------|
          |                                 |
          |------------- stop ------------->|
          |                                 | stop handler
          |---------- disconnect ---------->|
   clean  |                                 | clean
          |                                 |
```

首先渲染进程发送一个`connect`事件告知主进程事件和id，并对预计的事件进行监听；主进程在记录id，并监听`${event}_${id}_start`、`${event}_${id}_stop`、`${event}_${id}_disconnect`事件，给渲染进程一些操作空间。当收到开始事件时，就执行处理函数，并持续触发响应事件；收到结束事件就停止操作；断开连接时就清理监听器。

以下为主要部分的代码：

```js
// 主进程
export default class LongEventServer {
  static listen(eventName) {
    return new Promise((resolve) => {
      ipcMain.on(eventName, (e, args) => {
        const {id, data} = args;
        const server = new LongEventServer({
          eventName, id, data,
          sender: e.sender
        });
        ipcMain.on(server._getEventName('start'), () => server.onStart(server.startData));
        ipcMain.on(server._getEventName('stop'), () => server.onStop());
        ipcMain.on(server._getEventName('disconnect'), () => {
          ['start', 'stop', 'disconnect'].forEach(el => ipcMain.removeAllListeners(server._getEventName(el)));
          server.onDisconnect();
        });
        resolve(server);
      })
    });
  }
  
  _getEventName(event) {
    return `${this.eventName}_${this.id}_${event}`;
  }
  
  // constructor/emit etc.
}
```

```js
// 渲染进程
class LongEventClient {
  constructor(id, eventName, data) {
    this.id = id;
    this.eventName = eventName;
    this.startData = data;
  }

  _getEventName(event) {
    return `${this.eventName}_${this.id}_${event}`;
  }
  
  // other function like on(event) etc.
}

const sendLongEvent = (eventName, options = {}) => {
  const { data } = options;
  const id = uuid.v1();
  const client =  new LongEventClient(id, eventName, data);
  client.connect();
  return client;
};

// client.on('xxx') ...
// client.start()
```

这样就能愉快的进行异步通信了。
