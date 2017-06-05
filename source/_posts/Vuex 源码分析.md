---
title: Vuex 源码分析
date: 2017-05-23
---

> 本文解读的Vuex版本为2.3.1

#### Vuex代码结构
![](https://haitao.nos.netease.com/2393d861bc7249169c83b59e2a441779.png)

Vuex的代码并不多，但麻雀虽小，五脏俱全，下面来看一下其中的实现细节。

<!-- more -->

#### 源码分析

##### 入口文件
入口文件src/index.js:
``` javascript
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions } from './helpers'

export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions
}
```
这是Vuex对外暴露的API，其中核心部分是Store，然后是install，它是一个vue插件所必须的方法。Store
和install都在store.js文件中。mapState、mapMutations、mapGetters、mapActions为四个辅助函数，用来将store中的相关属性映射到组件中。

##### install方法
Vuejs的插件都应该有一个install方法。先看下我们通常使用Vuex的姿势：
``` javascript
import Vue from 'vue'
import Vuex from 'vuex'
...
Vue.use(Vuex)

```
install方法的源码:
```javascript
export function install (_Vue) {
  if (Vue) {
    console.error(
      '[vuex] already installed. Vue.use(Vuex) should be called only once.'
    )
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}

// auto install in dist mode
if (typeof window !== 'undefined' && window.Vue) {
  install(window.Vue)
}

```
方法的入参_Vue就是use的时候传入的Vue构造器。  
install方法很简单，先判断下如果Vue已经有值，就抛出错误。这里的Vue是在代码最前面声明的一个内部变量。
```javascript
let Vue // bind on install
```
这是为了保证install方法只执行一次。  
install方法的最后调用了applyMixin方法。这个方法定义在src/mixin.js中：
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
方法判断了一下当前vue的版本，当vue版本>=2的时候，就在Vue上添加了一个全局mixin，要么在init阶段，要么在beforeCreate阶段。Vue上添加的全局mixin会影响到每一个组件。mixin的各种混入方式不同，同名钩子函数将混合为一个数组，因此都将被调用。并且，混合对象的钩子将在组件自身钩子之前。

来看下这个mixin方法vueInit做了些什么:  
this.$options用来获取实例的初始化选项，当传入了store的时候，就把这个store挂载到实例的$store上，没有的话，并且实例有parent的，就把parent的$store挂载到当前实例上。这样，我们在Vue的组件中就可以通过this.$store.xxx访问Vuex的各种数据和状态了。

##### Store构造函数
Vuex中代码最多的就是store.js, 它的构造函数就是Vuex的主体流程。
```javascript
  constructor (options = {}) {
    assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
    assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)

    const {
      plugins = [],
      strict = false
    } = options

    let {
      state = {}
    } = options
    if (typeof state === 'function') {
      state = state()
    }

    // store internal state
    this._committing = false
    this._actions = Object.create(null)
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    this._modulesNamespaceMap = Object.create(null)
    this._subscribers = []
    this._watcherVM = new Vue()

    // bind commit and dispatch to self
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }

    // strict mode
    this.strict = strict

    // init root module.
    // this also recursively registers all sub-modules
    // and collects all module getters inside this._wrappedGetters
    installModule(this, state, [], this._modules.root)

    // initialize the store vm, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    resetStoreVM(this, state)

    // apply plugins
    plugins.concat(devtoolPlugin).forEach(plugin => plugin(this))
  }
```
依然，先来看看使用Store的通常姿势，便于我们知道方法的入参：
```javascript
export default new Vuex.Store({
  state,
  mutations
  actions,
  getters,
  modules: {
    ...
  },
  plugins,
  strict: false
})
```
store构造函数的最开始，进行了2个判断。
```javascript
assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
```
这里的assert是util.js里的一个方法。
```javascript
export function assert (condition, msg) {
  if (!condition) throw new Error(`[vuex] ${msg}`)
}
```
先判断一下Vue是否存在，是为了保证在这之前store已经install过了。另外，Vuex依赖Promise，这里也进行了判断。  
assert这个函数虽然简单，但这种编程方式值得我们学习。  
接着往下看：  
```javascript
const {
  plugins = [],
  strict = false
} = options

let {
  state = {}
} = options
if (typeof state === 'function') {
  state = state()
}
```
这里使用解构并设置默认值的方式来获取传入的值，分别得到了plugins, strict 和state。传入的state也可以是一个方法，方法的返回值作为state。  

然后是定义了一些内部变量：  
```javascript
// store internal state
this._committing = false
this._actions = Object.create(null)
this._mutations = Object.create(null)
this._wrappedGetters = Object.create(null)
this._modules = new ModuleCollection(options)
this._modulesNamespaceMap = Object.create(null)
this._subscribers = []
this._watcherVM = new Vue()
```
**this._committing** 表示提交状态，作用是保证对 Vuex 中 state 的修改只能在 mutation 的回调函数中，而不能在外部随意修改state。  
**this._actions** 用来存放用户定义的所有的 actions。  
**this._mutations** 用来存放用户定义所有的 mutatins。  
**this._wrappedGetters** 用来存放用户定义的所有 getters。  
**this._modules** 用来存储用户定义的所有modules  
**this._modulesNamespaceMap** 存放module和其namespace的对应关系。  
**this._subscribers** 用来存储所有对 mutation 变化的订阅者。  
**this._watcherVM** 是一个 Vue 对象的实例，主要是利用 Vue 实例方法 $watch 来观测变化的。  
这些参数后面会用到，我们再一一展开。   

继续往下看：
```javascript
// bind commit and dispatch to self
const store = this
const { dispatch, commit } = this
this.dispatch = function boundDispatch (type, payload) {
  return dispatch.call(store, type, payload)
}
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}
```
如同代码的注释一样，绑定Store类的dispatch和commit方法到当前store实例上。dispatch 和 commit 的实现我们稍后会分析。this.strict 表示是否开启严格模式，在严格模式下会观测所有的 state 的变化，建议在开发环境时开启严格模式，线上环境要关闭严格模式，否则会有一定的性能开销。

![](https://haitao.nos.netease.com/514ce1b831314a268dae031d987b42bc.png)

构造函数的最后：
```javascript
// init root module.
// this also recursively registers all sub-modules
// and collects all module getters inside this._wrappedGetters
installModule(this, state, [], this._modules.root)

// initialize the store vm, which is responsible for the reactivity
// (also registers _wrappedGetters as computed properties)
resetStoreVM(this, state)

// apply plugins
plugins.concat(devtoolPlugin).forEach(plugin => plugin(this))
```

##### Vuex的初始化核心
**installModule**

使用单一状态树，导致应用的所有状态集中到一个很大的对象。但是，当应用变得很大时，store 对象会变得臃肿不堪。

为了解决以上问题，Vuex 允许我们将 store 分割到模块（module）。每个模块拥有自己的 state、mutation、action、getters、甚至是嵌套子模块——从上至下进行类似的分割。


```javascript
// init root module.
// this also recursively registers all sub-modules
// and collects all module getters inside this._wrappedGetters
installModule(this, state, [], this._modules.root)
```
在进入installModule方法之前，有必要先看下方法的入参this._modules.root是什么。
```javascript
this._modules = new ModuleCollection(options)
```
这里主要用到了src/module/module-collection.js 和 src/module/module.js

module-collection.js:
```javascript
export default class ModuleCollection {
  constructor (rawRootModule) {
    // register root module (Vuex.Store options)
    this.root = new Module(rawRootModule, false)

    // register all nested modules
    if (rawRootModule.modules) {
      forEachValue(rawRootModule.modules, (rawModule, key) => {
        this.register([key], rawModule, false)
      })
    }
  }
  ...
}
```
module-collection的构造函数里先定义了实例的root属性，为一个Module实例。然后遍历options里的modules，依次注册。

看下这个Module的构造函数：
```javascript
export default class Module {
  constructor (rawModule, runtime) {
    this.runtime = runtime
    this._children = Object.create(null)
    this._rawModule = rawModule
    const rawState = rawModule.state
    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }
  ...
}
```
这里的rawModule一层一层的传过来，也就是new Store时候的options。  
module实例的_children目前为null，然后设置了实例的_rawModule和state。

回到module-collection构造函数的register方法, 及它用到的相关方法：
```javascript
register (path, rawModule, runtime = true) {
  const parent = this.get(path.slice(0, -1))
  const newModule = new Module(rawModule, runtime)
  parent.addChild(path[path.length - 1], newModule)

  // register nested modules
  if (rawModule.modules) {
    forEachValue(rawModule.modules, (rawChildModule, key) => {
      this.register(path.concat(key), rawChildModule, runtime)
    })
  }
}

get (path) {
  return path.reduce((module, key) => {
    return module.getChild(key)
  }, this.root)
}

addChild (key, module) {
  this._children[key] = module
}

```
get方法的入参path为一个数组，例如['subModule', 'subsubModule'], 这里使用reduce方法，一层一层的取值, this.get(path.slice(0, -1))取到当前module的父module。然后再调用Module类的addChild方法，将改module添加到父module的_children对象上。

然后，如果rawModule上有传入modules的话，就递归一次注册。

看下得到的_modules数据结构：

![](https://haitao.nos.netease.com/57f62e3975e34f679fdf2c4730c3197f.png)

扯了一大圈，就是为了说明installModule函数的入参，接着回到installModule方法。

```javascript
const isRoot = !path.length
const namespace = store._modules.getNamespace(path)
```
通过path的length来判断是不是root module。

来看一下getNamespace这个方法：
```javascript
getNamespace (path) {
  let module = this.root
  return path.reduce((namespace, key) => {
    module = module.getChild(key)
    return namespace + (module.namespaced ? key + '/' : '')
  }, '')
}
```
又使用reduce方法来累加module的名字。这里的module.namespaced是定义module的时候的参数，例如：
```javascript
export default {
  state,
  getters,
  actions,
  mutations,
  namespaced: true
}
```
所以像下面这样定义的store，得到的selectLabelRule的namespace就是'selectLabelRule/'
```javascript
export default new Vuex.Store({
  state,
  actions,
  getters,
  mutations,
  modules: {
    selectLabelRule
  },
  strict: debug
})
```

接着看installModule方法：
```javascript
// register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }
```
传入了namespaced为true的话，将module根据其namespace放到内部变量_modulesNamespaceMap对象上。

然后
```javascript
// set state
if (!isRoot && !hot) {
  const parentState = getNestedState(rootState, path.slice(0, -1))
  const moduleName = path[path.length - 1]
  store._withCommit(() => {
    Vue.set(parentState, moduleName, module.state)
  })
}
```
getNestedState跟前面的getNamespace类似，也是用reduce来获得当前父module的state，最后调用Vue.set将state添加到父module的state上。

看下这里的_withCommit方法：
```javascript
_withCommit (fn) {
  const committing = this._committing
  this._committing = true
  fn()
  this._committing = committing
}
```
this._committing在Store的构造函数里声明过，初始值为false。这里由于我们是在修改 state，Vuex 中所有对 state 的修改都会用 _withCommit函数包装，保证在同步修改 state 的过程中 this._committing 的值始终为true。这样当我们观测 state 的变化时，如果 this._committing 的值不为 true，则能检查到这个状态修改是有问题的。

看到这里，可能会有点困惑，举个例子来直观感受一下，以 Vuex 源码中的 example/shopping-cart 为例，打开 store/index.js，有这么一段代码：
```javascript
export default new Vuex.Store({
  actions,
  getters,
  modules: {
    cart,
    products
  },
  strict: debug,
  plugins: debug ? [createLogger()] : []
})
```
这里有两个子 module，cart 和 products，我们打开 store/modules/cart.js，看一下 cart 模块中的 state 定义，代码如下：
```javascript
const state = {
  added: [],
  checkoutStatus: null
}
```
运行这个项目，打开浏览器，利用 Vue 的调试工具来看一下 Vuex 中的状态，如下图所示：
![](https://haitao.nos.netease.com/6e138337b1a14b3fbbf39f24391b4570.png)


来看installModule方法的最后：
```javascript
const local = module.context = makeLocalContext(store, namespace, path)

module.forEachMutation((mutation, key) => {
  const namespacedType = namespace + key
  registerMutation(store, namespacedType, mutation, local)
})

module.forEachAction((action, key) => {
  const namespacedType = namespace + key
  registerAction(store, namespacedType, action, local)
})

module.forEachGetter((getter, key) => {
  const namespacedType = namespace + key
  registerGetter(store, namespacedType, getter, local)
})

module.forEachChild((child, key) => {
  installModule(store, rootState, path.concat(key), child, hot)
})

```
local为接下来几个方法的入参，我们又要跑偏去看一下makeLocalContext这个方法了：
```javascript
/**
 * make localized dispatch, commit, getters and state
 * if there is no namespace, just use root ones
 */
function makeLocalContext (store, namespace, path) {
  const noNamespace = namespace === ''

  const local = {
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (!store._actions[type]) {
          console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
          return
        }
      }

      return store.dispatch(type, payload)
    },

    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (!store._mutations[type]) {
          console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
          return
        }
      }

      store.commit(type, payload, options)
    }
  }

  // getters and state object must be gotten lazily
  // because they will be changed by vm update
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}

```
就像方法的注释所说的，方法用来得到局部的dispatch，commit，getters 和 state， 如果没有namespace的话，就用根store的dispatch, commit等等

以local.dispath为例：  
没有namespace为''的时候，直接使用this.dispatch。有namespace的时候，就在type前加上namespace再dispath。

local参数说完了，接来是分别注册mutation，action和getter。以注册mutation为例说明：
```javascript
module.forEachMutation((mutation, key) => {
  const namespacedType = namespace + key
  registerMutation(store, namespacedType, mutation, local)
})
```

```javascript
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    handler(local.state, payload)
  })
}
```
根据mutation的名字找到内部变量_mutations里的数组。然后，将mutation的回到函数push到里面。  
例如有这样一个mutation：  
```javascript
mutation: {
  increment (state, n) {
    state.count += n
  }
}
```
就会在_mutations[increment]里放入其回调函数。

##### commit
前面说到mutation被放到了_mutations对象里。接下来看一下，Store构造函数里最开始的将Store类的dispatch和commit放到当前实例上，那commit一个mutation的执行情况是什么呢?
```javascript
  commit (_type, _payload, _options) {
    // check object-style commit
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    const entry = this._mutations[type]
    if (!entry) {
      console.error(`[vuex] unknown mutation type: ${type}`)
      return
    }
    this._withCommit(() => {
      entry.forEach(function commitIterator (handler) {
        handler(payload)
      })
    })
    this._subscribers.forEach(sub => sub(mutation, this.state))

    if (options && options.silent) {
      console.warn(
        `[vuex] mutation type: ${type}. Silent option has been removed. ` +
        'Use the filter functionality in the vue-devtools'
      )
    }
  }
```
方法的最开始用unifyObjectStyle来获取参数，这是因为commit的传参方式有两种：
```javascript
store.commit('increment', {
  amount: 10
})
```
提交 mutation 的另一种方式是直接使用包含 type 属性的对象：
```javascript
store.commit({
  type: 'increment',
  amount: 10
})
```
```javascript
function unifyObjectStyle (type, payload, options) {
  if (isObject(type) && type.type) {
    options = payload
    payload = type
    type = type.type
  }

  assert(typeof type === 'string', `Expects string as the type, but found ${typeof type}.`)

  return { type, payload, options }
}
```
如果传入的是对象，就做参数转换。  
然后判断需要commit的mutation是否注册过了，this._mutations[type]，没有就抛错。  
然后循环调用_mutations里的每一个mutation回调函数。  
然后执行每一个mutation的subscribe回调函数。
![](https://haitao.nos.netease.com/e2c9cf87df4b411dad45393fc3fa3ece.png)


##### Vuex辅助函数
Vuex提供的辅助函数有4个：

![](https://haitao.nos.netease.com/1192cba48d974c1d975f91e91a99d8f4.png)

以mapGetters为例，看下mapGetters的用法：  
![](https://haitao.nos.netease.com/0ebaa5858ea5409ca3e98b414cc2a655.png)

代码在src/helpers.js里：
```javascript
export const mapGetters = normalizeNamespace((namespace, getters) => {
  const res = {}
  normalizeMap(getters).forEach(({ key, val }) => {
    val = namespace + val
    res[key] = function mappedGetter () {
      if (namespace && !getModuleByNamespace(this.$store, 'mapGetters', namespace)) {
        return
      }
      if (!(val in this.$store.getters)) {
        console.error(`[vuex] unknown getter: ${val}`)
        return
      }
      return this.$store.getters[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})


function normalizeMap (map) {
  return Array.isArray(map)
    ? map.map(key => ({ key, val: key }))
    : Object.keys(map).map(key => ({ key, val: map[key] }))
}

function normalizeNamespace (fn) {
  return (namespace, map) => {
    if (typeof namespace !== 'string') {
      map = namespace
      namespace = ''
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/'
    }
    return fn(namespace, map)
  }
}

```
normalizeNamespace方法使用函数式编程的方式，接收一个方法，返回一个方法。  
mapGetters接收的参数是一个数组或者一个对象：
```javascript
computed: {
// 使用对象展开运算符将 getters 混入 computed 对象中
  ...mapGetters([
    'doneTodosCount',
    'anotherGetter',
    // ...
  ])
}
```
```javascript
mapGetters({
  // 映射 this.doneCount 为 store.getters.doneTodosCount
  doneCount: 'doneTodosCount'
})
```
这里是没有传namespace的情况，看下方法的具体实现。  
normalizeNamespace开始进行了参数跳转，传入的数组或对象给map，namespace为'' , 然后执行fn(namespace, map)  
接着是normalizeMap方法，返回一个数组，这种形式：
```javascript
{
  key: doneCount,
  val: doneTodosCount
}
```
然后往res对象上塞方法，得到如下形式的对象：
```javascript
{
  doneCount: function() {
    return this.$store.getters[doneTodosCount]
  }
}
```
也就是最开始mapGetters想要的效果：

![](https://haitao.nos.netease.com/4b7824f97a1c4b23a0417794be851a59.png)

#### 完

by kaola/fangwentian

