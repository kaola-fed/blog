---
title: Promise对象剖析
date: 2017-04-01
---

> 作为已经成为标准规范的Promise对象，如何使用想必大家都很了解能够正确的应用，本文从分析Promise对象和then方法源码入手来探索Promise到底是如何实现，文章篇幅有限，如有不对的地方欢迎指正。

<!-- more -->

### Promise对象和then方法
#### 根据ES2016标准实现的简单promise对象
```
var PENDING = 0;
var FULFILLED = 1;
var REJECTED = 2;
function Promise(fn) {
    var self = this;
    self.state = PENDING;
    self.value = null;
    self.handlers = [];

    function fulfill(result) {
       if(self.state === PENDING) {
            self.state = FULFILLED;
            self.value = result;
            for(var i = 0; i<self.handlers.length; i++) {
                self.handlers[i](result);
            }
        }
    }
    function reject(err) {
        if(self.state === PENDING) {
            self.state = REJECTED;
            self.value = err;
        }
    }
    fn && fn(fulfill,reject);
}

Promise.prototype.then = function(onResolved, onRejected) {
    var self = this;
    return new Promise(function(resolve, reject) {
        var onResolvedFade = function(val) {
            var ret = onResolved ? onResolved(val) : val;
                resolve(ret);
        };
        var onRejectedFade = function(val) {
            var ret = onRejected ? onRejected(val) : val;
                reject(ret);
        };
        self.handlers.push(onResolvedFade);
        if(self.state === FULFILLED) {
            onResolvedFade(self.value);
        }
        if(self.state === REJECTED) {
            onRejectedFade(self.value);
        }
    })
}
```
### Promise对象
- 三个私有属性
    - state：用来保存当前执行的状态
    - value：保存异步操作返回的值
    - handles：用来存异步回调的容器，在状态改变为fulfill或reject的时候遍历handles中的值（函数）来执行相应的方法
- 两个内置方法fulfill和reject
    - 变化状态，并执行对应的逻辑
- 一个参数
    - 接受一个回调函数，用内置的方法fulfill和reject作为该回调函数的参数

### then方法
then方法接受两个参数，返回一个新的Promise对象，主要实现点在传入到Promise对象中的回调函数。
- 该回调函数内部封装了两个函数，即成功和失败时要执行的函数。然后将成功的回调存到then方法上下文作用域的Promise对象中的handlers容器中（该实现方法只用成功回调作例子）
- onResolvedFade和onRejectedFade
    - onResolvedFade封装了传入到then方法的第一个参数，运行该函数，调用resolve，并将返回值作为then返回新的Promise对象内置函数fulfill的参数，作为下一次回调的初始值。
    - onRejectedFade和onResolvedFade同理

### 一个例子
```
function async(value){
    var pms = new Promise(function(resolve, reject){
        setTimeout(function(){
            resolve(value);
        }, 1000);
    });
    return pms;
}

async(1).then(function(result){
    console.log('the result is ',result);//the result is 1
    return result;
}).then(function(result){
    console.log(++result);//2
});
```
async函数返回一个Promise对象，传入该Promise的参数模拟异步操作，1s后执行resole方法。定义了两个then方法，分别传入了成功时的回调函数。根据刚才的promise函数和then方法分析，第一个then方法的第一个参数会被存到当前then上下文作用域的Promise对象的handlers中，第二个then方法的第一个参数会被存到当前then上下文作用域的promise对象(也就是第一个then返回的新的Promise对象)，等待1s后，执行async返回的promise对象中的fulfill方法，然后依次执行then定义的回调函数。

### 所以Promise的原理到底是什么
- 通过定义then方法中的成功和异步回调，存到当前then方法上下文作用域的promise对象的handlers容器中，当异步返回结果时，调用该promise的fulfill方法，改变状态，链式调用定义好的回调函数，达到将异步当同步写的目的。


### 总结

- 要搞懂Promise，最主要的是搞懂Promise构造器的写法，和then函数的封装。

- 要用好Promise，最主要的是封装好要传入Promise中的匿名函数或者函数表达式。

- 有几个调用就有几个Promise对象，每个then方法传入的**函数**都是前一个Promise成功或者失败的异步回调。保存到每一个promise中的handlers中的函数都是**闭包**

- 第一个调用返回的值是后一个调用then方法传入函数的参数

- 定义promise与then的时候就像按次序放置的多米诺骨牌，当我们推动第一张骨牌的时候，也就是异步结果返回成功时，定义在then方法中的函数会开始链式调用。






