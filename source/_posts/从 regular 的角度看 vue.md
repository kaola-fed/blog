---
title: 从 regular 的角度看 vue
date: 2017-05-05
---
vue + vuex + vue-router 一些使用方式文档都已经介绍得非常详细了。
https://cn.vuejs.org/v2/guide  
http://router.vuejs.org/zh-cn/essentials/getting-started.html  
https://vuex.vuejs.org/zh-cn/  
下面说一点关于 vue 和 regular 的相同点和不同点

<!-- more -->

1. vue 的插值和 regular 的插值语法类似，但是 vue 的绑定一次是 `v-once` 而且这个指令会直接影响整个节点上的数据。   
事件和条件渲染、列表渲染的语法都不太一样，没有 regular 的语法那么自由，在属性上的绑定用的是指令 v-bind: 而且字段用“”包起来，比如`<div v-bind:id="dynamicId" v-bind:href="dynamicId"></div>`。 
 
个人感觉这里这么做是因为 regular 和 vue 对模板的渲染过程是不一样的，这个后面再说。对于插值 vue 仅仅只能处理文本，我猜这个插值最后也是转换成了指令来做的，并没有类似 regular 中的 expression 对象。   
对于条件渲染 vue 的语法是 `v-if`，比如`<p v-if="seen">Now you see me</p>`表达式的真假直接控制整个节点（而且必须控制整个节点）的插入和移除，这么做的话如果想要实现 regular 里面的`<p>Now you see {#if seen} me {/if}</p>`就得给 me 这个文本包一层节点， 因为 vue 对于 model => view 的过程是把 template string 先解析成 dom 树， 然后解析各种组件和指令（template里面就只剩指令了），指令都是和 dom 树里面的某个节点绑定的。  
对于列表渲染也是的
```
<ul id="example-1">
  <li v-for="item in items">
    {{ item.message }}
  </li>
</ul>
``` 
是对整个节点的重复，对于数组来说 `v-for="(index, item) in items"` index 就相当于 regular 里面的 item_index，对于对象来说 `v-for="(value, key, index) in object"` 
，对于 vue 里面的过滤器，只能用在插值和 v-bind 的指令中。  
对于事件处理器，不同于 regular 里面的 on-xxx={this.huidiao($event)}， vue 里面用的是指令 v-on:xxx="huidiao($event)"  
2. vue 中双向绑定的实现和 regular 中的脏值检测不一样，是用 definePorperty 重写了 getter/setter函数来实现的，并没有像 regular 中的 $update 一样的全局解药，所以在 vue 中所有的修改必须让 vue 知道。对于一些数组和对象的操作，有一些 vue 是检测不到的，
```
1. 当你利用索引直接设置一个项时，例如： vm.items[indexOfItem] = newValue
2. 当你修改数组的长度时，例如： vm.items.length = newLength
```
所以得用 vue 暴露出来的方法来修改数据。为了解决第一类问题，以下两种方式都可以实现和 vm.items[indexOfItem] = newValue 相同的效果， 同时也将触发状态更新：
```
Vue.set(example1.items, indexOfItem, newValue)
example1.items.splice(indexOfItem, 1, newValue)
```
为了解决第二类问题，你也同样可以使用 splice：
```
example1.items.splice(newLength)
```
对比一些思路：  
regular 的 compile 过程是把 template string 整个的去编译，每当碰到插值，或者指令里面的数据时会生成一个 expression 对象，并生成一个 watcher 推到脏值检测的队列里面，每次进入 digest 过程是就逐个的去检测。  
vue 的 compile 过程是将 template string 编译成一棵 dom 树，然后去匹配所有的指令，将指令和对应的 dom 节点关联起来，每一次匹配到表达式的时候也会生成一个 watcher ，在 vue 的 data 函数阶段会用 definePorperty 将每个数据的 getter/setter 重写。这里用订阅者模式，当该字段赋值之后 set 函数触发，发出消息，节点的 watcher 是订阅者，收到消息之后执行对应的操作。
3. regular 的 config 方法在 vue 中可以用 data 方法来替代
4. regular 中的 init 基本上和 vue 中的 beforeMount 方法一样
5. vue 中组件接受参数需要显示的声明 props ，这个字段的值是一个数组，包含所有接收的值以及值的一些信息（非必填），比如
```
Vue.component('example', {
  props: {
    // 基础类型检测 （`null` 意思是任何类型都可以）
    propA: Number,
    // 多种类型
    propB: [String, Number],
    // 必传且是字符串
    propC: {
      type: String,
      required: true
    },
    // 数字，有默认值
    propD: {
      type: Number,
      default: 100
    },
    // 数组／对象的默认值应当由一个工厂函数返回
    propE: {
      type: Object,
      default: function () {
        return { message: 'hello' }
      }
    },
    // 自定义验证函数
    propF: {
      validator: function (value) {
        return value > 10
      }
    }
  }
})
```
6. vue 中所用到的字段都必须在 data 函数中显示的声明，如果在模板中使用了没有被定义的数据 vue 会有警告
7. vue 中一个组件用到另外一个组件也得显示的声明一个 components ，包含用到的所有组件，比如
```
components: {
  'component-name': Component,
}
```
8. 在 regular 中只要声明了组件的 name 属性，那么这个组件就自动被注册到全局去了，但是在 vue 里面是没有这么一个属性的，如果想要注册到全局只能`Vue.component('my-component', {})`
9. vue 的方法都放在 methods 里面，不过调用的时候都是 `this.xxx`
10. 在 vue 中数据不是 `this.data.xxx` 而是直接 `this.xxx`
11. 在给内嵌组件传递参数的时候，这个时候参数是一个属性，应该用 `v-bind:message="messgae"` 
12. vue 在组件之前的数据默认是单向的，但是如果传入的是一个对象那么子组件修改这个对象会影响到父组件。传入的参数在子组件内部是不推荐修改的，如果要改的话最好是在子组件内部定义另外一个变量，用计算属性或者直接定义一个一样的变量去代替他。如果子组件想要把修改的数据传回去，就得自定义事件去做，自定义事件和 regular 里面的自定义事件类似，比如
```
<div id="counter-event-example">
  <p>{{ total }}</p>
  <button-counter v-on:increment="incrementTotal"></button-counter>
  <button-counter v-on:increment="incrementTotal"></button-counter>
</div>

Vue.component('button-counter', {
  template: '<button v-on:click="increment">{{ counter }}</button>',
  data: function () {
    return {
      counter: 0
    }
  },
  methods: {
    increment: function () {
      this.counter += 1
      this.$emit('increment')
    }
  },
})
new Vue({
  el: '#counter-event-example',
  data: {
    total: 0
  },
  methods: {
    incrementTotal: function () {
      this.total += 1
    }
  },
  beforeMount() {
    var vm = new Counter();
    vm.$on('increment', this.incrementTotal);
  },
})
```
13. 对于组件之间的通信，如果是非父子组件会有点麻烦，这个时候可以进入一个中间组件专门用来通信使用，regular 中也适用
```
var bus = new Vue()


// 触发组件 A 中的事件
bus.$emit('id-selected', 1)


// 在组件 B 创建的钩子中监听事件
bus.$on('id-selected', function (id) {
  // ...
})
```