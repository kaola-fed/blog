>react-redux和vuex是怎么解决深层嵌套组件获取到store的呢，本文将进行分析和比较。

#### react-redux的方式
首先看下react-redux的使用方式，利用react-redux提供的Provider组件，包裹在我们的根组件外面，并将store作为props传入Provider.
```javascript
import React from 'react';
import ReactDom from 'react-dom';
import { createStore, applyMiddleware } from 'redux';
import { Provider } from 'react-redux';
import App from 'jsModules/app/app';
import reducer from './reducers/index';

const store = createStore(reducer);

render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
)
```

那么这个store是怎么让所有的后代组件都能获取到的呢。关键就在于react的[context api](https://reactjs.org/docs/context.html).

看下react-redux中Provider组件的关键代码：
```javascript
export function createProvider(storeKey = 'store', subKey) {
    const subscriptionKey = subKey || `${storeKey}Subscription`

    class Provider extends Component {
        getChildContext() {
            return {
                [storeKey]: this[storeKey], [subscriptionKey]: null }
        }

        constructor(props, context) {
            super(props, context)
            this[storeKey] = props.store;
        }

        render() {
            return Children.only(this.props.children)
        }
    }

    ...
    Provider.childContextTypes = {
        [storeKey]: storeShape.isRequired,
        [subscriptionKey]: subscriptionShape,
    }

    return Provider
}

export default createProvider()

```
其中的storeKey默认是'store'，可以看到这里就是一个context api的简单使用。通过定义childContextTypes和getChildContext就能将store放在context对象上，子组件通过定义contextTypes来获取。

然后就是后代组件怎么获取store呢，后代组件是这么用的：
```javascript
export default connect(mapStateToProps, mapDispatchToProps)(SideBar);

```
这就涉及到react-redux的connect部分，看关键代码：

```javascript
    const contextTypes = {
      [storeKey]: storeShape,
      [subscriptionKey]: subscriptionShape,
    }

    class Connect extends Component {
      constructor(props, context) {
        super(props, context)
        ..
        this.store = props[storeKey] || context[storeKey]
        ...
      }
      render() {
        const selector = this.selector
        selector.shouldComponentUpdate = false

        // selector.props
        // 就是categoryList currentApi getCategoryList 这些属性

        if (selector.error) {
          throw selector.error
        } else {
          return createElement(WrappedComponent, this.addExtraProps(selector.props))
        }
      }
    }

    Connect.contextTypes = contextTypes
    return hoistStatics(Connect, WrappedComponent)
```
子组件定义了contextTypes，然后在构造函数的时候拿到context.store赋给this.store.

connect的render方法是依据传入的组件，加上props（就是mapStateToProps, mapDispatchToProps转换之后的对象）返回一个新的组件。这就是react中[HOC](https://react.bootcss.com/docs/higher-order-components.html)的概念。

#### vuex中的方式
vuex中的方式可以参考我的另一篇文章[vuex源码分析](https://github.com/kaola-fed/blog/issues/47)

这里简单的回顾下。

vuex的installl方法里会调用appMixin(Vue)给vue添加全局的mixin

```javascript
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    const usesInit = Vue.config._lifecycleHooks.indexOf('init') > -1
    Vue.mixin(usesInit ? { init: vuexInit } : { beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}

```
给init钩子加上了方法vuexInit。当传入了store的时候，就把这个store挂载到实例的$store上，没有的话，并且实例有parent的，就把parent的$store挂载到当前实例上。这样，我们在Vue的组件中就可以通过this.$store.xxx访问Vuex的各种数据和状态了。

相对于react中的context api这种全局的获取方法，vuex是逐层往上找的方式。

完

by kaolafed/fangwentian

参考文档：
- [Context](https://reactjs.org/docs/context.html)
- [React-Redux 的用法](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_three_react-redux.html)
- [高阶组件](https://react.bootcss.com/docs/higher-order-components.html)