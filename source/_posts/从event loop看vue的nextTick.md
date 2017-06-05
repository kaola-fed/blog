---
title: 从event loop看vue的nextTick
date: 2017-05-30
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
-->

关于js的event loop知识点，大家可以阅读之前的一个同学写的文章[JavaScript并发模型与Event Loop](https://github.com/kaola-fed/blog/issues/34)，这里就不着重介绍了，相信将他的文章看完，能有一个清晰的认识了。在此，再推荐一篇关于event loop的文章[Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/?utm_source=html5weekly&utm_medium=email)。

好了，在对js的event loop有一定了解后，我们就来进入今天的主题，关于vue的nextTick。

<!-- more -->

在vue中，数据监测都是通过Object.defineProperty来重写里面的set和get方法实现的，vue更新DOM是异步的，每当观察到数据变化时，vue就开始一个队列，将同一事件循环内所有的数据变化缓存起来，等到下一次event loop，将会把队列清空，进行dom更新，内部使用的microtask MutationObserver来实现的。

虽然数据驱动建议避免直接操作dom，但有时也不得不需要这样的操作，这时就该```Vue.nextTick(callback)```出场了，它接受一个回调函数，在dom更新完成后，这个回调函数就会被调用。不管是vue.nextTick还是vue.prototype.$nextTick都是直接用的nextTick这个闭包函数。

```
export const nextTick = (function () {
  const callbacks = []
  let pending = false
  let timerFunc

  function nextTickHandler () {
    pending = false
    const copies = callbacks.slice(0)
    callbacks.length = 0
    for (let i = 0; i < copies.length; i++) {
      copies[i]()
    }
  }
  
  //other code
})()
```

callbacks就是缓存的所有回调函数，nextTickHandler就是实际调用回调函数的地方。

```
if (typeof Promise !== 'undefined' && isNative(Promise)) {
    var p = Promise.resolve()
    var logError = err => { console.error(err) }
    timerFunc = () => {
        p.then(nextTickHandler).catch(logError)
        if (isIOS) setTimeout(noop)
    }
} else if (typeof MutationObserver !== 'undefined' && (
    isNative(MutationObserver) ||
    MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
    var counter = 1
    var observer = new MutationObserver(nextTickHandler)
    var textNode = document.createTextNode(String(counter))
    observer.observe(textNode, {
        characterData: true
    })
    timerFunc = () => {
        counter = (counter + 1) % 2
        textNode.data = String(counter)
    }
} else {
	timeFunc = () => {
		setTimeout(nextTickHandle, 0)
	}
}
```
为让这个回调函数延迟执行，vue优先用promise来实现，其次是html5的MutationObserver，然后是setTimeout。前两者属于microtask，后一个属于macrotask。下面来看最后一部分

```
return function queueNextTick(cb?: Function, ctx?: Object) {
    let _resolve
    callbacks.push(() => {
        if (cb) cb.call(ctx)
        if (_resolve) _resolve(ctx)
    })
    if (!pending) {
        pending = true
        timerFunc()
    }
    if (!cb && typeof Promise !== 'undefined') {
        return new Promise(resolve => {
            _resolve = resolve
        })
    }
}
```
这就是我们真正调用的nextTick函数，在一个event loop内它会将调用nextTick的cb回调函数都放入callbacks中，pending用于判断是否有队列正在执行回调，例如有可能在nextTick中还有一个nextTick，此时就应该属于下一个循环了。最后几行代码是promise化，可以将nextTick按照promise方式去书写（暂且用的较少）。

nextTick就这么多行代码，但从代码里面可以更加充分的去理解event loop机制。

### 参考资料
[vue](https://github.com/vuejs/vue)

[JavaScript并发模型与Event Loop](https://github.com/kaola-fed/blog/issues/34)

[Tasks microtasks queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/?utm_source=html5weekly&utm_medium=email)

[MutationObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)