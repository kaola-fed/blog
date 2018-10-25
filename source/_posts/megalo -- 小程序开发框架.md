# megalo -- 小程序开发框架

megalo 是基于 Vue 的小程序框架（没错，又是基于 Vue 的小程序框架），但是它不仅仅支持微信小程序，还支持支付宝小程序，同时还支持在开发时使用更多 Vue 的特性。

## 背景

对于用户而言，小程序能提供更好的体验，但对于开发者而言，要让一个应用跑在多个平台上，则需要写多套代码。如何提高小程序开发效率让很多开发者都感到头疼。

业界也有相关的解决方案，如 taro 和 mpvue，二者都是基于 react 和 vue 的开发模式实现，让开发者能够以他们熟知的 react 或 vue 模式来开发小程序，提高开发效率。

mpvue 的发布给了我们很多启发，更早的时候，我们基于 RegularJS（网易自研的前端框架）开发了一个名为 mpregular 的小程序框架。在 mpregular 的开发和实际使用过程中，我们发现如果小程序框架所支持的特性只是原框架的子集（例如不支持 filter、模版复杂表达式等），开发效率会大打折扣。

所以，我们在方案上做了很多尝试，目的是支持更多的特性，减少小程序与 H5 开发之前的差异。目前 mpregular 已经在考拉的小程序业务中大量应用，相关业务的开发同学纷纷表示，学习成本变低，跨端业务（H5 和小程序）的开发效率提升近一倍。

方案经过一段时间验证后，我们决定把这套方案用 vue 再实现一次，一是为了适应技术栈的变更升级，二是为社区做一点微小的工作，于是就便有了 megalo。

## 特性

### 支持更多模版语法特性

目前为止，megalo 支持的模版语法特性如下

特性 | 小程序 | mpvue | megalo |
:--- |:---  |:---|:---
computed 计算属性 | ❌ | ⭕ | ⭕️
v-model 双向绑定 | ❌ | ⭕ | ⭕️
slot 插槽 | ❌ | ⭕ | ⭕️
scoped-slot 插槽 | ❌ | ❌ | ⭕️
filter 过滤器 | ❌ | ❌ | ⭕️
v-html 富文本 | ❌ | ❌ | ⭕️
复杂表达式插值 | ❌ | ❌ | ⭕️

从表格可以看到，megalo 最大的特点之一是，支持更多的 Vue 语法特性。这意味着，如果你有一个需求是要把现有的 Vue 代码迁移到小程序上，不需要太多改动。因为你的代码中可能大量使用 filter、scoped-slot、复杂表达式插值。

#### 基本语法

支持 vue 的基本面模版月，包括 v-for、v-if。class 和 styel 的绑定方式没有限制，官方的用法都支持。

```html
<!-- v-if & v-for -->
<div v-for="(item, i) in list">
  <div v-if="isEven(i)">{{ i }} - {{ item }}</div>
</div>

<!-- style & class -->
<div :class="classObject"></div>
<div :class="{ active: true }"></div>
<div :class="[activeClass, errorClass]"></div>
<div :style="{ color: activeColor, fontSize: fontSize + 'px' }"></div>
<div :style="styleObject"></div>
<div :style="[baseStyles, overridingStyles]"></div>
```

#### slot & scoped-slot

支持 slot 和 scoped-slot。

```html
<div>
  <Container>
    <Card>
      <div slot="head"> {{ title }} </div>
      <div> I'm body </div>
      <div slot="foot"> I'm footer </div>
    </Card>
  </Container>
  <List :list="list">
    <span slot-scope="scopeProps">{{ scopeProps.item.label }}</span>
  </List>
<div>
```

#### 复杂表达式 & filter

可以在模版里面写复杂表达式、调用实例上的方法，当然也可以用更简洁的 filter 语法，跟平时用 Vue 开发一样。

```html
<div>
  <div>{{ message.toUpperCase() }}</div>
  <div>{{ toUpperCase( message ) }</div>
  <div>{{ message | toUpperCase }}</div>
</div>
```

#### v-html

要使用 `v-html` 需要添加插件 `@megalo/vhtml-plugin`，并引入模版解析库 `octoparse`，在页面入口安装一下插件：

```javascript
import Vue from 'vue'
import VHtmlPlugin from '@megalo/vhtml-plugin'

Vue.use(VHtmlPlugin)
```

利用 `v-html` 指令然后就可以在小程序中渲染 html 了。

```html
<div v-html="'<h1>megalo</h1>'"></div>
```

### 更好的数据更新性能

小程序的官方明确说明，在调用 setData 更新数据时如果数据量过大或频率更高，会引发性能问题。megalo 在框架底层已经帮开发者对此进行优化，每次数据发生变化时，megalo 只会将视图中要展示的且发生变化的数据进行更新，将 setData 的数据更新量最小化，同时对更新数据频率进行了限制。

像下面这段代码，如果视图只需要展示 `user.name` 这个字段的话，在进行数据同步时只会将 `user.name` 这个字符串更新到视图层，其余字段是不会同步到小程序的对象上的。

