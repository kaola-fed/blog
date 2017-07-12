---
title: js事件捕获、冒泡一些容易忽略的方面
date: 2017-07-09 15:29:03
---

最近网络问题的反馈问题有点多，国内网络的国情，大家都清楚，只能靠强化自身努力来解决了；为了了解我们资源加载的实际情况，最近在考虑统计CDN资源加载情况，以及做一些页面资源全局检查，比如：

- js、css资源的`load`，`error`不支持冒泡，如何支持全局捕捉？
- 如何在页面图片`load`后，做一个统一处理？

<!-- more -->

发现一些自己忽略掉的好玩的东西：之前一直以为捕获和冒泡只是方向上的差异，其他没有什么区别，但发现他们还是有点区别的。

- 有些事件，比如`load`虽然不冒泡，但是可以在**document**层捕获；
- `error`事件会在**window**上的捕获阶段触发（js异常可以在冒泡或者捕获阶段触发，但资源加载错误比如js、css只能在捕获阶段触发），而不会在document上触发；

有了这些，我们就可以在**window**上**捕获**阶段订阅`error`事件，监听资源加载错误情况； 在**document**上的**捕获**阶段监听`load`事件，获取资源加载成功回调。从而通过成功+失败的信息，了解资源的加载错误、成功情况，然后回传到我们自己的服务器，用于数据统计。

记录代码示例：

```javascript
var __et = [], //error list
    __st = [], //load successfull list
    __cting = true;  //record switch
window.addEventListener('error', function(e) {
    if (!__cting) {
        return; }
    if (!(e instanceof ErrorEvent)) {
        __et.push(e.target);
    }
}, true);
document.addEventListener('load', function(e) {
    if (!__cting) {
        return; }
    __st.push(e.target);
}, true);
```