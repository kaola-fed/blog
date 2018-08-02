Promise已经像血液一样融入到我们的日常工作中，thenable无时无刻不在发挥着它的作用。网络上关于Promise的文章也是汗牛充栋，人们一遍又一遍的咀嚼着Promise.prototype.then, Promise.prototype.catch的作用和功效。

很多时候，我们的执行函数会是p.then(onFulfilled).catch(onRejected)这种形式，并不会链接太多操作，这有点像try语句。

有时候，我们希望一个操作，无论是否抛出异常都会执行，即不管它进入了resolve，还是reject，下一步操作都会执行。比如我们发送请求之前会出现一个loading图，当我们请求发送完成之后，不管请求有没有出错，我们都希望关掉这个loading图以提升用户体验。过去我们可以这么做：

```JS
  this.loading = true
  
  request()
    .then((res) => {
      // do something
      this.loading = false
    })
    .catch(() => {
      // log err
      this.loading = false
    })
```

这种写法一点也不DRY，显得丑陋。为了让代码稍微好看一点，我们也可以这么写：

```JS
  this.loading = true
  
  request()
    .then((res) => {
      // do something
    })
    .catch(() => {
      // log err
    })
    .then(() => {
      this.loading = false
    })
```

这么写虽然让我们舒服了点，但是，总感觉哪里怪怪的，总觉得.then后面还有.catch。作为类比，我们可以看一下try语句的三种声明形式：

1. Try…catch
2. Try…finally
3. Try…catch…finally

那么问题来了，为什么Promise没有实现finally的写法，用于在任何情况下（不论成功或者失败）执行特定后续操作。这样在语义上，代码显得更加直观与合理。

好消息是，Promise.prototype.finally已于2018/01/24进入ES8的stage4阶段。半数以上的浏览器也做出了兼容。我们现在就可以尝鲜使用了。

于是上面的例子就变成了这样：

```JS
  this.loading = true
  
  request()
    .then((res) => {
      // do something
    })
    .catch(() => {
      // log err
    })
    .finally(() => {
      this.loading = false
    })
```

看起来好像也没有什么区别，但其实使用Promise.finally(onFinally) 和 Promise.then(onFulfilled, onRejected)还是有以下几点不同：
1. 调用内联函数时，不需要多次声明该函数或为该函数创建一个变量保存它。
2. 由于无法知道promise的最终状态，所以finally的回调函数中不接收任何参数，它仅用于无论最终结果如何都要执行的情况。
3. 与Promise.resolve(2).then(() => {}, () => {}) （resolved的结果为undefined）不同，Promise.resolve(2).finally(() => {}) resolved的结果为 2。
4. 同样，Promise.reject(3).then(() => {}, () => {}) (resolved 的结果为undefined), Promise.reject(3).finally(() => {}) rejected 的结果为 3。

1、2两点容易理解，至于3、4，可以通过下面的两个例子进行说明：

```JS
Promise.resolve('foo').
  finally(() => 'bar').
  then(res => console.log(res));
  
Promise.reject(new Error('foo')).
  finally(() => 'bar').
  catch(err => console.log(err.message));
  
// 最终打印的是‘foo’而不是‘bar'，因为finally()会透传fullfilled的值和rejected错误
```

如果浏览器没有实现finally，那么可以实现polyfill：

```JS
if (typeod Promise.prototype.done !== 'function') {
  Promise.prototype.finally = function(onFinally) {
  
    return this.then(
      /* onFulfilled */
      res => Promise.resolve(onFinally()).then(() => res),
      /* onRejected */
      err => Promise.resolve(onFinally()).then(() => { throw err; })
    );
  };
}
```

通过该polyfill，可以更加理解为什么finally()会透传fullfilled的值和rejected错误。

Promise.prototype.finally()会返回一个promise对象，Promise chain会延续下去。但是如果我们不想这条Promise chain继续执行下去，而想在执行一个操作后关闭Promise chain。这种时候就需要用到Promise.prototype.done().

Promise.prototype.done的使用方法和then一样，但是该方法不会返回Promise对象。

实现Promise.prototype.done的polyfill如下：

```JS
if (typeof Promise.prototype.done === 'undefined') {
    Promise.prototype.done = function (onFulfilled, onRejected) {
        this.then(onFulfilled, onRejected).catch(function (error) {
            setTimeout(function () {
                throw error;
            }, 0);
        });
    };
}
```
可以看到，Promise.prototype.done和Promise.prototype.finally存在两点不同：
1. done并不返回promise对象，也就是说，在done之后不能使用then，catch等方法组成方法链。
2. done中发送的异常会被直接抛给外部，也就是说，其不会进行Promise的错误处理（Error Handling）

> 由于Promise的try-catch机制，出错的问题可能会被内部消化掉。 如果在调用的时候每次都无遗漏的进行 catch 处理的话当然最好了，但是如果在实现的过程中出现了这个例子中的错误的话，那么进行错误排除的工作也会变得困难。这种错误被内部消化的问题也被称为 unhandled rejection ，从字面上看就是在Rejected时没有找到相应处理的意思。  - promise迷你书

以上便是Promise.prototype.finally和Promise.prototype.done的介绍和说明，也许实际使用的频率不会很高，但是为了高可读的代码，大家不妨一试。 (ゝ∀･)

### 参考

- [promise迷你书](http://liubin.org/promises-book/#promise-done)
- [proposal-promise-finally](https://github.com/tc39/proposal-promise-finally)
- [using-promise-finally-in-node-js.html](http://thecodebarbarian.com/using-promise-finally-in-node-js.html)
- [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/finally)

