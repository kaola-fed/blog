---
title: vuejs学习之路
date: 2017-04-30
---

>最近在做的种草项目，其中的管理后台采用了`vue.js`。为什么会选用vue呢，一是因为一个管理后台，不需要考虑兼容性、seo这些前台系统需要关心的点，后台系统的交互形式又特别适合做成一个单页应用，而vue不兼容IE8及以下版本，vue的route库很好的支持单页应用，适合我们的项目；再者，当然是因为vue是目前前端圈子很火很fashion的技术框架呀，截至目前有接近7k的fork，开发者们的参与热情堪称空前，技术社区氛围也很好，持续的版本更新，是一款被各大小项目考验过的稳定框架，岂有不用之理？

<!-- more -->

在此之前，我也没有过vue的项目实践经验，当初自己任性的敲定用vue的时候，内心还是很忐忑的，毕竟实际的开发时间也就一周。不过，学习一个新的框架或技术最好的方式，还是要运用到实际中，去学去写，才有更深的认识和体验。在项目时间很紧迫的情况下，会逼着自己快速的学习一个新框架，当然也因为要保证提测时间而没有做过的思考设计，在使用和理解这个框架的时候，也会遇到一些问题。

下面记录下，我在本项目中使用vuejs的过程。

### 体验vue
在确认选用vue之前，我没有完整的看过教程或api文档，只是先去使用了他的[命令行工具](https://cn.vuejs.org/v2/guide/installation.html#命令行工具)，本地跑起来一个demo工程，来揭开vue神秘的面纱，直观的初步认识下这个框架。简单体验完之后，我就决定用vue了，原因有三：

- 使用命令行工具，可以快速搭建一个单页应用，这正是我所想要的，只需要在初始化你的项目时选择Install vue-router就可以了；命令行工具生成的工程目录结构，也是符合前端开发习惯的，如图1。
- vue提供了一套完整的前端项目构建方式，包括不同生产环境的webpack打包配置，还可以配置eslint、unit test等可选项，可以说是省去了我们很多的前期准备和后顾之忧。
- 在生产环境下，可以启动`一个带热重载、保存时静态检查`的服务器，做到前后端完全分离开发。

总之，使用vue可以构建一个目前比较主流的现代化的前端开发环境。

![image](http://chuantu.biz/t5/76/1493567172x2890171636.png) 图1


### 开起我的vue工程
在体验过vue的简单实例之后，我依然没有立马去通读开发教程，而是在自己的工程里，用命令行搭建我的webapp。此间，参照了成功实践过vue的数据营销系统，借用了其mock数据的配置方式，了解他们系统中单页应用中模块与路由的配置，知道了vuex及其中的state、action、mutation等概念，老实说，一下子接触到这许多内容，非常凌乱，想来只有官方教程才能解惑了，但我并不急着去读文档，因为我想结合工程带着疑问再去看文档，先把页面框架搭起来。

单从工程目录中，基本能知道`index.html`是单页应用的承载页，`main.js`是页面入口脚本。在main.js里初始化了一个vue实例：

```
import Vue from 'vue'
import App from './App'
import router from './router'

Vue.config.productionTip = false

/* eslint-disable no-new */
new Vue({
  el: '#app',
  router,
  template: '<App/>',
  components: { App }
})
```

其中传入了router配置以及App.vue模板，
```
<template>
  <div id="app">
    <img src="./assets/logo.png">
    <router-view></router-view>
  </div>
</template>
```
其中有一个`<router-view>`组件，用来渲染路径匹配到的视图组件。首先，开始编写我的页面模板。常规的后台有顶栏、侧边栏和主内容区，其中主内容区是需要根据不同的路由切换的，可以用`<router-view>`组件，页面模板如下：

```
<template>
  <div id="app">
    <header></header>
    <div class="g-body" :class="{noHasSideNav: noHasSideNav}">
      <side-nav @toggle="toggleNav"></side-nav>
      <router-view></router-view>
      <span class="u-totop" @click="goToTop">Top</span>
    </div>
  </div>
</template>
```
其中`header`和`side-nav`分别是顶栏和侧边导航栏组件。再配上对应的路由，基本搭好了一个单页应用的框架。
接下来，带着工程中已经接触的概念和疑问，去看官方教程。


### 学习官方教程
vue的[教程](https://cn.vuejs.org/v2/guide/)很完整，由浅入深。集中花半天一天的时间，能基本看完。但在工作中抽出这么完整的时间有难度，而且一口气看完容易精神疲劳。可以选择重点的章节重点看，比如`组件`章节，其他章节快速翻阅，等用到的时候再来搜查细看。另外vue-router、vueX这些也需要花时间去看文档，理解设计意图，再结合实践到项目中去。


### 组件与功能开发

通过教程对vue有了更多了解以后，可以边学边试着开发主要功能，并编写通用组件。
开始我的第一个组件，状态Tab切换：
```
<template>
  <div class="m-statustabs">
    <ul class="tablst f-cb">
      <li :class="['tabitm',activeStatus==lb.status?'is-active':'']" v-for="lb in labelList" @click="tabclick(lb.status, $event)">{{lb.label}}</li>
    </ul>
  </div>
</template>

<script>
  import { mapMutations, mapGetters, mapState } from 'vuex'
  export default {
    name: 'status-tab',
    props: {
      labelList: Array
    },
    data () {
      return {
        activeStatus: this.labelList[0]&&this.labelList[0].status
      }
    },
    methods: {
      tabclick (val,evt) {
        if(val===this.activeStatus)
          return
        this.activeStatus = val
        this.$emit('tabchange',this.activeStatus)
      }
    }
  }
</script>

```
很简单的一个tab切换功能，需要设置传入的参数`props`，再绑定点击事件，事件中处理状态值，并对外emit事件，在父组件中使用此组件：
```
<status-tab :labelList="statusList" @tabchange="onTabChange"></status-tab>

```
之后就是具体功能的开发以及其他组件的设计实现。期间也遇到过一些问题，先网络查找定位，自己解决不了的，询问同事，在此特别感谢鸡苗同学，在我使用vuejs的过程中，给予的指导与启发，比如在使用`v-for`循环输入列表后，删除数组项没有更新到视图的问题，后与鸡苗一起讨论定位，给列表设置key属性后能正常删除，后自己定位到是因为组件内部用了`v-once`指令，导致视图不再更新，此指令要慎用，能确保节点不需要更新的，才加此指令。

### 总结
截至目前，种草后台1期的功能已开发联调完成。但我对vue只是到了一个会使用基本功能的阶段，之前教程看得粗浅，有些项目中没涉及到的功能并没有去细看，还有此框架的设计思路及内部实现，都还没有去深入了解，这都是后续需要继续进行的事。

