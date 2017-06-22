---
title: websocket homework之聊天室优化
date: 2017-06-06
---

### homework?
为什么叫homework？来源于[socket.io的聊天室例子](https://socket.io/get-started/chat/)，例子中介绍了用socket.io搭建简单聊天室的相关步骤，在例子结尾处留下了homework，即对该聊天室的几点优化建议。本文在最后也会给出这些优化的具体实现代码，但是一步一步来，我们先了解实现聊天室的原理和相关技术。

<!-- more -->

### websocket简介
> WebSocket一种在单个 TCP 连接上进行全双工通讯的协议，允许服务端主动向客户端推送数据。在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

websocket是html5中推出的协议，相比于http协议，它有如下优点：
- 更强的实时性。服务器可以随时主动推数据给客户端，并且延迟少。
- 保持连接状态。Websocket需要先创建连接，这就使得其成为一种有状态的协议，之后通信时可以省略部分状态信息，也因此减少了数据的传输。
- 没有同源限制。客户端可以与任意服务器通信。
- 更好的二进制支持。Websocket定义了二进制帧，可以更轻松地处理二进制内容。
- 更好的压缩效果。Websocket在适当的扩展支持下，可以沿用之前内容的上下文，在传递类似的数据时，可以显著地提高压缩率

再用[阮一峰博客](http://www.ruanyifeng.com/blog/2017/05/websocket.html)中的两张图补充对比：
![对比](http://www.ruanyifeng.com/blogimg/asset/2017/bg2017051502.png)
![对比](http://www.ruanyifeng.com/blogimg/asset/2017/bg2017051503.jpg)

ws是websocket的协议标识符，（如果加密，则为wss）。

### websocket用法
#### 客户端代码
``` javascript
var ws = new WebSocket("ws://echo.websocket.org"); //通过WebSocket构造函数打开连接
ws.onopen = function(){//连接成功后回调
    ws.send("Test!"); //向服务端发送消息
}; 
ws.onmessage = function(evt){//接收服务端数据后回调
    console.log(evt.data);
    ws.close();//关闭连接
}; 
ws.onclose = function(evt){//连接关闭后回调
    console.log("WebSocketClosed!");
}; 
ws.onerror = function(evt){//报错时回调
    console.log("WebSocketError!");
};
```

#### 服务端代码
websocket服务器被[多种语言实现](https://en.wikipedia.org/wiki/Comparison_of_WebSocket_implementations)，下文会介绍其中的一种node实现。

### 浏览器兼容性
![兼容性](https://haitao.nos.netease.com/0740dfb0-e891-4600-bdf3-a1c2737b157e.png)

### socket.io
socket.io不仅仅实现了websocket的服务端代码，而且封装了websocket的客服端代码，在不支持websocket的客户端中用轮询机制实现相似的效果。我们只需要使用socket.io提供的通用接口即可。下文的聊天室例子也是运用了socket.io。

### 实现一个多人聊天室

#### Improvements
代码主要是对[socket.io官方聊天室](https://socket.io/get-started/chat/)例子的完善，有以下几点：

1. Broadcast a message to connected users when someone connects or disconnects
1. Add support for nicknames
1. Don’t send the same message to the user that sent it himself. Instead, append the message directly as soon as he presses enter.
1. Add “{user} is typing” functionality
1. Show who’s online
1. Add private messaging

#### 源码
> [github repo](https://github.com/Martin0417/websocket-chatroom)

- app.js：集成socket.io、express，并启动服务
- handleSocket.js：处理websocket连接
- public：客户端代码

#### 启动
1. `npm i`
2. `npm start`

#### 聊天效果图
![效果图](https://haitao.nos.netease.com/d268fa64-51c1-4514-a9a9-81f824d7fc41.png)


### 谢谢阅读！
by小马哥
