<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

#  用 RegularJS 开发小程序  —— mpregular 解析

Mpregular 是基于 RegularJS（简称 Regular） 的小程序开发框架。开发者可以将直接用 RegularJS 开发小程序，或者将现有的 RegularJS 应用通过较少修改移植到小程序上。Mpregular 为 RegularJS 开发者提供了一套跨 h5 和小程序的前端应用解决方案，让开发者能在不同平台有一致的开发体验和开发效率。

## 0 序

以下是使用 mpregular 前后的效果对比图：

<div style="display: flex; justify-content: space-around;">
  <figure>
    <p>旧版（原生小程序）</p>
    <img alt="1424614_old" src="https://haitao.nos.netease.com/86ff93a5-b8be-4305-9399-928901bc64fc.gif" width="200px">
  </figure>

  <figure>
    <p>新版（mpregular）</p>
    <img alt="1424614_new" src="https://haitao.nos.netease.com/7ab537a9-bfae-49ad-a790-9dcc8673750a.gif" width="200px">
  </figure>
</div>

## 1 为何而生

### 1.1 原生小程序开发

小程序本身提供的特性相对简单，在开发复杂应用的时候，用原生小程序进行开发就会显得比较吃力。为了更好支持复杂应用，小程序也推出自定义组件、wxs 等新特性，但这些新特性无形中又会给开发者带来一定的学习成本。另外，小程序的开发规范和通常的 web 应用的开发规范有着较大差异，如果需要同时在两端上开发同样功能的应用，则要求投入双倍的人力，无疑大大增加了开发和维护的成本。

### 1.2 考拉前端业务现状

目前考拉的 wap 前台页面大部分都是采用 RegularJS 开发的，包括 wap 首页、详情页，因此考拉的前端们都拥有丰富的 RegularJS 开发经验，RegularJS 可谓是我们最熟悉的前端开发框架之一。相比之下，熟悉小程序的开发就比较少了。微信是一个庞大的流量入口，最近小程序又掀起了一波热潮，伴随而来的就是小程序相关业务的增加。我们不仅需要把现有前台页面迁移到小程序，还需要开发和维护跨小程序和 wap 两端的业务。因此，我们迫切需要一个能够支撑当前业务的解决方案，保证我们的开发效率，降低开发和维护成本。

### 1.3 业界的解决方案

业界关于小程序也有许多解决方案。

小程序官方很早就推出了一个组件化解决方案 —— Wepy，它有自己的一套语法规范，构建时将 Wepy 代码编译转换为小程序代码。它强依赖小程序自身的特性，因此受小程序自身特性所限，开发规范与考拉当前的前端技术栈差异较大，并不适用。

京东的凹凸实验室推出了新的跨端开发框架 Taro，它是一个 React-like 的开发框架，有完善的配套设施，支持大部分 React 特性。但 Taro 是在 mpregular 开发完成后才出来，而且不符合我们当前的技术栈。

美团今年早些时候推出 mpvue，一个基于 Vue 实现的小程序开发框架，Vue 开发者看到这个框架以后欢天喜地，Github 上 star 数迅速攀升。对此，我们也做了一些调研，它不仅支持了大部分 Vue 的特性，而且有完善的文档教程、配套设施，可以说是一个非常完善的解决方案。但我们当前存在的大量 RegularJS 页面到小程序的迁移需求，在这一场景面前，mpvue 显得无能为力。

纵观业界的解决方案，都很难满足我们当前的需求。我们受到了 mpvue 的启发，并借鉴它的基本设计思想（包括名称...），决定对 RegularJS 进行了改造，开发 mpregular 这一个基于 RegularJS 的小程序开发框架。

## 2 框架特性

既然是基于 RegularJS 实现的框架，语法规范必然是与 RegularJS 基本一致。在开发的的时候，基本上只要遵循 RegularJS 的开发规范进行即可，大大降低了 RegularJS 开发者的学习成本。

### 2.1 生命周期

