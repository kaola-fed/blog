---
title: Koa之中间件执行解析
date: 2017-10-25
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

> 本文是基于Koa@1.4.0来讲解的

从14年开始接触 Koa，翻过源码，写过文章，后面也陆续用 Koa 做过一些项目，但一直都没系统性的学习总结过。今天想通过这篇文章，给大家介绍下 Koa 中中间件从加载到执行的整个过程剖析。如有不准确的地方还忘指出。

### 发展历程

我们先来看下 Koa 的整个发展历程，每个里程碑都发生了哪些变化

* 2013.12 First Commit
* 2015.08 发布Koa v1版本
* 2015.10 发布Koa v2-alpha.1版本，用 ES6 重写了代码并且更新了中间件支持 async 和 await
* 2017.02 发布Koa v2版本，弃用v1版本中的 generator 中间件，如想继续使用，须用 koa-covert 来做适配。明确提出在 v3 中将删除 generator 中间件的支持

纵观整个发展历程，我们似乎发现个规律，Koa 每两年发布一个大版本。那么，我们是不是可以期待下2019年迎来 v3 呢？

### 知识点回顾

在正式分析 Koa 源码之前，我们还需要一些其他知识的储备。这里我们就来简单回顾下 Generator 函数、Co & Promise 的使用。如果想深入学习的，可以网上找找相关的资料来学习。如果你已熟练掌握它们的用法，可以直接跳过下面内容，继续阅读后面内容。

> Generator 函数

generator 是 ES6 中处理异步编程的解决方案，我们通过一个简单的例子，来回顾下它的用法。

```js
function* gen1() {
  console.log('gen1 start');
  yield 1;
  var value = yield 2;
  console.log(value);
  yield* gen2();
  yield 4;
  console.log('gen1 end');
}

function* gen2() {
  console.log('gen2 start');
  yield 3;
  console.log('gen2 end');
}

var g = gen1();
g.next(); 			// gen1 start {value: 1, done: false}
g.next(); 			// {value: 2, done: false}
g.next('delegating to gen2');   // delegating to gen2 gen2 start  {value: 3, done: false}
g.next(); 			// gen2 end {value: 4, done: false}
g.next(); 			// gen1 end {value: undefined, done: true}
```
看完例子，我们需要注意几个点：

1. 直接调用 generator function 并没有真正开始执行函数，只有通过 next 方法，才开始执行
2. 每次调用 next 方法，到 yield 右侧暂停，并不会去执行 yield 左侧的赋值操作
3. 通过对 next 方法传入参数，可以给 yield 左侧变量赋值
4. 通过 yield* 的方式，可以代理到 gen2 内部，继续执行 gen2 中的内容，这个称之为 delegating yield
5. 当执行完所有 yield 后，再次调用 next，返回 value 为 undefined， done 为true

> Co & Promise 的使用

通过上面的例子我们可以看到，通过不断的调用 next 方法，可以执行完整个 generator function。那有没有什么解决方案，可以自动的调用这些 next 方法呢？答案就是 co。另外，Promise 作为 ES6 中提供的解决异步编程的方案，可以有效的规避 callback hell 的产生。这里同样通过一个小例子，来回顾下 co 的实现原理以及 Promise 的用法。

```js
var co = require('co');

co(function* () {
  var a = Promise.resolve(1);
  var b = Promise.resolve(2);
  var c = Promise.resolve(3);
  var res = yield [a, b, c];
  console.log(res);
  // => [1, 2, 3]
}).catch(onerror);

function onerror(err) {
  console.log(err.stack);
}
```
这个例子 `res` 直接输出了 `[1, 2, 3]`。那我们来看下，co 内部到底做了什么操作，直接上代码

```js
function co(gen) {
  // xxx
  return new Promise(function(resolve, reject) {
    if (typeof gen === 'function') gen = gen.apply(ctx, args);
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    onFulfilled();

    function onFulfilled(res) {
      var ret;
      try {
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    function next(ret) {
      if (ret.done) return resolve(ret.value);
      var value = toPromise.call(ctx, ret.value);
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
  }

  // xxx
}

function toPromise(obj) {
  if (!obj) return obj;
  if (isPromise(obj)) return obj;
  if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
  if ('function' == typeof obj) return thunkToPromise.call(this, obj);
  if (Array.isArray(obj)) return arrayToPromise.call(this, obj);
  if (isObject(obj)) return objectToPromise.call(this, obj);
  return obj;
}

function arrayToPromise(obj) {
  return Promise.all(obj.map(toPromise, this));
}
```
可以看到 co 最终返回的是一个 Promise 对象，所以才有了例子中的 catch 方法，这个先不管。我们来看下 Promise 内部的具体实现

