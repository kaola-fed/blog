<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

# Node Streams

## 为什么要使用流？

### 流的简介

计算机世界里引入流（Stream）是一个很形象的概念，对比现实生活中，从湖、河或者池塘等引出水流，以用于日常生产，而且引流的过程会有个管道的工具，相应的概念（Pipe）也被引入计算机世界。早在*nix类系统中，流（Stream）通过“|”管道（Pipe）引入系统，使用流的根本原因是设备的处理能力无法满足需求，比如有限的内存处理海量的数据，所以采用流的方式来分批处理。典型的，没有人会直接去cat一个几个GB的日志文件，而会采用cat|less的方式去查看需要的信息。

### Node中的流

Node中引入流是同样的目的，使用有限的内存处理海量的数据。
Node中通过 `Stream` 来实现流的管理，主要实现有四种流 `readable`，`writable`，`duplex`，`transform`，对应的管道实现是 `pipe()` 函数。`Stream` 是Node的核心模块之一，从 `EventEmitter` 派生出来，是许多其它模块的基础, 如 [Http Server的请求](https://nodejs.org/dist/latest-v8.x/docs/api/http.html#http_class_http_incomingmessage)，[process.stdout](https://nodejs.org/dist/latest-v8.x/docs/api/process.html#process_process_stdout) 等。

四种类型的流的分别如下：

- [Readable](https://nodejs.org/dist/latest-v8.x/docs/api/stream.html#stream_class_stream_readable): 可以读取数据的流（比如 [fs.createReadStream()](https://nodejs.org/dist/latest-v8.x/docs/api/fs.html#fs_fs_createreadstream_path_options) ）

- [Writable](https://nodejs.org/dist/latest-v8.x/docs/api/stream.html#stream_class_stream_writable): 可以写入数据的流（比如 [fs.createWriteStream()](https://nodejs.org/dist/latest-v8.x/docs/api/fs.html#fs_fs_createwritestream_path_options) ）

- [Duplex](https://nodejs.org/dist/latest-v8.x/docs/api/stream.html#stream_class_stream_duplex): 既可以读出又可以写入数据的流（比如 [net.Socket](https://nodejs.org/dist/latest-v8.x/docs/api/net.html#net_class_net_socket) ）

- [Transform](https://nodejs.org/dist/latest-v8.x/docs/api/stream.html#stream_class_stream_transform)：是一种特殊的 `Duplex` 流，可以在写入和读出的过程中对流进行加工（比如 [zlib.createDeflate()](https://nodejs.org/dist/latest-v8.x/docs/api/zlib.html#zlib_zlib_createdeflate_options) ）

> **小插曲:** `Transform` 源码中的注释，作者对 `加工` 的描述不满足于仅仅为 `filter`，因为 `Transform` 输出的流相对输入的流不是简单的、同步的过滤或变换，可能存在复杂的多对一或一对多的异步映射关系。

### `objectMode`

Node中流的数据类型默认只能是 `String` 和 `Buffer（或Uint8Array ）`，可以通过设置 `objectMode = true` 来使用其它类型的数据。不推荐改变已经创建流的 `objectMode` 属性。

> `objectMode` 会影响流内部缓存中数据长度（length）的计算，对于普通流（`objectMode = false`），数据长度的计算即为内部缓存（buffer）中每个entry的长度之和，对于对象模式的流（`objectMode = true`），数据长度即为entry的数目。

下面对四种类型的流一一介绍。

## 可读流 Readable

`Readable` 流是几种类型流之中最复杂的一种，常见的典型可读流如 [HTTP responses, on the client](https://nodejs.org/dist/latest-v8.x/docs/api/http.html#http_class_http_incomingmessage), [HTTP requests, on the server](https://nodejs.org/dist/latest-v8.x/docs/api/http.html#http_class_http_incomingmessage), [fs read streams](https://nodejs.org/dist/latest-v8.x/docs/api/fs.html#fs_class_fs_readstream)等。

### 两种模式：流动模式（flowing）和暂停模式（paused）

可读流工作在两种模式下，内部使用其状态属性 `state.flowing` 的不同值来标识，两种模式的区别主要在于 `是否需要手动调用 Readable.prototype.read(n) 函数来读取数据`。

`flowing` 不同的属性对应可读流的不同模式如下

- `flowing` 为 `null`，初始状态，此时流处于 `暂停模式（paused）`
- `flowing` 为 `true`，此时流处于 `流动模式（flowing）`
- `flowing` 为 `false`，此时流处于 `暂停模式（paused）`

所有可读流初始状态为 `暂停模式`，可以通过如下方式之一切换到 `流动模式`:

- 添加 `data` 事件处理函数
- 调用 `stream.resume()` 方法
- 调用 `stream.pipe()` 方法将数据导向一个可写流

可读流可以通过下面方式之一切换回 `暂停模式`：

- 如果没有被导向任何可写流，调用 `stream.pause()` 方法来暂停可读流
- 如果有被导向可写流，需要删除 `data` 事件处理函数，并删除对所有可写流的导向（调用 `stream.unpipe()` 方法）来暂停可读流

> 出于向前兼容的原因，删除 `data` 事件处理函数并不会自动暂停流。同样的，如果已被导向其他可写流，调用 `stream.pause()` 方法并不能保证可读流一直处于暂停模式，比如当下游可写流抛出 `drain` 事件已请求更多数据时会恢复流动模式。

> 如果一个可读流处于 `流动模式`，但是没有绑定 `data` 事件处理函数来处理取到数据，数据将会丢失。 

### `option.read` 和 `_read` 方法

`_read` 是一个抽象函数，必须实现，是从底层读取数据的逻辑，即生成数据的逻辑。

```javascrtipt
// abstract method. to be overridden in specific implementation classes.
// ...
Readable.prototype._read = function(n) {
  this.emit('error', new Error('_read() is not implemented'));
};
```

通过 `option` 传递的 `read` 函数参数最终也是赋值给内部的 `_read`。

外部通过可读流实例读取数据 `stream.read(n)`，如果内部缓存存有足够的数据，则直接返回数据，否则通过 `_read` 函数进行底层数据获取（同步或异步），最终将数据加入缓存或直接抛出。

### `data` 事件

添加 `data` 事件处理函数会将不是被显示暂停的可读流（即调用 `stream.pause()` 暂停的可读流 ）切换为 `流动模式`。如下面源码所展示，对于非暂停模式的流，绑定 `data` 事件，即会调用 `resume()` 方法，开始流动模式。

```javascript
  Readable.prototype.on = function(ev, fn) {
    const res = Stream.prototype.on.call(this, ev, fn);

    if (ev === 'data') {
      // Start flowing on next tick if stream isn't explicitly paused
      // 只有非显式的pause，才可以通过绑定data事件resume
      if (this._readableState.flowing !== false)
        this.resume();
    } ...
    ...
  };
```

`resume()` 方法中会设置 `state.flowing = true`，并触发 `flow` 方法，`flow` 中会源源不断地触发 `stream.read()`，

```javascript
Readable.prototype.resume = function() {
  var state = this._readableState;
  if (!state.flowing) {
    debug('resume');
    state.flowing = true;
    resume(this, state); // will call flow method
  }
  return this;
};
... 
function flow(stream) {
  const state = stream._readableState;
  debug('flow', state.flowing);
  while (state.flowing && stream.read() !== null);
}
```

进一步 `read()` 方法中可能会触发 `push(chunk)`, `push(chunk)` 方法又可能会触发 `flow` 和 `read`，即形成流动回路。

```
graph TD

flow(("flow()"))--"返回数据时一直调用"-->read(("read()"))
read--"需要底层数据时<br/>同步或异步调用"-->push(("push()"))
push--"立刻需要数据时<br/>emit('data', chunk)<br/>read(0);"-->read
push--"不是立刻需要数据时<br/>存入缓存<br/>emitReadable()时异步flow()"-->flow

```

### `readable` 事件

当可读流内部缓存中有可用的数据时会抛出 `readable` 事件。

外部流使用者通过 `stream.read(n)` 向流请求数据时，如果返回 `null`，流内部就会设置状态 `state.needReadable = true` 标记需要触发 `readable` 事件，然后去底层取数据，待数据被取回后，通过 `readable` 告知外部使用者数据已可用，外部即可以通过 `read` 方法再次请求数据。

首次监听 `readable` 事件时，会触发一次 `read(0)` 的调用，可能会进一步引起 `_read` 和 `push` 方法的调用，从而会启动循环，将数据取到内部缓存中。

当流中数据被消耗尽，并在 `end` 事件触发之前，也会触发一次 `readable` 事件。

所以 `readable` 事件意味着可读流中新的信息：有数据可用或者流已经被耗尽。

对于暂停模式可读流的消耗，典型的方法即是组合使用 `readable` 事件和 `read` 方法。

```javascript
const Readable = require('stream').Readable

// 底层数据
const dataSource = ['a', 'b', 'c']

const readable = Readable()
readable._read = function () {
  process.nextTick(() => {
    if (dataSource.length) {
      this.push(dataSource.shift())
    } else {
      this.push(null)
    }
  })
}

readable.pause()
readable.on('data', data => process.stdout.write('\ndata: ' + data))

readable.on('readable', function () {
  while (null !== readable.read()) ;;
})
```

输出：

```javascript
data: a
data: b
data: c
```

### `end` 事件

可读流执行 `stream._read()` 方法从底层获取数据，获取数据逻辑中约定调用 `push(null)` 就意味着底层数据已被取完，此时可读流的状态 `state` 会设置 `state.ended = true`。 进一步当流缓存中数据被取尽，即 `state.length = 0`，则意味流中所有的数据都被消耗了。此后再执行 `read(n)` 便会触发 `end` 事件，并且只会触发一次。此后不能再向缓存添加数据，即 `stream.push(chunk)` 会报错。

### `_readableState`

可读流内部的状态对象

### `objectMode`

对于可读流来说，`push(data)` 时，`data` 只能是 `String` 和 `Buffer（或Uint8Array ）` 类型，而消耗时 `data` 事件输出的数据都是 `Buffer` 类型。对于可写流来说，`write(data)` 时，`data` 只能是 `String` 和 `Buffer（或Uint8Array ）` 类型，`_write(data)` 调用时传进来的 `data` 都是 `Buffer` 类型。

也就是说，流中的数据默认情况下都是 `Buffer` 类型。产生的数据一放入流中，便转成 `Buffer` 被消耗；写入的数据在传给底层写逻辑时，也被转成 `Buffer` 类型。

可读流未设置 `objectMode` 时：

```javascript
const Readable = require('stream').Readable;

const readable = Readable();

readable.push('a');
readable.push('b');
readable.push(null);

readable.on('data', data => console.log(data));
```

输出：

```javascript
<Buffer 61>
<Buffer 62>
```

可读流设置 `objectMode = true` 后：

```javascript
const Readable = require('stream').Readable;

const readable = Readable({ objectMode: true });

readable.push('a');
readable.push('b');
readable.push({});
readable.push(null);

readable.on('data', data => console.log(data));
```

输出：

```javascript
a
b
{}
```

### 内部缓存 `BufferList`

可读流内部缓存 `buffer` 的结构如下

```javascript
{
  head: null,
  tail：null,
  length：0
}
```
  
`buffer` 中每个 `entry` 的结构

```javascript
{
  data: v,
  next: null
}
```

源码中的 `push` 操作

```javascript
push(v) {
  const entry = { data: v, next: null };
  if (this.length > 0)
    this.tail.next = entry;
  else
    this.head = entry;
  this.tail = entry;
  ++this.length;
}

```

### 示例

```javascript
const Readable = require('stream').Readable;

const source = ['a', 'b', 'c'];

const readable = new Readable({
  read: function() {
    if (source.length) {
      this.push(source.shift());
    } else {
      this.push(null);
    }
  }
});

readable.on('data', data => process.stdout.write('data: ' + data + '\n'));
readable.on('end', () => process.stdout.write('end'));
```

输出

```javascript
data: a
data: b
data: c
end
```

`readable` 事件可以取得同样的效果

```javascript
const Readable = require('stream').Readable;

const source = ['a', 'b', 'c'];

const readable = new Readable({
  read: function() {
    if (source.length) {
      this.push(source.shift());
    } else {
      this.push(null);
    }
  }
});

readable.on('readable', function() {
  var data = null;
  while(data = readable.read(1)) {
    process.stdout.write('data: ' + data + '\n');
  }
});
readable.on('end', () => process.stdout.write('end'));
```


## 可写流 Writable

### `option.write 和 _write` 方法

同可读流的 `_read` 类似，`_write` 方法是可写流必须实现的方法，否则会报错。

通过 `writable.write()` 方法向可写流写入数据，当数据被处理结束后需要调用最后的回调参数，标识当前数据已经处理完毕。当写入过程中出错时，错误可以选择性的作为回调函数的第一个参数传入。更保险的方法，是监听 `error` 事件。

当内部缓存没有满时（即 `buffer.length` 小于 `highWaterMark`），`write` 方法会返回 `true`；否则会返回 `false`，此时，在 `drain` 事件抛出之前，不应该进一步向可写流写入数据。

> 当 `write()` 返回 `false` 时，尽管允许继续写入数据，但Node将会缓存所有写入的数据直到最大可用内存被耗尽，会导致程序直接崩溃。即使在没有崩溃之前，高内存使用率也会导致垃圾回收器的性能很差以及高RSS率（即使内存不被使用，也不会释放掉）。对于TCP sockets而言，如果远端连接的终端始终没有读取数据，则其不会触发 `drain` 事件，继续一直写入数据可能会导致远程可利用的漏洞。

如果数据是按需进行加载，推荐将数据源封装为可读流 `Readable`，并使用 `stream.pipe()` 方法；否则，建议要遵循 `背压原则`，并使用 `drain` 事件来避免内存问题。

### `drain` 事件

当可写流内部缓存满了之后，继续写入数据 `writable.write(chunk)` 会返回 `false`，当缓存数据被处理并写入底层后，可写流会抛出 `drain` 事件通知上游可以继续写入数据。

### `finish` 事件

`finish` 会在 `writable.end()` 方法调用之后抛出，此时所有缓存数据都已经被写入底层。

> 为什么可读流的结束事件为 `end`，而可写流的事件为 `finish` ? 主要是对于 `Duplex流`（既可读又可写并且两种流完全独立的流），对外抛出结束事件需要区分，所以取了不同的名字，否则无法区分是哪一种流结束了。

### 示例

```javascript
const Writable = require('stream').Writable;

const buffer = [];

const writable = Writable({
  write: function(data, enc, next) {
    buffer.push(data);
    process.nextTick(next);
  }
});

writable.on('finish', () => process.stdout.write(buffer.toString()));

writable.write('a');
writable.write('b');
writable.end('c');
```

输出：

```javascript
a,b,c
```


## Duplex流

`Duplex流` 是同时继承了 `可读流Readable` 和 `可写流Writable` 的一类流。所以，一个 `Duplex流` 对象既可当成可读流来使用（需要实现 `_read方法`），也可当成可写流来使用（需要实现 `_write` 方法），可以通过选项配置成为只可读、只可写等。

### 示例

```javascript
const Duplex = require('stream').Duplex;

const source = [1, 2, 3];
const buffer = [];

const duplex = Duplex({
  // read
  read: function() {
    if (source.length) {
      this.push(source.shift() + '');
    } else {
      this.push(null);
    }
  },

  // write
  write: function(data, enc, next) {
    buffer.push(data);
    process.nextTick(next);
  }
});


duplex.on('data', (data) => process.stdout.write('data: ' + data + '\n'));
duplex.on('end', () => process.stdout.write('end\n'));
duplex.on('finish', () => process.stdout.write(buffer.toString() + '\n'));

duplex.write('a');
duplex.write('b');
duplex.end('c');
```

输出：

```javascript
data: 1
data: 2
data: 3
end
a,b,c
```


## Transform流

`Tranform流` 继承自 `Duplex流`。但与 `Duplex流` 不同的是，`Duplex流` 的可读流和可写流是完全独立的，各不相干，而 `Transform流` 中可写端写入的数据经变换后会自动添加到可读端，其内部已经实现了 `_read` 和 `_write` 方法，其要求用户实现一个 `_transform` 方法。

由于 `Transform流` 中可读流和可写流是直接连接的，大多数情况来说两端的效率是对等的，所以一般其内部不存在 `背压` 的问题，`背压` 问题的源头主要是外部生产者和消费者速度导致的。

### 示例

```javascript
const Transform = require('stream').Transform;
const offset = 13;
const transform = Transform({
  transform: function(buf, enc, next) {
    const res = buf.toString().split('').map(c => {
      let code = c.charCodeAt(0);
      if (c >= 'a' && c <= 'z') {
        code += offset
        if (code > 'z'.charCodeAt(0)) {
          code -= 26;
        }
      } else if (c >= 'A' && c <= 'Z') {
        code += offset;
        if (code > 'Z'.charCodeAt(0)) {
          code -= 26;
        }
      }

      return String.fromCharCode(code);
    }).join('');

    this.push(res);
    process.nextTick(next);
  }
});

transform.on('data', data => process.stdout.write(data));
transform.write('hello, ');
transform.write('world!');
transform.end();
```

输出：

```javascript
uryyb, jbeyq!
```


## 背压机制 back pressure

考虑下面的例子：

```javascript
const fs = require('fs');
fs.createReadStream(file).on('data', fileProcessor);
```

监听 `data` 事件后文件中的内容便立即开始源源不断地传给 `fileProcessor()`。
如果 `fileProcessor()` 处理数据较慢，就需要缓存来不及处理的数据 `data`，会占用大量内存。

理想的情况是下游消耗一份数据，上游才生产一份新数据，这样整体的内存使用就能保持在一个水平。
可读流 `Readable` 提供的 `pipe` 接口，正是用来实现这个功能。

### `pipe()` 接口

可读流中的 `pipe()` 接口用来将上游（可读流）和下游（可写流）连接起来，接口内部会自动调用上下游的接口对数据的读写进行控制，形成“背压反馈”的效果。

`pipe` 接口的核心思想：

```javascript
readable.on('data', function (data) {
  if (false === writable.write(data)) {
    readable.pause()
  }
});

writable.on('drain', function () {
  readable.resume()
});
...
return writable; // 实现链式调用
```

其中下游可写流队列 `writable`，当内部写缓存 `buffer` 达到阈值 `state.highWaterMark` 时，
继续写入数据 `write(data)` 会返回 `false`，否则返回 `true`。因此，上游可读流 `readable` 可以根据 `write(data) === false` 切换到暂停模式，此后将不再触发 `data` 事件；当下游 `writable` 将内部缓存清空后，会触发一个 `drain` 事件，此时上游 `readable` 再调用 `resume()` 开始流动模式，继续触发 `data` 事件。

#### 示例：

```javascript
const stream = require('stream');

let c = 0;
let pushable = true;
const readable = stream.Readable({
  highWaterMark: 3,
  read: function() {
    process.nextTick(() => {
      const data = c < 6 ? String.fromCharCode(c + 65) : null;
      pushable = this.push(data);
      process.stdout.write('readable: push' + ++c + ' ' + data + ', further pushable is ' + pushable + '\n');
    })
  }
});

const writable = stream.Writable({
  highWaterMark: 2,
  write: function(chunk, enc, next) {
    process.stdout.write('writable: data is ' + chunk + '\n');
    // process.nextTick(next);
  }
});

readable.on('data', (data) => {
    process.stdout.write('readable: data is ' + data + ', paused is ' + readable.isPaused() + '\n');
    process.nextTick(() => {
        process.stdout.write('writable: length is ' + writable._writableState.length + ', writing is ' + writable._writableState.writing + '\n');
    });
});

readable.on('pause', (data) => {
    process.stdout.write('readable: paused is ' + readable.isPaused() + '\n');
});


readable.pipe(writable);
```

输出：

```javascript
readable: data is A, paused is false
writable: data is A
readable: push1 A, further pushable is true
writable: length is 1, writing is true  // cannot be further written here!!!
readable: data is B, paused is false
readable: paused is true // paused here due to writable back pressure
readable: push2 B, further pushable is true
writable: length is 2, writing is true
readable: push3 C, further pushable is true
readable: push4 D, further pushable is true
readable: push5 E, further pushable is false // return false due to readable back pressure
```

虽然上游一共有6个数据 `ABCDEF` 可以生产，但实际只生产了5个 `ABCDE`。
因为第一个数据 `A` 迟迟未能写完（`writable` 一直未调用 `next()`，导致一直是 `writing` 状态），所以后面通过 `write` 方法（`pipe` 里面调用的）添加进来的数据便被内部缓存起来，而不会真正写到底层（未调用 `_write`）。下游的缓存队列到达2时，`write` 返回 `false`，`背压机制` 会促使上游切换至暂停模式。此时下游保存了 `AB`。由于 `Readable` 的 `highWaterMark === 3`，最多只能缓存3个数据，所以上游 `push('E')` 时会返回 `false`，已经不能进一步从底层读入数据了，此时上游保存了 `CDE`。

可读流和可写流内部缓存加起来的长度一共为5，所以一共就生产了`ABCDE` 5个数据。

## 参考文献

- [Node Stream API](https://nodejs.org/dist/latest-v8.x/docs/api/stream.html)
- [Node.js Stream - 基础篇](https://tech.meituan.com/stream-basics.html)
- [Node.js Stream - 进阶篇](https://tech.meituan.com/stream-internals.html)

by kaolafed/Hong
