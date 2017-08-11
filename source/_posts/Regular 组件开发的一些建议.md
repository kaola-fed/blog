title: Regular 组件开发的一些建议 
date: 2017-07-27 22:27:31
tags:
---
## 前言

近几年来，前端开发这个概念发生了明显的变化。多年以前，前端开发的重点可能是使用 HTML/CSS/JS 和一些兼容知识来完成页面的构建，动效的制作。

而现阶段的前端开发，工作的很重要的部分，就是使用 Vue/React 或其他 MVVM 框架来完成我们的组件，并组织我们的页面。

我们来进入现实的开发场景。你接手一个需求后，最先做的是什么？

如果是一个新项目，你可能需要思考一些架构上的东西，比如：是否启用单页架构、使用什么开发框架、是否需要状态管理工具、资源构建怎么处理。嗯，你会饶有乐趣，然后兴致满满地开工。

事实上，新的项目并不多，我们更多时候也都是在维护现有代码，可能有很多事情会让你你觉得无奈的。比如，你对现有架构的不满意，对当前的业务逻辑书写得这么复杂表示不理解。于是，你想要重构。当然，我也希望重构。但是，我们的生命是否允许你消耗这段时间来重构这段老的逻辑。答案，是否。

所以，更重要地就是，阻止更多的代码恶化，这似乎比你单枪匹马搞重构，有用的多。这也是本文的重点，如何在现有项目，新需求的场景下，书写更可维护的组件化代码。

## 关于 RegularJS 开发的一些槽点
RegularJS 是网易出品的一款 MVVM 框架，本身其设计思路，和学习成本，都算中上。但是，作者对于设计思路在文档中的体现太少，导致顺势而为的使用者不多。比如说，下面的一些用法：

### 1. 过度庞大的组件
#### 描述
一个组件，原本可以拆成若干个组件来单独处理 UI 和 行为，现在被放到了一个组件里，使得组件过大，导致维护性、扩展性降低。

#### 建议
组件拆分有很多原则可以参考：
1. 六大设计原则的 「单一职责原则」，尽可能保证每个组件的职责单一性；
2. 当前比较盛行的 「容器组件」与「UI 组件」的方案，「容器组件」用于控制用户状态，并做一些数据更新操作，不直接表现UI；「UI组件」则主要负责 UI 展现，不做具体的逻辑处理，需要更新数据，尽可能使用事件，交给容器组件处理；

### 2. 组件原型上存放一些原本可以抽出的方法
#### 描述
组件原型链上放置了过多不属于 `this` 的处理逻辑，导致组件的可维护性与可理解性降低。

```js
const ComponentA = Regular.extend({
    getList() {
        this.$request({
            url,
            success({
                code, body
            }) {
                if (code === 200) {
                    this.getListCallBack();
                }
            }
        })
    },
    getListCallBack({list}) {
        this.filterList(list);
    },
    filterList(list) {
        this.data.list = list.map(({title, content}) => {
            return {
                title, content
            };
        });
    }
});
```
#### 建议
纯函数对于可测试性和可预期性都有较大的优势。

所以，不如把一些无关 `this` 的方法抽出来吧，把他们作为一些纯函数在你的组件生命周期中调用。

比如，把发 Ajax 请求的方法抽成 service，统一管理，把一些对象处理函数独立成 filter，总之 this 上尽可能的干净，对后期的维护是很有帮助的。


```js
const filterList(list) {
    return list.map(({title, content}) => {
        return {
            title, content
        };
    });
}

const ComponentA = Regular.extend({
    computed: {
        title(){
            this.getTitles();
        }
    },
    onButtonClick() {
        service.getList({

        }).then(() => {
            this.setList(list);
        })
    },
    setList({list}) {
        this.data.list = filterList(list);
    },
    getTitles() {
        return this.data.list.map(item => item.title);
    }
});
```

> 一般三类方法是必须的 1. 事件 handler; 2. compute get 方法; 3. data 的操作方法

1. 事件 handler 可以让你的函数更贴近于事件编程方式，更便于理解；
2. 使用 set 方法去设置 data 的数据，可以让你的 data 修改更可控；
3. 将 computed 抽出成方法，可以在 js 中复用 computed 的逻辑；

原则就是，能挪出去的就挪出去作为纯函数。

### 3. 滥用双向绑定
#### 描述
双向绑定，是指数据更新与UI行为的绑定操作。经常会出现一个对象依次传入多层组件的情况，这时候最省力的数据更新方式，可能就是直接在子组件内修改这个对象。

Index 组件
```
<ComponentA obj={obj}></ComponentA>
```

ComponentA 组件
```
<ComponentB obj={obj}></ComponentB>
```

ComponentB 组件
```
<ComponentC obj={obj}></ComponentC>
```

