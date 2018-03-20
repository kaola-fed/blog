redux 主要包含 5 个方法，分别是：

* createStore
* combineReducers
* bindActionCreators
* applyMiddleware
* compose

今天主要讲解下 `applyMiddleware` 和 `compose` 这两个方法。在 redux 中引入了中间件的概念，没错如果你使用过 Express 或者 Koa 的话，一定不会对中间件陌生。我们知道，在 Koa 中，串联各个中间件的正是 `compose` 方法，所以在 redux 中也同样使用了这个命名，作用也是串联所有中间件。

### reduce 用法

在正式讲解前，我们先来看下 `reduce` 的用法。根据 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce) 上的解释，

> reduce() 方法是对累加器和数组中的每个元素（从左到右）应用一个函数，将其减少为单个值。

```js
arr.reduce(callback[, initialValue])
```
##### 参数

* `callback`

	执行数组中每个值的函数，包含四个参数：
	
	* accumulator：累加器累加回调的返回值; 它是上一次调用回调时返回的累积值，或 initialValue
	* currentValue：数组中正在处理的元素。
	* currentIndex：数组中正在处理的当前元素的索引。 如果提供了initialValue，则索引号为0，否则为索引为1。
	* array：调用 reduce 的数组
* `initialValue `
	
	用作第一个调用 callback的第一个参数的值。 如果没有提供初始值，则将使用数组中的第一个元素。 在没有初始值的空数组上调用 reduce 将报错。

##### 返回

函数累计处理的结果

### compose 分析

有了上面 reduce 的基础，我们再来看下 compose 的代码。compose 的代码很简单，10行代码左右，但你看到 `reduce` 部分的时候，估计会一脸懵逼，短短的一行代码看上去却很绕。

```js
/**
 * Composes single-argument functions from right to left. The rightmost
 * function can take multiple arguments as it provides the signature for
 * the resulting composite function.
 *
 * @param {...Function} funcs The functions to compose.
 * @returns {Function} A function obtained by composing the argument functions
 * from right to left. For example, compose(f, g, h) is identical to doing
 * (...args) => f(g(h(...args))).
 */
 
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

看注释，它的作用应该是 

```
执行 compose(f, g, h)
得到 (...args) => f(g(h(...args)))
```
我们来推导下，它是怎么得出这个结果的。假设 `funcs` 等于 `[f1, f2, f3]`，其中 `f1`，`f2`，`f3` 是三个中间件，`(a, b) => (..args) => a(b(...args))` 等于 `f`，那么 `funcs.reduce((a, b) => (...args) => a(b(...args)))` 可以简化为 `[f1, f2, f3].reduce(f)`。

第 1 次执行 `f`： 

```
a = f1
b = f2 
返回 (...args) => f1(f2(..args))
```

第 2 次执行 `f`：

```
a = (...args) => f1(f2(...args))
b = f3
返回 (...args) => a(f3(...args)) = f1(f2(f3(...args)))
```
通过上面的推导，证实了先前得出的结论

```
compise(f, g, h) = (...args) => f(g(h(...args)))
```

### applyMiddleware 分析

通过上面的分析，我们知道 `compose` 是对中间件的串联，那么 `applyMiddleware` 就是对中间件的应用了。最终返回 `createStore` 中的方法以及经过中间件包装处理过的 `dispatch` 方法。

```js
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }
    let chain = []

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```
我们通过一个具体的中间件 `redux-thunk`，来查看它内部到底是怎么来执行加载的中间件的。

```js
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```
中间件中包含了三个箭头函数，在 `applyMiddleware` 中的 `map` 操作后，返回了第二层箭头函数，所以 `chain` 中存储的是各个中间件的第二层函数。

根据 `compose` 的分析，

```
dispatch = compose(...chain)(store.dispatch)
等于
dispatch = f1(f2(f3(store.dispatch)))
```
我们先执行第三个中间件，并把返回结果作为第二个中间件的入参继续执行，以此类推，下一个中间件的入参是上一个中间件的返回。如果说这里第三个中间件是上面的 `redux-thunk`，那么函数中的 `next` 就是 `store.dispatch`，返回第三个箭头函数 `action`。这里返回的第三个箭头函数，就是第二个中间件的 `next` 形参。以此类推，第二个返回的 `action` 就是第一个中间件的 `next` 形参。但是这里都还没真正开始执行中间件。

当我们外部调用 `store.dispatch(action)` 方法的时候，才要真正开始执行各个中间件。首先执行中间件 `f1`，当执行到 `next` 的时候，开始执行第二个中间件 `f2`，以此类推直到最后一个中间件，调用原生 `store.dispatch` 方法。

之所以要写这么绕，也是为了符合 redux 单一数据源的原则，applyMiddleware 的写法保证了 action 的流向，而且每一步的数据变化都是可以追踪的。

### 其他

对比了 `4.0.0-beta.1` 之前版本的 `applyMiddleware` 的[区别](https://github.com/reactjs/redux/commit/dd165dfc6878bc9aa6045bc1fc1943640a23e5e8#diff-00cf56e35f4f14a3079fa426caa8ef42)，发现内部 `dispatch` 从之前的 `store.dispatch` 改成了现在的直接抛出一个错误。根据这个 [issues](https://github.com/reactjs/redux/issues/1240) 的讨论，在中间件顶层调用了 `store.dispatch`，结果导致无法执行后面的中间件。这个调用应该是在处理 `map` 操作的时候执行的，此时的 `applyMiddleware` 还没执行完，`store.dispatch` 调用的还是原生 `createStroe` 中的方法才导致的这个问题。
  
另外如果在中间件中即 `action` 层使用 `dispatch` 会怎样呢？我们知道我们可以通过 `next` 进入到下个中间件，那如果调用 `store.dispatch` 的话又会从外层重新来一遍，假如这个中间件内部只是粗暴的调用 `store.dispatch(action)` 的话，就会形成死循环。如下图所示

![](https://haitao.nos.netease.com/5166f563-2424-49fe-9ce9-a71fb31d3ecb.png)
	
### 参考

> [redux middleware 详解](https://segmentfault.com/a/1190000004485808)
> 
> [Dispatching in a middleware before applyMiddleware completes](https://github.com/reactjs/redux/issues/1240)

by 贾克斯