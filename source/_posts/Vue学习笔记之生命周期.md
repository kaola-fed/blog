---
title: Vue学习笔记之生命周期
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

我们知道，通过构造函数Vue就可以创建一个Vue的根实例，并启动Vue应用：
```
var app = new Vue({
    el: ‘#app’,
    data: myData
});
```
其中变量app就代表了这个Vue实例。
每个Vue实例创建时，都会经历一系列的初始化过程，同时也会调用相应的生命周期钩子，我们可以利用这些钩子，在合适的时机执行我们的业务逻辑。下图是官网的Vue生命周期示例图。
![](https://haitao.nos.netease.com/7e1bb009-1195-466e-a425-e44de2ef0eb4.png)

Vue生命周期，顾名思义，就是指Vue实例从创建到销毁的过程。我们来简单翻译下这张图：
![](https://haitao.nos.netease.com/afee0a6a-932b-40f2-863a-d1564a625c43.png)
注：上图来自[https://www.cnblogs.com/fly_dragon/p/6220273.html](https://www.cnblogs.com/fly_dragon/p/6220273.html)
    
在上图中红色框中的钩子是Vue提供的可以注册的钩子，包括：
1.beforeCreate：在实例初始化之后，数据观测（Data Observer）和event/watch事件配置之前被调用；
2.created：该钩子在实例已经创建完成之后被调用。在这一步，实例已完成这些配置：数据（Data Observer）、属性和方法的运算，event/watch事件回调。但尚未挂载，实例存在、但$el还不可用（需要初始化处理一些数据时会比较常用这个钩子）；
3.beforeMount：在挂载开始之前被调用，相关的render函数首次被调用；
4.mounted：el挂载到实例上后调用，一般我们的第一个业务逻辑会在这里开始；
5.beforeUpdate：数据更新时调用，发生在虚拟DOM重新渲染和打补丁之前。我们可以在这个钩子中进一步地更改状态，这不会触发附加的重新渲染过程；
6.updated：由于数据更改导致的虚拟DOM重新渲染和打补丁，在这之后会调用该钩子。当这个钩子被调用时，组件DOM已经更新，所以我们现在可以执行依赖于DOM的操作。然而在大多数情况下，我们应该避免在此期间更改状态，因为这可能会导致更新无限循环。该钩子在服务器端渲染期间不会被调用；
7.beforeDestroy：实例销毁之前调用，主要解绑一些使用addEventListener监听的事件等；
8.destroyed：实例销毁之后调用。调用之后，Vue实例指示的所有东西都会解绑，所有事件监听器都会被移除，所有的子实例也会被销毁。该钩子在服务端渲染期间不会被调用。

举一个简单的例子：
```
var app = new Vue({
    el: '#app',
    data() {
        return {
            name: 'Frida'
        }
    },
    beforeCreate: function () {
        console.log('创建前...')
        console.log(this.name)
        console.log(this.$el)
    },
    created: function () {
        console.log('已创建')
        console.log(this.name)
        console.log(this.$el)
    },
    beforeMount: function () {
        console.log('挂载之前')
        console.log(this.name)
        console.log(this.$el)
    },
    mounted: function () {
        console.log('挂载完成')
        console.log(this.name)
        console.log(this.$el)
    },
    beforeUpdate: function () {
        console.log('更新前')
        console.log(this.name)
        console.log(this.$el)
    },
    updated: function () {
        console.log('更新完成')
        console.log(this.name)
        console.log(this.$el)
    },
    beforeDestroy: function () {
        console.log('销毁前')
        console.log(this.name)
        console.log(this.$el)
    },
    destroyed: function () {
        console.log('销毁之后')
        console.log(this.name)
        console.log(this.$el)
    }
});
```

控制台打印结果：
![](https://haitao.nos.netease.com/064430f9-8bcf-42c3-9469-a24ab9fd5113.png)
接着在控制台修改data的值，控制台打印结果发生变化：
![](https://haitao.nos.netease.com/335371d3-26d9-4aa1-b7b9-82919608f356.png)
然后执行app.$destroy()：
![](https://haitao.nos.netease.com/ca49153f-36f5-4fec-9961-a5a3fc639c9b.png)
对以上过程进行总结：
![](https://haitao.nos.netease.com/d1995b0e-f7ba-4db7-b244-9b06f773f0dd.png)
注：上图来自[https://www.cnblogs.com/gagag/p/6246493.html](https://www.cnblogs.com/gagag/p/6246493.html)

此文仅是笔者在学习vue的过程中的学习笔记，我们仍需要在实践中不断深入学习相关知识点。

by Frida

参考资料：
1.[https://cn.vuejs.org/v2/guide/instance.html](https://cn.vuejs.org/v2/guide/instance.html)
2.Vue.js实战 梁灏
3.[https://www.cnblogs.com/fly_dragon/p/6220273.html](https://www.cnblogs.com/fly_dragon/p/6220273.html)
4.[https://www.cnblogs.com/gagag/p/6246493.html](https://www.cnblogs.com/gagag/p/6246493.html)