ComponentC 组件
```
<input r-model={obj.a}/>
```

我们看到如上这种情况，看似很巧妙的结合 RegularJS 的双向绑定，做了数据更新的操作，也能让最外层组件可以使用最新的 obj 对象做一些事情。但是，对于最外层组件而言， obj 的变化是不可预期的。换言之，开发者很难通过直接查看最外层组件，感知到 obj 对象可能发生的改变。这也是，双向绑定带来的弊端。

#### 解决

双向绑定的使用，非常方便，在深层嵌套的组件数据传递上，尤为明显。但它会使得数据的改变源头难以追踪，代码接手时也容易留坑，不好理解。

那么，我们应该控制住自己，不要过分依赖这种能力。换句话说，如果你肯定你的项目是不需要被维护的，那么你去使用吧（逃）

一般场景下，单向数据流的启用方式，是通过事件。

ComponentA

```html
<ComponentB
    obj={obj}
    on-btnClick={}
></ComponentB>
```

```js
Regular.extend({
    config() {
        extend(this.data, {
            obj: {
                clicked: false
            }
        })
    },
    onComponentBBtnClick() {
        this.setClicked();
    }
    setClicked() {
        this.data.obj.clicked = true;
    }
})
```
---- 
ComponentB
```html
<button disable={clicked} on-click={this.$emit('btnClick')}>点我！</button>
```

那么，如何解决深层嵌套组件的数据更新问题呢，就是使用状态管理工具，后面会提到。

### 4. 习惯把所有数据都放置到 `this.data` 上
#### 描述
习惯把所有数据都挂载到 `this.data` 上。

如果我们正视 data 的职责，data 只负责放置 UI 渲染相关数据。那么，我们平时开发中，很多操作是存在争议的：
1. 请求获取下来的数据，直接放置到 this.data 上的方式，后端在提供在前端的数据中，难免会存在一些冗余的字段，会直接赋值到 data 上，最直接的影响是导致脏值检查耗时增加，这些字段的存在会让你的 data 显得凌乱不清晰；
2. 大家可能会将一些逻辑计算的数据也放置到 data 上，但是这些数据对于模板渲染也是无关的。

#### 解决
这个问题的关键，是正确地认知 `this.data`。对于 MVVM 框架而言，最大的亮点莫过于视图与数据的绑定（单向 or 双向）， `this.data` 他真正的职责就是作为视图层渲染时提供必要的数据。

那么，你可能就能理解一些类似的话了：
「React只是一个优秀的视图层框架，他将复杂的状态管理留给了你」

所以，原则上不该把所有数据都放置到 `data` 上。其次，可以根据 1 中提到的方案，使用 「容器组件」和「UI组件」的分类，来区分我们的组件状态。
* 「容器组件」可以放置一些逻辑处理相关的字段，用于与后端交互和分发组件。
* 「UI 组件」的 `data` 应该避免赋值一些视图无关的状态。

当然，必要时，推荐引入状态管理工具来管理你的状态。

### 5. 组件传入的参数结构不明了
#### 描述
Index
```html
<ComponentA obj={obj}></ComponentA>
```

组件 ComponentA 的传入参数是 obj，但是 obj 对于调用者来说，是一个黑盒。直接阅读该组件的例子或现有代码，无法直接地了解到该组件的使用姿势。

#### 解决
所以，尽可能地直接明了地展现组件所需的属性，同时需要遵守遵守程序设计的「迪米特法则（最少知道原则）」，让所需的属性尽可能的少。

```html
<ComponentA 
    name={name}
    age={age}
></ComponentA>
```

### 6. 较长的表达式
#### 描述
Index
```html
{#list coupons as coupon}
    使用条件：{coupon.couponType === 1 ? `满¥ ${coupon.threshold} 使用` : '无条件使用'}
{/list}
```

#### 解决
模板中不该存在过多的计算逻辑，很重要一点是，我们一般无法对模板进行 debug，所以将逻辑抽出安放到 filter 或 computed 中，是个不错的方式。

```js
Regular.extend({
    template: `
    {#list coupons as coupon}
        使用条件：{coupon|getCouponCondition}
    {/list}
    `,
}).filter({
    getCouponCondition(coupon) {
        return coupon.couponType === 1 ? `满¥ ${coupon.threshold} 使用` : '无条件使用';
    }
})
```


## 总结
篇幅原因，简单列举几条。

书写可维护的 Regular 组件，需要我们有关于设计基本原则的一些理解，并在实际的 MVVM 框架下展开。

有接触过 Redux 的同学，可能会发现，上述的一些解决方案，有很多是借鉴自 Redux 的，如 单向数据流 、「容器组件」与「UI 组件」。那么如何使用状态管理来更为优雅地解决这些问题，本人另一篇文章 [Redux基础与实践]() 正在飞奔而来！！！