小程序有 App 和 Page 两个重要的概念，但通常业务代码是写在 Page 里的，这里就以小程序页面为例。开发者在开发小程序页面的时候，基本只需要了解 Regular 实例的生命周期。小程序 Page 的 `onLoad`、`onReady` 已经通过 mpregular 与 Regular 的实例生命周期绑定在一起了。页面 url 的 querystring 也可以通过 `this.$mp.options` 获取。 `onShow`、`onPageScroll` 等小程序特有的生命周期钩子都同样绑定到 Regular 实例上。

```html
<template>
  <div>
    <ComponentA />
  </div>
</template>
<script>
  import ComponentA from './component-a.rgl';
  export default {
    mpType: 'page',
    config() {
      // this.$mp.options 与 onLoad 中的 options 相同
      // 用于获取 options.query
      console.log('config', this.$mp.options);
    },
    init() {
      console.log('init');
    },
    onShow() {
      console.log('onShow');
    }
  }
</script>
```

### 2.2 语法和特性

mpregular 支持 RegularJS 的语法和大部分特性。例如：

```html
<template>
  <div>
    <input r-model="{ input }" on-confirm="{ this.onConfirm($event) }">
    <div>
      {#list toDoList as item}
        <div class="item { item.checked ? 'z-checked' : '' }">
          <span>{ item.name }</span>
          <span>{ item.date | dateFormat: 'yyyy-MM-dd' }</span>
        </div>
      {/list}
    </div>
  </div>
</template>
```

