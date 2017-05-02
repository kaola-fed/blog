---
title: 意外发现regular v0.4.3的一个bug，它竟然这么干的
date: 2017-05-01
---

哈哈，标题很有吸引力吧。

言归正传，前几天在页面搭建工程下进行开发时，碰到了一个很困惑的问题，很容易复现，也很容易理解，就直接上代码吧：

<!-- more -->

```javascript

let Suggest = Regular.extend({
    template: '<div>标题：{schemeData.title}</div><div>Regular版本：{version}</div>',
    config(data){
        data.schemeData = Object.assign({}, {
            title: '默认标题'
        }, data.schemeData);
        // 这里很奇怪：强制修改标题，regular v0.4.3的表现和v0.5.2不一致
        data.schemeData.title = '正常标题';
        this.supr(data);
    }
});

let Page = Regular.extend({
    template: '<suggest schemeData={schemeData} version={version}></suggest>',
    config(data){
        data.version = '0.4.3';
        data.schemeData = {
            title: '不正常标题'
        };
        this.supr(data);
    }
}).component('suggest', Suggest);

let page = new Page({data: {}});
page.$inject('#app');

```
这段代码在regular v0.4.3下，输出的结果是：
```html
<div>标题：不正常标题</div>
<div>Regular版本：0.4.3</div>
```

> 请注意这里输出的标题是: "不正常标题"

可以看到，我在子组件的config方法中有强制覆盖父组件传进来的title：

```javascript
// 这里很奇怪：强制修改标题，regular v0.4.3的表现和v0.5.2不一致
data.schemeData.title = '正常标题';
```
但是很明显，没有起到我想要的效果。

0.4.3对应的demo可以在这里查看: https://codepen.io/lidong639/pen/wddQzM 。


同样的代码在0.5.2下却可以按照期望输出想要的结果：https://codepen.io/lidong639/pen/QvvVpB 。


起初，我以为是使用Object.assign方法引起的，因为Object.assign方法会返回一个新的对象的引用，这样就和父组件传进来的对象的引用断开了，并且，可以使用工程封装的extend方法进行验证：

```javascript
_.extend(data.schemeData, {
    title: '默认标题'
});
```
将Object.assign改成_.extend后，输出的结果符合预期：https://codepen.io/lidong639/pen/NjjEgb 。

但是，同样的代码在 v0.4.3 和 v0.5.2 下表现不一致，可以从侧面印证这是regular的一个bug，并且 v0.4.3 以上的版本都修复了这个问题。

暂且这样下结论吧，是否0.4.3的本意就是如此设计，还没来得及仔细分析，如果有知道内情的同学，请发表下您的看法吧！