首先判断这个 `gen` 是不是一个 `function`，如果是就直接调用；再通过判断是否有 next 方法，来判断是不是一个 generator 实例，如果不是就直接 `resolve` 返回；在函数 `onFulfilled` 内部**第一次**调用 `gen.next` 方法，ret 的值 `{value: 数组，done: false}`，再把 ret 传给内部的 next 方法；因为我们知道 `ret.value` 的值是一个数组，所以我们直接来看 `arrayToPromise` 这个方法。

大家先回忆下 Promise 的用法，`Promise.resolve()` 可以创建一个 Promise 实例，`Promise.all()` 用于将多个 Promise 实例，包装成一个新的 Promise 实例。只有数组中的每个实例状态都变成 `fulfilled` 的时候，`Promise.all` 的状态才会是 `fulfilled`，此时数组中每个实例的返回值组成一个数组，传递给 `all` 的回调函数。

因为我们例子中用的都是 `Promise.resolve()`，所以我们 `Promise.all` 的状态肯定是 `fulfilled`。回头看 `next` 方法中的

```js
if (value && isPromise(value)) 
	return value.then(onFulfilled, onRejected);
```
此时的 value 值是 `Promise.all` 包装成的新实例，且状态是 `fulfilled`，调用 `then` 的第一个回调函数 `onFulfilled`，参数是有各个子实例返回值组成的数组，也就是 `[1, 2, 3]`。所以到**第二次**调用 `gen.next` 方法的时候，`res` 的值是数组 `[1, 2, 3]`。通过前面 generator 中的回顾，我们知道，传给 `gen.next` 的参数会赋值给外部 yield 左侧的变量，所以上述例子中，`res` 最终输出 `[1, 2, 3]`。而此时我们内部的 `onFulfilled` 中，`ret` 的值为 `{value: undefined, done: true}`，在 next 方法中直接 resolve 返回，结束运行。

### 登堂入室

有了上面的这些基础打底，终于可以来讲讲正题了。老规矩，我们先来看下 Koa 中如何添加中间件。

```js
var koa = require('koa');
var app = koa();

app.use(function* (next) {
  console.log('gen1 start');
  yield next;
  console.log('gen1 end');
});

app.use(function* (next) {
  console.log('gen2 start');
  yield next;
  console.log('gen2 end');
});

app.listen(3000);
```
在这里，我们添加了两个中间件，先不着急知道代码的输出结果，当你第一次看到这段代码的时候，会不会有疑问？

* next 是个什么鬼，不传会怎么样
* yield next 是做了什么操作
* 他们之间的执行顺序又是怎样的

那么我们就带着这些问题，来看下 Koa 内部是怎么实现的，为方便理解，删去了部分代码

```js
function Application() {
  // xxx
  this.middleware = [];
  // xxx
}

app.use = function(fn){
  // xxx
  this.middleware.push(fn);
  return this;
}

app.listen = function(){
  debug('listen');
  var server = http.createServer(this.callback());
  return server.listen.apply(server, arguments);
};

app.callback = function(){
  // xxx
  var fn = this.experimental
    ? compose_es7(this.middleware)
    : co.wrap(compose(this.middleware));

  if (!this.listeners('error').length) this.on('error', this.onerror);

  return function handleRequest(req, res){
    // xxx
    fn.call(ctx).then(function handleResponse() {
      respond.call(ctx);
    }).catch(ctx.onerror);
  }
};
```
由代码可知，我们通过 `use` 方法添加的中间件，都被塞到了一个事先定义好的 `middleware` 数组中。通过 `app.listen` 入口方法，创建了一个 http server 服务。在 `callback` 回调中，我们着重来看下这行代码

```js
co.wrap(compose(this.middleware))
```
这是嵌套了两层方法，第一层由中间件组件的 `middleware` 数组被传入了 `compose` 方法中；第二层是由第一层返回的结果传给了 `co.wrap` 方法。那我们先来看第一层的 `compose` 方法

```js
function compose(middleware){
  return function *(next){
    if (!next) next = noop();

    var i = middleware.length;

    while (i--) {
      next = middleware[i].call(this, next);
    }

    return yield *next;
  }
}

function *noop(){}
```
返回一个 generator 函数 ，在函数内部，`i` 是数组长度，`middleware[i]` 表示数组最末尾一个中间件，执行`middleware[i].call(this, next)` 生成一个 generator 实例并赋值给 `next`。大家注意下这里的参数 `next`，**第一次** `next` 为 `noop` 这个空的 generator function。做完一次 `i--` 后，`next` 成了上一个 generator 函数的实例对象，以此类推。换句话说，这个 `while` 循环从最后一个中间件开始处理，一直往前，把后一个 generator 函数的实例作为前一个 generator 函数的参数传入，那么执行完整个循环，我们的 `next` 的值就是数组第一个 generator 的实例。