```html
<div>{{ user.name }}</div>
<script>
export default {
  data() {
    return {
      user: {
        name: 'kaola',
        age: 3,
        favorite: [
          'encalyptus',
          'sleeping'
        ]
      }
    }
  }
}
</script>
```

### 支持更多平台

今年以来，各大流量平台都在小程序领域有所动作，蚂蚁金服成立小程序事业部，百度、今日头条也纷纷推出自己的小程序。megalo 目前已经支持微信和支付宝小程序，百度小程序等平台的支持也在计划当中。

<div style="display: flex; justify-content: space-around;">
  <figure>
    <img alt="1424614_old" src="https://haitao.nos.netease.com/eb33a105-cd7e-4ca8-ac25-2c241cc707cb_1441_764.gif" width="500px">
    <p>微信和支付宝小程序</p>
  </figure>

</div>

## 使用

使用 megalo 开发非常简单，只需在常见的 Vue 项目 webpack 构建配置上配置 `@megalo/target` 并引入 `@magalo/template-compiler` 即可。如果需要编译成支付宝小程序，只需要设置 `platform: 'alipay'`。

```javascript
const { createMegaloTarget } = require('@megalo/target');
module.exports = {
  target: createMegaloTarget( {
    compiler: require('@megalo/template-compiler'),
    platform: 'wechat'
  } )
  // 其他 webpack 配置，如 vue-loader 等
};
```

