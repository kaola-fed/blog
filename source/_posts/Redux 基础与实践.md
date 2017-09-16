title: Redux 基础与实践
date: 2017-07-27 22:27:31
tags:
---
之前写过一篇 [Regular 组件开发的一些建议](https://github.com/kaola-fed/blog/issues/103) 的文章提到了日常开发Regular组件的一些槽点，并提出了在简单需求，不使用状态管理框架时的一些替代方案。本文的目的便是填前文的一个坑，即较复杂需求下 Redux 引入方案。

# [关于 Redux](http://cn.redux.js.org/index.html)
> Redux 是 JavaScript 状态容器，提供可预测化的状态管理。

<!-- more -->

# 看一个简单的 DEMO
```js
const store = redux.createStore(function(prevState, action) {
    if (!prevState) {
        prevState = {
            count: 0
        };
    }

    switch(action.type) {
        case 'REQUEST':
            return {
                count: prevState.count + 1
            }
    }

    return prevState;
});

store.dispatch({
    type: 'REQUEST'
});

store.subscribe(function () {
    console.log(store.getState());
});
```

理解他，我们可以结合后端 MVC 的 web 模型
![web架构](https://haitao.nos.netease.com/b9190c99-f5a3-4637-890e-9a597f2099f4.jpeg)
![redux架构](https://haitao.nos.netease.com/d8d909af-b660-48bb-a197-542421332ecc.jpeg)

# 演员表
## Store 饰演 应用服务器
```js
const store = redux.createStore(f);
```

createStore 这个 API 会创建一台应用服务器，包含数据库存储，以及一个 web Server

## State 饰演 DataBase
```js
const state = store.getState()
```
getState 操作会返回当前的服务器数据库数据，这台数据库 bind 了 0.0.0.0（闭包），所以外界无法操作关于数据库的信息，只能通过内部的服务对数据库做修改

## Action 饰演请求
```js
store.dispatch({
    type: 'type',
    payload: {}
})
```

dispatch(action) 的操作就像是往服务器发送请求。action.type 就像是请求的 uri，action.payload 就像是请求的 body/query。

## Reducer 饰演 Controller + Service + DAO
```js
const reducer = function (prevState, action) {
    if (!prevState) {
        return {}
    }
    switch(action.type) {
        // 分发处理
    }
    return prevState;
}
redux.createStore(reducer);
```

相应请求会进入对应的控制器（就像 reducer 的 switch 判断），而控制器内也会对请求携带的信息(payload)做分析，来实现对数据库的增删改查。

# 设计原则
## 单一数据源
> 整个应用的 state 被储存在一棵 object tree 中，并且这个 object tree 只存在于唯一一个 store 中。

一般情况下 createStore 创建的 store ，将作用于整个应用

## State 是只读的
> 唯一改变 state 的方法就是触发 action，action 是一个用于描述已发生事件的普通对象。

避免人为的操作 state 变量，造成状态变化不可预测的情况，人为修改 state 的方式被切断了。

至于如何实现，[点我看源码](https://github.com/ImHype/tiny-redux/blob/master/src/createStore.js#L5)

`createStore` 的操作，会创建一个内部变量 state, 由于 return 出来的 `dispatch` 和 `subscribe` 方法保持了对 state 变量的引用，所以 state 会以闭包的形式存活下来。

## 使用纯函数来执行修改
> 为了描述 action 如何改变 state tree ，你需要编写 reducers。

使用纯函数，可测试性和可维护性得到了保障。

# 完成与视图层的绑定
Redux 职责是状态管理，并非只限定于前端开发，要使用到 web 开发当中，还缺少一个部分，完成 MVC 中剩下的一些操作（渲染页面）。

![web架构](https://haitao.nos.netease.com/b9190c99-f5a3-4637-890e-9a597f2099f4.jpeg)
![v-redux架构](https://haitao.nos.netease.com/6f685b72-19bd-4dfd-a191-0de7d0dfa10e.jpg)

## 两个重要的 API
* store.subscribe(f) - 发布订阅模型， 方法 f 会在 dispatch 触发后执行
* store.getState() - return 出当前完整的 state 树

## 简单粗暴的方式
```js
const store = redux.createStore(reducer);
const ComponentA = Regular.extend({
    config() {
        const ctx = this;
        store.subscribe(function () {
            const state = store.getState();
            const mapData = {
                clicked: state.clicked,
            }
            Object.assign(ctx.data, mapData);
            ctx.$update();
        });
    }
});
```

## 两个问题
1. Store 获取
2. mapData 与 $update 操作不可复用

## 更优雅的绑定方式

```html
<StoreProvier store={store}>
    <Component></Component>
</StoreProvier>
```

```js
const Component = connect({
    mapState(state) {
        return {
            clicked: state.clicked,
        }
    }
})(Regular.extend({
    config() {
    }
}))
```

### 1. [StoreProvier](https://github.com/ImHype/regular-redux-connector/blob/master/src/StoreProvider.js)

```js
Regular.extend({
    name: 'StoreProvider',
    template: '{#include this.$body}',
    config({store} = this.data) {
       if (!store) {
           throw new Error('Provider expected data.store to be store instance created by redux.createStore()')
       }
       
       store.subscribe(() => {
           this.$update();
       });
    }
})
```

### 2. [connect](https://github.com/ImHype/regular-redux-connector/blob/master/src/connect.js)
> 统一从最外层的 StoreProvider 组件获取 store，保证单一数据源

```js
function getStore(ctx) {
    let parent = ctx.$parent;
    while(true) {
        if (!parent) {
            throw new Error('Expected root Component be Provider!')
        }

        if (parent.data.store) {
            return parent.data.store;
        }

        parent = parent.$parent;
    }
}

function connect({
    mapState = () => ({}),
    dispatch
} = {}) {
    return (Component) => Component.implement({
        events: {
            $config(data = this.data) {
                const store = getStore(this);
                const mapStateFn = () => {
                    const state = store.getState();
                    const mappedData = mapState.call(this, state);
                    mappedData && Object.assign(this.data, mappedData);
                }
                mapStateFn();

                const unSubscribe = store.subscribe(mapStateFn);
                
                if (dispatch) {
                    this.$dispatch = store.dispatch;
                }
                
                this.$on('destroy', unSubscribe);
            } 
        }
    });
}
```

至此回过头看 regualr-redux 的架构图，发现正是 `StoreProvier` 和 `connect` 操作帮助 `redux` 完成了与 MVC 相差的更新视图操作。


# 总结
借助 redux 与特定框架的连接器，我们会发现，对特定 MVVM 框架的要求会变得很低 -- mapState 操作可以完成 类似 Vue 中 computed/filter 的操作。

所以，如今还在半残废中的微信小程序也很适合基于这套思路来结合 redux (逃)。

带来的好处是，你可以安心维护你的模型层，不用担心 `data` 过大导致的脏值检查缓慢，也不需要考虑一些逻辑相关的数据不放入 `data` 那该放入何处。

相信阅读此文，你会对 `Redux` 解决问题的方式有了一定的认识，而继续深入的一些方向有:
* [middleway](http://cn.redux.js.org/docs/advanced/Middleware.html) 实现原理，redux-logger 和 redux-thunk 等实现方案；
* [State 范式化](http://cn.redux.js.org/docs/recipes/reducers/NormalizingStateShape.html)；
* [Presentational and Container Components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)
* 单页的 redux 架构设计

全文完 ；）

by [君羽](http://github.com/imhype/)