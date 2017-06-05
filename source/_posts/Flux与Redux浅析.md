---
title: Flux与Redux浅析
date: 2017-05-31
---

## 什么是Flux?
—— 简单地说，Flux是用来构建用户端的Web应用程序的体系架构，用来管理和控制应用中数据的流向。

![Flux数据流](https://haitao.nos.netease.com/1dd8055f-2ba0-47c6-b15d-30ae205cc2f3.png)

<!-- more -->

从上图可知，Flux最大的一个优点就是数据的单向流动。
> 1.首先要有 action，通过定义一些 action creator 方法根据需要创建 Action 提供给 dispatcher
  2.View 层通过用户交互（比如 onClick）会触发 Action
  3.Dispatcher 会分发触发的 Action 给所有注册的 Store 的回调函数
  4.Store 回调函数根据接收的 Action 更新自身数据之后会触发一个 change 事件通知 View 数据更改了
  5.View 会监听这个 change 事件，拿到对应的新数据并调用 setState 更新组件 UI

因此，我们可以将上图总结一下：
**Action**可以看成是修改Store的行为抽象；
**Dispatcher**管理着应用的数据流，可以看为Action到Store的分发器；
**Store**管理着整个应用的状态和逻辑，类似MVC中的Model。

相对于MVC中View具备多个修改Modal的能力，导致VC<->M在复杂页面中会比较混乱。
![MVC](https://haitao.nos.netease.com/f9abd847-0212-4584-b580-c268bb621630.png)

Flux实现了所有的状态都通过Store来维护，Store统一控制View，View没有直接修改Store的能力，而是发起Action通过Dispatcher去修改Store(通过 Action 传递数据)。避免了数据流的混乱。

[Flux Demo](https://github.com/facebook/flux/tree/master/examples)

---

## 什么是Redux?

—— Redux是基于Flux架构的一次改进，官方的定义：Redux is a predictable state container for JavaScript apps（可预测的状态容器）。

![Redux数据流](https://haitao.nos.netease.com/116462f9-e8fb-4daa-89e2-73a129d29dce.png)

从图中可以看出，Redux和Flux最大的不同在于Redux用Reducer代替了Flux的Dispatcher。Redux设想你永远不会变动你的数据，而应该在Reducer中返回新的对象来作为应用的新状态。因此，State 应该是只读的，唯一改变 State 的方法就是触发Action。而为了描述 Action 如何改变 State Tree ，需要编写 Reducers来实现。

**Reducer函数**：是纯函数，不应该有副作用，不应有API调用，Date.now()或者随机获取等不稳定的操作，应当保证相同的输入Reducer计算的结果应该是一致的输出，它只会进行单纯的计算。

```JavaScript
function Reducer(state, action) {
  switch (action.type) {
    case ACTION_TYPE:
      // calc...
      return newState;
    default: return state;
  }
  return newState;
}
```

State是不可修改的，所以返回的新State应该是基于输入State副本的修改，而不是直接修改State后的返回。

### Redux三大原则
> **1.单一数据源(Store)**
    整个应用的State被存放在一棵Object tree中，并且这个Object tree只存在唯一一个Store中；
  **2.State是只读的 唯一改变 State 的方法就是触发 Action，Action 是一个用于描述已发生事件的普通对象。**
  确保了View和网络请求无法直接修改State，所有的修改都能被集中化处理。
  **3.通过纯函数Reducer来修改Store,**
  Reducer 只是一些纯函数，它接收先前的 State 和 Action，并返回新的 State。
  即reducer(state, action) => new state
  
  
### Redux与Flux的对比

前面我们称Redux是基于Flux架构的一次改进，那么Redux和Flux具体有哪些区别呢？从代码层面来看：
**Action:**
- *Flux:*
```JavaScript
/*Flux直接在Action中调用Dispatch*/
export function addTodo(text) {
  AppDispatcher.dispatch({
    type: ActionTypes.ADD_TODO,
    text: text
  });
}
```
- *Redux:*
```JavaScript
/*Redux将Action和Dispatch解耦*/
export function addTodo(text) {
  return {
    type: ActionTypes.ADD_TODO,
    text: text
  };
}
```

**Store:**
- *Flux:*
```JavaScript
let _todos = [];
const TodoStore = Object.assign(new EventEmitter(), {
  getTodos() {
    return _todos;
  }
});
AppDispatcher.register(function (action) {
  switch (action.type) {
  case ActionTypes.ADD_TODO:
    _todos = _todos.concat([action.text]);
    TodoStore.emitChange();
    break;
  }
});
export default TodoStore;
```
- *Redux:*
```JavaScript
/*Redux将Dispatch从Store中剥离*/
const initialState = { todos: [] };
export default function TodoStore(state = initialState, action) {
  switch (action.type) {
  case ActionTypes.ADD_TODO:
    return { todos: state.todos.concat([action.text]) };
  default:
    return state;
}
```

### Redux简单demo实现
```JavaScript
// actions
const INCREMENT = 'INCREMENT';
const DECREMENT = 'DECREMENT';

function incrementCreator(number) {
  return {
    type: INCREMENT,
    number,
  };
}

function decrementCreator(number) {
  return {
    type: DECREMENT,
    number,
  };
}

// 初始化state
const initialState = {
  counter: 0,
};

// reducers函数，注意最后一定要return state防止不能匹配到action的时候state丢失
function countReducer(state = initialState, action) {
  switch (action.type) {
    case INCREMENT:
      return Object.assign({}, {
        counter: state.counter + action.number,
      });
    case DECREMENT:
      return Object.assign({}, {
        counter: state.counter - action.number,
      });
    default: return state;
  }
}

// 创建store
const store = Redux.createStore(countReducer);

// 订阅store的修改
store.subscribe(function log() {
  console.log(store.getState());
});

// 通过dispatch action来改变state
store.dispatch(incrementCreator(5)); //Object {counter: 5}
store.dispatch(decrementCreator(4)); //Object {counter: 1}

```
  