看到这，我们是不是可以解答上面第一个疑问了呢，参数 `next` 其实就是下一个中间件的 generator 实例。

### 揭开面纱

compose 的存在，让整个中间件都串联了起来，但它并没有让中间件跑起来。要让整个过程跑起来，关键还是要看 co。我们继续来看下面的代码

```js
co.wrap = function (fn) {
  createPromise.__generatorFunction__ = fn;
  return createPromise;
  function createPromise() {
    return co.call(this, fn.apply(this, arguments));
  }
};
```
这里的 `fn` 就是我们上面 compose 返回的 generator 函数。在 `wrap` 内部调用 `co`，继续我们 co 内部的分析，在函数 `onFulfilled` 内部**第一次**调用 `gen.next` 方法，执行到 compose 内部的 `yield *next`，next 是我们的第一个中间件，根据 delegating yield 的知识，它会代理到第一个中间件的内部去，在我们中间件的 `yield next` 处暂停，此处 `next` 是下一个中间件的 generator 实例。`ret` 的值是 `{value: generator实例, done: false}`，再把 ret 传给内部的 next 方法。

```js
if (isGeneratorFunction(obj) || isGenerator(obj)) 
	return co.call(this, obj);
```
如果 value 值是 generator 函数或者是 generator 实例，则继续调用 co。在函数 `onFulfilled` 内部**第二次**调用 `gen.next` 方法到第二个中间的 `yield next` 处暂停。以此类推，直到最后遇到那个空的 generator 函数 `noop` 为止，执行 `if (ret.done) return resolve(ret.value)`，promise 状态置为 `fulfilled`。 

```js
var value = toPromise.call(ctx, ret.value);
if (value && isPromise(value)) 
	return value.then(onFulfilled, onRejected);
```
因为每次调用 co 都是返回一个 promise 实例，且 `ret.done` 为 true 的时候，状态被置为 `fulfilled`，所以执行回调中的 `onFulfilled` 函数。这样又从最后一个中间件往回执行，像个回形标一样，整个流程串了起来。

看完这部分，我们再来解答下剩下的两个疑问，`yield next` 用于执行下一个中间件的内容；中间件之间的执行顺序也是按照 `use` 的顺序来执行的。

是不是觉得整个过程很绕呢，但又不得不佩服作者设计的巧妙。我们可以从下图更直观的理解整个执行过程

![](https://camo.githubusercontent.com/9e70f81c6cccdd5b8ecb5d6199188c4fec6efcba/687474703a2f2f6f647373676e6e70662e626b742e636c6f7564646e2e636f6d2f2545342542382541442545392539372542342545342542422542362e706e67)
> 图片来自参考3

### 知识点扩展

到此为止，中间件的整个执行过程都已经讲解完了。不过大家在看 compose 源码的时候，有没有一个疑惑呢，为什么在 compose 内部是 `yield *next`，而在我们的中间件里都是 `yield next` 呢？按理说这里的 next 都是 generator 实例啊，有什么区别？

Koa 的维护者们也曾讨论过这个问题，具体看查看最后两个参考链接。性能上来说，`yield *next` 稍微优于 `yield next`，毕竟前者是原生的，后者是需要经过 co 包装处理过的，但写成后者也没什么大影响，Koa 作者 TJ 本人也是很反对两者之间切换来写的，推荐使用后者。再者，`yield *` 后面只能跟 generator 函数或者是可迭代的对象，而中间件中的 `yield` 我们可以跟 `function`，`promise`，`generator`，`array` 或者 `object`，因为 co 最终都会帮我们处理成 promise，所以建议大家在用 Koa 做开发的时候，都能写成 `yield next`。

### 参考

* [http://es6.ruanyifeng.com/#docs/generator](http://es6.ruanyifeng.com/#docs/generator)
* [http://es6.ruanyifeng.com/#docs/promise](http://es6.ruanyifeng.com/#docs/promise)
* [https://github.com/qianlongo/resume-native/issues/1](https://github.com/qianlongo/resume-native/issues/1)
* [http://taobaofed.org/blog/2015/11/19/yield-and-delegating-yield/](http://taobaofed.org/blog/2015/11/19/yield-and-delegating-yield/)
* [http://www.jongleberry.com/delegating-yield.html](http://www.jongleberry.com/delegating-yield.html)
* [https://github.com/koajs/compose/issues/2](https://github.com/koajs/compose/issues/2)

---
by 贾克斯