上述模版中的语法可以直接在小程序上执行。除此以外，mpregular 还支持 r-html、r-hide、{#include this.$body }、filter 等特性。这些特性在现有业务代码中被大量使用，因此在迁移现有代码时，几乎可以原封不动地拷贝过来（除非原有代码中包含大量 DOM 操作...）。

mpreguar 支持的特性:

- RegularJS 基本语法，包括 {#list}，{#if}, {#include this.$body }
- filter
- r-model
- r-hide
- r-html
- r-class(即将支持)
- r-style(即将支持)

相比于原生小程序和业界其他框架而言，mpregular 给 RegularJS 开发者提供了他们更熟悉的开发模式，支持更多的特性，对模版的处理能力进一步增强，更适应于我们当前复杂应用的业务场景。

特性 | 小程序 | mpvue | mpregular |
:--- |:---  |:---|:---
语法规范 | 原生小程序 | Vue 规范 | RegularJS 规范
组件化 | 小程序组件化规范 | Vue 组件 | RegularJS 组件
computed 计算属性 | ❌ | ⭕ | ⭕️
model 双向绑定 | ❌ | ⭕ | ⭕️
slot 插槽 | ❌ | ⭕ | ⭕️
filter 过滤器 | ❌ | ❌ | ⭕️
html 富文本 | ❌ | ❌ | ⭕️
复杂表达式插值 | ❌ | ❌ | ⭕️
小程序分包 | ⭕ | ❌ | ⭕️

## 3 基本原理

小程序在结构上主要有 Service(JavascriptCore) 和 View(WebView) 两部分组成，分别运行在独立的环境上，之间不具备共享数据通道，二者的通信方式是将数据封装在 js 脚本后传递。Page 实例就在 Service 中，通过 setData 方法将数据传递到 View。View 则通过事件绑定将视图层触发的事件传递给 Service。

<div style="text-align:center;">
  <img alt="小程序" src="https://haitao.nos.netease.com/a1f4d2fc-2725-4e3e-9ae4-58aff66aa2b3.jpg" width="300px">
</div>

Regular 是基于 Living Template 实现的，它使用一个内建 DSL 将模版字符串解析成 AST，然后在编译阶段结合数据模型将 AST 进行递归遍历，并在这个遍历过程中生成 DOM 节点，同时完成插值、指令等的绑定，实现 DOM 与数据的链接。

<div style="text-align:center;">
  <img alt="RegularJS" src="https://haitao.nos.netease.com/203cc7e4-df08-4abd-8ee8-67085319248f.jpg" width="500px">
</div>

Mpregular 要做的就是将 Regular 的视图层从 DOM 替换成小程序的 View。在小程序中不能直接操作 View 中的 DOM 节点，而是需要通过小程序的 Service 层 `setData` 方法去更新 View 的数据。

构建时，mpregular 会将 Regular 的模版字符串预先编译成小程序的模版 .wxml，通过小程序的 Service 与小程序的 View 建立联系，实现数据更新和事件监听。由于小程序中无法使用 `eval` 和 `new Function` 等操作，所以 mpregular 会在构建阶段预先生成 AST ，运行时从源码中读取 AST。在执行 `this.$update` 时把更新数据通知 Service，调用 `setData` 完成视图更新。View 触发的事件会被代理到 Service 的 `proxyEvent` 方法，这个方法会在 RegularVM 中找到对应的事件处理函数并执行。

<div style="text-align:center;">
  <img alt="mpregular" src="https://haitao.nos.netease.com/415c386d-265c-46bd-b65e-ee5b5d57f75c.jpg" width="500px">
</div>

Mpregular 要做的，就是在 Regular 实例和小程序 Service 之间建立联系，完成生命周期绑定、数据更新、事件代理等工作。

### 3.1 生命周期

小程序中通过调用 Page 方法注册页面，而页面加载时创建的页面实例 `PageVM` 就是 mpregular 与小程序建立连接的通道。

Mpregular 在定义页面入口的 Regular 组件时去调用 Page 方法注册页面，并将 Page 的生命周期钩子与 Regular 的生命周期进行绑定。

```javascript
page.init = function(config) {
  Page({
    onLoad(options) {
      this.rootVM = initRootVM(this, config);
      callHook(this.rootVM,'onLoad');
    },
    onReady() {
      callHook(this.rootVM,'onReady', options);
      initDataToMP(this.rootVM);
    }
  })
}
```

在 Page 实例化（页面加载）时，会触发 `onLoad` 钩子，此时会对这个页面对应的 Regular 入口组件进行实例化，并将 `PageVM` 和 `RegularVM` 绑定在一起。由于每个页面只有一个 `PageVM`，所以 `PageVM` 会与 `RegularVM.$root` 进行绑定，之后 Regular 的逻辑会利用 `RegularVM.$root` 所绑定的 `PageVM` 与小程序进行通信。当页面初次渲染完成后，会触发 `onReady` 钩子，对应于 Regular 的 `init`。当页面的其他钩子函数触发时，如 `onShow`、`onHide`，`PageVM` 会通过 `callHook` 方法调用 `RegularVM` 上定义的同名方法。在页面退出销毁时，`onUnload` 中则会触发 `RegularVM` 的 `destroy` 方法，将页面绑定对应的 Regular 实例销毁。

<div style="text-align:center;">
  <img alt="lifecycle" src="https://haitao.nos.netease.com/50709dc9-67d4-4767-9746-fc7d74bd514e.jpg" width="500px">
</div>

### 3.2 模版转换

由于 Regular 的模版语法与小程序模版语法不一样，所以在构建阶段，mpregular-loader 会把 Regular 的模版字符串转换成小程序的 .wxml，不仅会对标签进行转换，还会对模版的语法、子组件模版进行处理。所定义的每个 Regular 组件，包括入口组件，都会被转换成一个个模版片段，存放到对应的 .wxml 文件中，并用 `<template name="${componentName}">` 包裹起来，用组件名命名。

```html
<!-- app.rgl -->
<template>
  <CustomComponent></CustomComponent>
  <div>
    <span>{ title }</span>
    <input r-model="{ input }" on-confirm="{ this.onConfirm($event) }">
  </div>
</template>
```

上面这段 Regular 的模版就会被转换会符合小程序模版语法的模版文件，如：标签 `<div>`、`<span>` 会被转换为 `<view>`、`<label>`，事件监听的语法则会进行转换且把所以事件统一代理到 `PageVm` 上的 `evenProxy` 方法上。对于外部组件，则会通过 `<import>` 把组件的模版片段引入。由于所有模版片段都在同一个 Page 的作用域下，即从 `PageVm.data` 上取数据，因此需要一个规则将Regular 各个组件实例的数据映射到对应的模版片段中。

```html
<!-- app.wxml -->
<import src="./components/custom-component.wxml">
<template name="app">
  <template is="./CustomComonent" data={{ customComonentData }}>
  <view>
    <label>{{ title }}</label>
    <input bindinput="proxyEvent" bindconfirm="evenProxy" value="{{ input }}">
  </view>
</template>
```

### 3.3 数据和视图的绑定

小程序对于 mpregular 而言只起到了视图层的作用，小程序的模版全都会汇集通过 `<import>` 标签汇集到页面的入口 .wxml 中，这些被引入的模版的所有数据都是从 `PageVM.data` 上获取的，意味着需要一定的映射规则，才能将 RegularVM 树上各个子组件的数据绑定到小程序模版对应的节点上。对此，mpregular 借鉴了 mpvue 的数据结构设计，利用子组件在 VM 树上的路径生成唯一的 id，将子组件上的数据映射到对应的 View 节点上。

用以下这段简单的代码进行说明。`<Page>` 是整个页面的入口模版，包含三个组件，分别是 `<Header>`、`<Counter>`、`<Panel>`。

```html
<!-- Counter.rgl -->
<template>
  <div>
    <Panel></Panel>
  </div>
</template>

<!-- page.rgl -->
<template>
  <Header></Header>
  <Counter></Counter>
</template>
```

以 `<Page>` 作为根节点，结构如下图所示，是一个三层的树结构。按照组件声明的顺序，每一层级的组件序号从 0 开始递增。每个组件在树中的 id 则根据它在树中的路径生成，如果 `<Header>` 则为 `0,0`，`<Panel>` 的 id 为 `0,1,0`，利用 `,` 进行分隔，根据 id 可以反推出该组件实例在树中的位置。 

<div style="text-align:center;">
  <img alt="vm_tree" src="https://haitao.nos.netease.com/9911a7e3-a2c2-4b6a-a8c3-f74889862df3.jpg" width="200px">
</div>

根据组件的 id，就可以把每个组件要更新到视图的数据收集起来，并将收集的数据保存到小程序 `PageVM.data.$root` 上。

```javasript
{
  $root: {
    '0': { ... }      // Page
    '0,0': { ... }    // Header
    '0,1': { ... }    // Counter
    '0,1,0': { ... }  // Panel
  }
}
```

利用 id 就可以把各个各个组件的数据映射到模版对应的节点上，转换出来的模版如下所示（为了方便理解，这里时简化的实例代码，并不是实际转换结果）。

```html
<!-- counter.wxml -->
<template>
  <view>
    <template is="./Panel" data={{ ...$root[ '0,1,0' ] }}>
  </view>
</template>

<!-- page.wxml -->
<template>
  <template is="./Header" data={{ ...$root[ '0,0' ] }}>
  <template is="./Counter" data={{ ...$root[ '0,1' ] }}>
    <Panel></Panel>
</template>
```

而 `Page.data.$root` 上的挂载的各个组件实例的数据，与模版的映射关系如下图所示。

<!-- ![data_struct](https://haitao.nos.netease.com/a93c00d0-6b2c-4bf4-bfaf-c72d95b37e16.jpg) -->

<div style="text-align:center;">
  <img alt="data_mapping" src="https://haitao.nos.netease.com/dc7e8390-fb01-43e6-aeed-4b1579a3659a.jpg" width="500px">
</div>

有了这个映射关系之后，通过 `PageVM.setData` 更新 `PageVM.data.$root` 上的数据，就完成了数据的更新。

### 3.4 事件代理

如上所述，所有模版片段的作用域都与该页面的 `PageVM` 一致，事件只能由 `PageVM` 进行代理转发。构建时，mpregular-loader 会为每个包含事件监听的元素添加上 eventId 和 compId， 用于标记该元素和所属组件（如下所示）。在注册页面的时候，mpregular 会在 Page 上挂载 `proxyEvent` 方法，所有事件都将代理到这个方法。

```html
<!-- RegularJS 模版 -->
<div on-click="{ this.onClick($event) }"></div>

<!-- 转换后的小程序 .wxml -->
<div bindtap="proxyEvent" event-id="0" comp-id="0"></div>
```

Mpregular 在为各个事件注册处理方的时候，为每个组件创建一个 `eventHandlers` 对象，根据事件类型和 `eventId` 记录各个事件处理函数。

```javascript
{
  componentId: '0',
  // ...
  eventHandlers: {
    '0': {
      'tap': function() handler{}
    }
  }
}
```

当事件触发时，`PageVM.proxyEvent` 方法会根据 `compId` 找到对应的 `RegularVM`，再根据事件类型和 `eventId` 找到对应的 `handler`，最后执行对应的处理函数，完成事件代理。

<div style="text-align:center;">
  <img alt="proxyEvent" src="https://haitao.nos.netease.com/951e13c4-934c-4f48-87be-fd0707c13d46.jpg" width="400px">
</div>

### 3.5 性能优化

上面所讲述的原理，就是让 RegularJS 在小程序中运行的关键，但是仅仅运行起来还是不够的，在实际业务场景下，还需要进一步优化才能更好地支撑业务，尤其是对于数据更新的优化。小程序官方文档中特别强调 `setData` 在传递大数据时会大量占用 WebView JS 线程。同时我们发现，`PageVM` 上挂载的数据过大，也会严重影响 `setData` 的性能。为此 mpregular 做了特别的优化，核心方向有两个：

1. 降低频率
2. 减少数据量

#### 3.5.1 缓存数据，定期更新

降低频率的方法比较简单，mpregular 会在调用 `this.$update` 时，先把需要更新的数据会缓存起来，每间隔 50ms 从缓存中取出数据进行批量更新，以减少避免频繁的 `setData` 操作。

#### 3.5.2 只更新 View 需要的数据

通常，在进行原生小程序的开发时，需要通过 `setData` 把数据更新到 `PageVM.data` 和 View  上，这也是唯一让 View 和 Service 线程保持数据一致的方式。但这样带来的一个问题，在调用 `setData` 时，开发者很少会去区分哪些数据真正是 View 需要的，从而使得有大量的视图无关数据被传递到 View，影响数据更新性能。

举一个例子，视图层需要从一个大对象上读取其中一个值，`largeData.info.countdown.time`。最简单直接的做法时直接将模版编译成下面这样，把 `largeData.info.countdown.time` 写到 .wxml 上，mpregular 在运行时把 `largeData` 更新到 View，由 View 去解析这个对象，取得所需的值。如果只是一次性传递也还好，但如果这个是一个毫秒级的倒计时模版，每次时间更新，就要重新把 `largeData` 传给 View，性能变得极为糟糕。当然，开发者可以通过把值提取到 `this.data.time` 就可以绕过这个问题，但这样会为开发者带来许多不便。

```html
<!-- RegularJS template -->
<span>{ largeData.info.countdown.time }</span>

<!-- 转换后的小程序 wxml -->
<label>{{ largeData.info.countdown.time }}</label>
```

为此，mpregular 做了深度优化，在构建时 mpregular-loader 会对视图层用到的插值表达式进行标记，将标识同步到 AST 上，把模版转换成如下面代码那样。mpregular 在运行时，会根据 AST 上的标志将执行插值表达式的执行结果填入对应的位置上，最后再更新到视图层。这样，数据的传递由一个大对象变成了一段字符串，大大提升数据更新性能。

```html
<!-- RegularJS template -->
<span>{ largeData.info.countdown.time }</span>

<!-- 转换后的小程序 wxml -->
<label>{{ __holders[0] }}</label>
```

有了这一机制，像 filter、r-html 等特性，都可得以实现。在 Regular 里面，包含 filter 的插值、r-html 指令都会被转换成插值表达式，用同样的方法根据插值表达式的标志将执行结果映射到对应模版节点上，就能够实现原生小程序不支持的各种特性，极大地强化了模版的能力。

此外，mpregular 对列表渲染也进行了优化。在对 `source` 进行遍历时，视图层是不需要获取 `source` 的实际内容的，mpregular 将 `source` 重新映射成一个具有同等长度的简单数组，如 `[0, 1, ...]`，再传递给视图层去遍历渲染，而所渲染的列表内容也会采用相同机制，将数据映射到列表中的对应位置。

```html
<!-- RegularJS template -->
{#list source as item}
  <span>{ item.name }</span>
{/list}

<!-- 转换后的小程序 wxml -->
<block wx:for="{{ __holders[ 0 ] }}" wx:for-item="item" wx:for-index="item_index">
  <label>{{ __holder[ 1 + '-' + item_index ]}}</label>
</block>
```

## 4 实践

Mpregular 初版完成以后，我们立马把它投入到生产当中。目前，考拉的小程序商品详情页已经用 mpregular 重构完成，页面性能有明显提升。

旧版商品详情页使用原生小程序进行开发，在处理多 sku 商品时，会存在性能问题。如果在处理下图中包含 140+ 个 sku 数据的的商品时，点击加入购物车按钮后，sku 选择弹层出来有明显延时，这正是因为在调用 `setData` 更新大量 sku 数据时引发性能问题。使用 mpregular 重构后，sku 选择弹层的弹出速度明显加快。

<div style="display: flex; justify-content: space-around;">
  <figure>
    <p>旧版（原生小程序）</p>
    <img alt="1424614_old" src="https://haitao.nos.netease.com/86ff93a5-b8be-4305-9399-928901bc64fc.gif" width="200px">
  </figure>

  <figure>
    <p>新版（mpregular）</p>
    <img alt="1424614_new" src="https://haitao.nos.netease.com/7ab537a9-bfae-49ad-a790-9dcc8673750a.gif" width="200px">
  </figure>
</div>

另外还有一个包含 220+ sku 数据的商品，新版详情页性能没有受到大量 sku 数据的影响，而旧版详情页因为单次 `setData` 数据量超出限制，使页面无法正常渲染（下方加车栏渲染失败）。

<div style="display: flex; justify-content: space-around;">
  <figure>
    <p>旧版（原生小程序）</p>
    <img alt="1488086_old" src="https://haitao.nos.netease.com/8751616e-92a5-4b26-ad60-c187552873ee.gif" width="200px">
  </figure>

  <figure>
    <p>新版（mpregular）</p>
    <img alt="1488086_new" src="https://haitao.nos.netease.com/f924fd6d-9921-4167-bff9-f5af99f11eeb.gif" width="200px">
  </figure>
</div>

除了商品详情页以外，小程序的售后、拉新等新老业务都陆续开始使用 mpregular 进行开发。

## 5 总结与展望

Mpregular 为当前考拉跨 wap 和小程序两端的老新业务开发和维护提供了有效的跨端解决方案，并能解决部分场景的性能问题。我们将长期维护 mpregular，继续完善文档和教程，增加单元测试保障代码质量，继续对性能、构建打包方式等进行优化，相关的配套设施也在将进一步完善。

Mpregular 验证了 RegularJS 在小程序相中运行的可行性，相信 RegularJS 也能与 weex 相结合，成为一个跨端开发框架，也希望 RegularJS 生态能够活跃起来。

## github

- [mpregular](https://github.com/kaola-fed/mpregular)

- [mpregular-loader](https://github.com/kaola-fed/mpregular-loader)

- [mpregular-example](https://github.com/kaola-fed/mpregular-example)

## 参考

- [mpvue](https://github.com/Meituan-Dianping/mpvue)
- [一个对前端模板技术的全面总结](http://www.html-js.com/article/Regularjs-Chinese-guidelines-for-a-comprehensive-summary-of-the-front-template-technology)
- [小程序优化建议](https://developers.weixin.qq.com/miniprogram/dev/framework/performance/tips.html)

 by 
[fengzilong](https://github.com/fengzilong)
& [elcarim](https://github.com/elcarim5efil)