接着，就可以像开发 Vue 应用一样去开发小程序。示例可以参考 [megalo-demo](https://github.com/kaola-fed/megalo-demo)。

如果想用 typescript 进行开发，可以参考 [megalo-ts-simple](https://github.com/kaola-fed/megalo-examples/tree/master/ts-simple)。

## 实现

小程序在结构上主要有 Service(JavascriptCore) 和 View(WebView) 两部分组成（微信和支付宝小程序有着类似的结构，下文均以微信小程序为例，并简称为小程序），分别运行在独立的环境上，之间不具备共享数据通道，二者的通信方式是将数据封装在 js 脚本后传递。Page 实例就在 Service 中，通过 setData 方法将数据传递到 View。View 则通过事件绑定将视图层触发的事件传递给 Service。Service 层中无法操作视图层的 DOM 节点。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/a63df2e7-21ed-40e5-a2e6-1c6c900ac2ba_1343_367.jpg" width="500px">
</div>

实际开发中，小程序的逻辑和模版需要写在 .js 和 .wxml 两个文件中，分别在 Service 和 View 中执行。如果要将在浏览器端的 Vue 放到小程序中跑，需要将 .vue 文件中的 template 片段转换成 .wxml 文件，并对 Vue 的 runtime 部分改造，将其中的 DOM 操作移除，通过小程序的 Service 中的页面实例上的 API 与 View 进行通信。

最终的运行效果是，当 Vue 的 vm 上数据发生更新时，会重新渲染出 vdom，在的 patch 阶段，框架不在去操作 DOM，而是通过 Page 上的 setData 方法将变化的数据更新到视图层，完成 Vue 和 小程序的视图更新，这就是 megalo 底层所做的工作。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/99823784-11f8-4e4a-935b-9d58ba3e6474_2184_294.jpg" width="500px">
</div>

megalo 的实现，主要分成以下四个部分，下面本文将对每个部分进行介绍。

### 生命周期

小程序中，每一个页面（Page）是一个实例，页面的生命周期钩子有很多，但和实例创建的两个关键生命周期分别是 `onLoad` 和 `onReady`，它们分别在「`页面加载，实例初始化后`」和「`初次页面渲染完成`」时触发。Vue 的实例要和小程序实例建立起联系，则需要在小程序 Page 实例创建好以后，即在 `onLoad` 的钩子函数里，去初始化对应的 Vue 根实例，将页面实例 `page` 挂载到 Vue 实例的 `$mp` 上，此时也会触发 Vue 的生命周期钩子 `created`。在页面初次渲染完成后，则会调用 `$mount` 方法，与在浏览器中挂载 DOM 节点不同，这里会将 Vue 实例上的数据初始化到视图层中。由此，Vue 实例就与小程序的 Page 实例简历起了联系。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/c336b7e6-dd88-4162-b443-571d011e1737_1776_898.jpg" width="500px">
</div>

除了这两个生命周期钩子以外，像小程序的 `onShow`、`onHide` 等生命周期钩子在触发时，也会尝试触发 Vue 实例上的同名钩子函数，实现两种实例间生命周期的绑定。在小程序页面退出销毁时，会触发 `onUnload` 钩子，此时 Vue 的实例也会跟着销毁。

### 模版转换

小程序有它特有的模版语法和文件名后缀，所以在构建阶段，我们会将 .vue 文件中的 template 部分提取出来并转换成对应的 .wxml 文件。标签名、语法都会进行相应的转换，如图所示。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/f1428e67-7bc6-4465-9207-9659537073f6_2270_860.jpg" width="500px">
</div>

这一部分是在构建阶段完成的，这意味着，megalo 不支持 render 函数的写法。在构建阶段除了将模版转换成 .wxml 以外，还需要对模版中的每个节点进行转换，并在生产的 render 函数中加入相关的节点标记信息，数据映射和事件代理需要这些信息。

### 数据映射

由于无法直接操作视图层的 DOM，所以我们只能利用 page.setData 这个方法完成数据到视图层的映射。最简单暴力的方法，是将 Vue 实例上的所有数据统统收集起来，通过调用 page.setData 方法更新 Page 实例的数据，这个方法会将数据挂载到 Page 实例上，同时也会把数据传递给视图层。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/ebad8c36-32f1-456d-85bb-163900d2a966_1986_782.jpg" width="500px">
</div>

但是，这种粗暴的更新方式有两个弊端：

i. 全量更新 VM 上的数据是无法区分哪些数据是视图层需要的，冗余无用的数据会被更新到 page 实例上。像下图这个例子，视图层只需要展现两个字符串，如果 vm 上还存在两个大数组，它们也会被无脑同步到 page 上。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/afaf232b-1ad8-48d6-933f-af03bd302562_1998_818.jpg" width="500px">
</div>

ii. 同步到 page 实例上的数据其实就是原始数据，并不是视图层实际要展示的数据，所以展示数据的格式化与转换需要依赖小程序模版的解析能力，导致一些 Vue 支持的模版语法无法支持，例如 filter、复杂表达式、传递 class 对象等。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/43f0e97b-1e24-49e7-b7fc-d1996f961be7_1105_484.jpg" width="300px">
</div>

当然以上两个弊端不会对功能开发造成影响，但在实际的业务开发中，会让开发体验不一致，尤其是 h5 代码迁移到小程序时，对效率影响颇大。为了解决这个问题，megalo 采用另一中方式，即将 render 时生成的 vnode 上的数据更新到视图，vnode 的数据就是已经处理好的展示数据，根据 vnode 构造每个节点的数据结构，再同步到视图层。

例如以下这段代码，在构建阶段 megalo 会对每个节点进行标记，使 render 时生成的 vnode 和模版中每个插值能够对应上。

```html
<!-- 编译前的 Vue 模版 .vue -->
<div :class=“classObj”>
  {{ date | format( 'YYYY' ) }}
</div>

<!-- 编译后的小程序模版 .wxml -->
<view class="{{ node_1.class }}">
  {{ node_1.text }}
</view>
```

以这种方式实现的数据映射，只有视图层需要的两个字符串数据会被同步到小程序的 Page 实例上，其余数据则被认为与视图无关则不会进行同步。

```javascript
export default {
  data() {
    return {
      classObj: {
        'kaola': true
      },
      date: new Date(),
      users: {
        // big object
      }
    }
  }
}
```

如下图所示，Vue 渲染出来的 vnode 会被以特定的数据结构映射到 page 上，再同步到小程序视图层。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/2b289583-598c-440f-a0f0-0f45089e7717_1932_424.jpg" width="700px">
</div>

以这种方式实现的数据映射，可以更好地支持 Vue 的模版语法，且更大限度地减少更新视图时传输的数据量，从框架层面规避 setData 的性能问题。

### 事件代理

小程序视图触发事件后，会将 event 对象通知到 Page 实例，那么我们只需要将视图层中所有的事件都代理到 page.proxy 这个方法中，然后再靠这个方法从 Vue 的实例树上找到对应的 vm 和 handler 做事件处理。为了实现这一目的，在构建阶段对模版进行编译时，除了要将事件监听方法转换为 `proxy` 以外，还需要通过 `data-` 在元素上标记对应的组件 `compid` 和节点 `nodeid`。

```html
<!-- 编译前的 Vue 模版 .vue -->
<div @click="onClick"></div>

<!-- 编译后的小程序模版 .wxml -->
<view bindtap="proxy" data-compid="0" data-nodeid="0"></view>
```

事件触发时，proxy 方法会从 event 对象上获取对应的 id 信息和事件类型，进而从 Vue 的根 vm 开始查找，最终在 vnode 上找到对应的 handler 并执行事件处理，完成小程序事件到 Vue 实例的事件代理。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/ef68aa3e-6968-4129-b6ad-8ac79bd632ec_2099_1069.jpg" width="500px">
</div>

## 现在与未来

目前，megalo 已经逐步在考拉的小程序应用开发中投入使用，但 megalo 的数据映射方案早已通过 mpregular 在考拉的大量小程序应用中得到了验证。现在，megalo 支持 typescript 开发，支持支付宝小程序。

百度智能小程序的支持也在计划之内，同时，我们还计划开发一个兼容个平台的 UI 组件库、API 库，尝试将跨 H5 和各小程序平台的应用开发之间的差异最小化，提升开发效率。

## github

- [megalo](https://github.com/kaola-fed/megalo)
- [megalo-demo 示例](https://github.com/kaola-fed/megalo-demo)
- [megalo-aot 构建工具](https://github.com/kaola-fed/megalo-aot)
- [megalo-examples 工程样例](https://github.com/kaola-fed/megalo-examples)

## 参考

- [mpvue](https://github.com/Meituan-Dianping/mpvue)
- [小程序优化建议](https://developers.weixin.qq.com/miniprogram/dev/framework/performance/tips.html)