---
title: vue学习笔记之组件通信
date: 2017-12-25
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
组件之间的通信可以用下图表示：
![](https://haitao.nos.netease.com/14166f31-f2b8-4bb5-bea1-d16336e51631.png)
由此图可知，组件通信可分为：父子组件通信、兄弟组件通信、跨级组件通信。

一.父子组件通信
父组件向子组件通信，通过props传递数据就可以了；子组件则通过$emit来向父组件抛出事件（父组件需要监听相应的事件）。
看下以下实例：
```
// parent.vue
<template>
    <div class="parent">
        <p>父亲:给你{{ money }}元零花钱</p>
        <kid :money="money" @repay="repay"></kid>
        <br>
        <button @click="add">那给你加100</button>
        <p v-if="back" @repay="repay">儿子：超过300我不要，还给你 {{ back }}元</p>
    </div>
</template>
<script>
    export default {
        name: 'parent',
        data () {
            return {
                money: 100,
                back: 0
            }
        },
        components: {kid},
        methods: {
            repay (back) {
                this.back = back
            },
            add(){
                this.money += 100;
            }
        }
    }
</script>
```
```
// kid.vue
<template>
    <div class="parent">
        <p v-if="pocketMoney < 300">儿子: {{ pocketMoney }}太少了，不够 </p>
        <p v-else>儿子: 谢谢老爸！</p>
        <button v-if="pocketMoney > 300" @click="repay">点击有惊喜</button>
    </div>
</template>
<script type="text/javascript">
    export default {
        name: 'kid',
        props: {
            money: Number
        },
        data () {
            return {
            }
        },
        computed: {
            pocketMoney () {
                return this.money;
            }
        },
        methods: {
            repay () {
                this.$emit('repay',this.pocketMoney-300)
            }
        }
    }
</script>
```
父亲给儿子零花钱，父亲通过props将钱给到儿子，儿子拿到钱后通过计算属性得到自己的零花钱；父亲觉得零花钱太少了，给儿子超过300块；儿子比较乖，只需要300，于是发送了一个自定义事件repay，将超过300块的钱还给父亲，父亲通过监听repay这个事件的回调拿到儿子还给他的钱。

二.非父子组件通信（兄弟组件通信、跨多级组件通信）
Vue.js 1.x中提供了两个方法$dispatch()和$broadcast()。$dispatch()用于向上级派发事件，只要是它的父级（一级或多级），都可以在Vue实例的events选项内接收；同理，$broadcast()是由上级向下级广播事件的，用法与$dispatch()完全一致，只是方向相反。这两种方法一旦发出事件后，任何组件都是可以接收到的，根据就近原则，会在第一次接收到后停止冒泡、除非返回true。
    这两个方法虽然看起来很好用，但在Vue.js 2.x中被废弃了，因为基于组件树结构的事件流方式让人难以理解，并且在组件结构扩展的过程中会变得越来越脆弱，并且不能解决兄弟组件通信的问题。
    在Vue.js 2.x中，推荐使用一个空的Vue实例作为**中央事件总线（bus）**，就跟房屋中介一样：租房者和出租者都会找中介登记，中介把匹配的租房者信息给出租者、把匹配的出租者的信息给租房者。整个过程中，租房者和出租者并没有任何角落，都是通过中介来传话的。另外，假如租房者打算换房，租房者找中介登记，并订阅跟租房者需求有关的资讯，一旦有符合需求的房子出现，中介就会通知你，并传达该房子的具体信息。
    在这两个例子中，租房者和出租者担任的就是两个跨级（一级或多级）组件，而房产中介就是这个中央事件总线（bus）。示例代码如下：
```
<div id="app">
    {{ message }}
    <renter></renter>
</div>
<script>
    var bus = new Vue();

    //出租者组件
    Vue.component('renter', {
        template: '<button @click="handleEvent">传递信息</button>',
        methods: {
            handleEvent: function () {
                bus.$emit('on-message', '来自出租者的信息')
            }
        }
    });

    //租房者实例
    var app = new Vue({
        el: '#app',
        data: {
            message: ''
        },
        mounted: function () {
            var that = this;
            //在实例初始化时，监听来自bus实例的事件
            bus.$on('on-message', function(msg){
                that.message = msg;
            });
        }
    });
</script>
```
这种方法巧妙而轻量地实现了任何组件间的通信，包括父子、兄弟、跨级。当我们的项目比较大时，可以选择更好的状态管理解决方案vues。
除了bus外，还有两种方法可以实现组件间通信：父链和子组件索引。
   
**父链**
在子组件中，使用this.$parent可以直接访问该组件的父实例或组件，父组件也可以通过this.$children访问它所有的子组件，而且可以递归向上或向下无限访问，直到根实例或最内层的组件。尽管Vue允许这样操作，但在业务中子组件应该尽可能地避免依赖父组件的数据，更不应该去主动修改它的数据，因为这样使得父子组件紧耦合：只看父组件，很难理解父组件的状态，因为它可能被任意组件修改。理想情况下，只有组件自己能修改它的状态。父子组件最好还是通过props和$emit来通信。

**子组件索引**
当子组件较多时，通过this.$children来一一遍历出我们需要的一个组件实例是比较困难的，尤其是组件动态渲染时，它们的序列是不固定的。Vue提供了子组件索引的方法，用特殊的属性ref来为子组件指定一个索引名称。用法：在父组件模板中，子组件标签上使用ref指定一个名称，并在父组件内通过this.$refs来访问指定名称的子组件。
    注意：$refs只在组件渲染完成之后才填充，并且它是非响应式的，它仅仅作为一个直接访问子组件的应急方案，应当避免在模板或计算属性中使用$refs。

总结，父子组件通信使用props和this.$emit；兄弟组件通信、跨级组件通信可以使用中央事件总线（bus）、子组件索引，复杂项目可以使用vuex。

by Frida

参考资料：
1.Vue.js实战 梁灏
2.[http://whutzkj.space/2017/08/05/vue%E7%BB%84%E4%BB%B6%E4%B9%8B%E9%97%B4%E7%9A%84%E9%80%9A%E4%BF%A1%EF%BC%88%E4%B8%80%EF%BC%89/#more](http://whutzkj.space/2017/08/05/vue%E7%BB%84%E4%BB%B6%E4%B9%8B%E9%97%B4%E7%9A%84%E9%80%9A%E4%BF%A1%EF%BC%88%E4%B8%80%EF%BC%89/#more)