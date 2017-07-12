---
title: fetch简述
date: 2017-06-27
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

在与服务端进行异步请求时，你是否还在写着地狱般的回调，是否还在写着繁琐的配置，是否还觉得调用非常的混乱，这就是我们一直使用了很长时间的XMLHttpRequest来实现的ajax技术，尽管各种框架的封装就是为了解决上述的问题，但有时需要阅读到深层的源码时，又将会是另外一番痛苦，同时ajax技术也不符合关注分离的原则，因此，fetch应运而生了，它的出现正式为了解决传统xhr存在的问题。

<!-- more -->

概念
----
**fetch是基于标准的Promise设计的**，使得语法上更加直观和简介，与前端ES6/ES7语法能更好的结合。既然fetch是新产生的技术，相信你心里肯定在想它的兼容性怎么样，是否能完美的运用到所需场景中去。那我们就先来看看它的兼容性以及不兼容的解决方法。
![fetch-polyfill](https://user-images.githubusercontent.com/5706155/27586474-ff0ed236-5b72-11e7-9586-df6dd72c11ec.png)

通过上图可以看到，很多低版本的浏览器，fetch还是如你所想的那样不被支持的，因此为了支持更多的浏览器，就需要实现fetch的polyfill了。思路其实很简单，就是判断浏览器是否支持原生的fetch，不支持的话，就仍然使用XMLHttpRequest的方式实现，同时结合Promise来进行封装。常见的polyfill就有：
> [es6-promise](https://github.com/stefanpenner/es6-promise)、[fetch-ie8](https://github.com/camsong/fetch-ie8)、[isomorphic-fetch](https://github.com/matthew-andrews/isomorphic-fetch)等等。

使用
----
fetch函数的参数本质就是一个[Request](https://developer.mozilla.org/en-US/docs/Web/API/Request)，函数本身返回一个promise，可以通过then方法接收一个[Response](https://developer.mozilla.org/en-US/docs/Web/API/Response)实例，因此基本用法如下

```
let request = new Request('/someurl', {
    method: 'GET', 
    mode: 'cors', 
    headers: new Headers({
        'Content-Type': 'application/json'
    })
});
fetch(request).then((response) => {
	//处理返回的数据
}).catch((err) => {
	//错误处理
})

//同样可以直接写成
fetch('someurl', {
    method: 'POST', 
    mode: 'cors', 
    body: JSON.stringify(jsonData);
    headers: new Headers({
        'Content-Type': 'application/json'
    })
}).then().catch()
```

---
关于Request的参数，这里挑几个重点说下

1. mode

    这个是设置跨域的参数，值为**same-origin**时，不允许跨域，它需要遵守同源策略。值为**cors**时，默认值，该模式支持跨域请求，顾名思义它是以CORS的形式跨域。值为**no-cors**时，该模式用于跨域请求但是服务器不带CORS响应头，也就是服务端不支持CORS，这也是fetch的特殊跨域请求方式，该模式允许浏览器发送本次跨域请求，但是不能访问响应返回的内容。
2. credentials

    fetch本身请求默认是不发送cookie的，需要带cookie就需要设置此参数。**omit**默认值，忽略传送cookie。**same-origin**表示cookie只允许同域传送，不允许跨域进行传送。**include**表示cookie都会被传送。
    >fetch默认对服务端通过Set-Cookie头设置的cookie也会忽略，若想选择接受来自服务端的cookie信息，同样必须要配置credentials选项

---
Response介绍

response本身继承自[Body](https://developer.mozilla.org/en-US/docs/Web/API/Body)，因此可以使用Body的方法，例如返回的是json对象，可以这样获取到返回结果

```
fetch(/someurl.json).then((response) => {
	if (response.status >= 200 && response.status < 300) {
		return response.json()
	}
}).then((data) => {

})

```
>注意: 当使用了response.json()、response.blob()等方法后，body.bodyUsed就变为true了，将不能再继续使用这些方法了，否则会抛出错误。Body还有一个clone方法，但需要在response读取之前调用。

特殊处理
----

* fetch对于网络响应400或500等状态并不会reject，相反它会被resolve，只有在网络本身发生错误时才会被reject，因此在我们使用的时候可以对其进行一层封装

```
const fetchApi = (url, cfg = {}) => {
    return new Promise((resolve, reject) => {
        fetch(url, cfg).then((response) => {
            if (response.ok || response.status >= 200 && response.status < 300) {
                resolve(response.json())
            } else {
                throw response
            }
        }).catch((err) => {
            reject(err)
        })
    })
}
```

* fetch本身不支持超时处理，因此我们可以通过自己进行一些polyfill。

```
function timeout(ms = 5 * 60 * 1000) {
    return new Promise((resolve, reject) => {
        setTimeout(function () {
            reject({ok: false, text: 'timeout', title: '服务器超时！请重试！'});
        }, ms);
    });
}

const fetchApi = (url, cfg = {}) => {
    return new Promise((resolve, reject) => {
        Promise.race([fetch(url, cfg), timeout()]).then((response) => {
            if (response.ok || response.status >= 200 && response.status < 300) {
                resolve(response.json())
            } else {
                throw response
            }
        }).catch((err) => {
            reject(err)
        })
    })
}
```
> fetch的timeout即使超时发生了，本次请求也不会被abort丢弃掉，它在后台仍然会发送到服务器端，只是本次请求的响应内容被丢弃而已

* fetch本身也是不支持取消的，因为promise本身是不能取消的，有提议希望未来出现一个可取消的Promise标准，但暂且我们也只能进行一个封装

```
const fetchApi = (url, cfg) => {
    retrun new Promise((resolve, reject) => {
        let abort = () => {
            reject({ok: false, text: 'abort', title: 'abort'});
        }

        let p = fetch(url, cfg).then((response) => {
            if (response.ok || response.status >= 200 && response.status < 300) {
                resolve(response.json())
            } else {
                throw response
            }
        }).catch((err) => {
            reject(err)
        })
        p.abort = abort
        return p
    })
}
```

## 参考资料
[传统 Ajax 已死，Fetch 永生](https://github.com/camsong/blog/issues/2)

[fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)

[es6 Promise](http://es6.ruanyifeng.com/#docs/promise)

[isomorphic-fetch](https://github.com/matthew-andrews/isomorphic-fetch)

[Request](https://developer.mozilla.org/en-US/docs/Web/API/Request)

[Response](https://developer.mozilla.org/en-US/docs/Web/API/Response)

[Body](https://developer.mozilla.org/en-US/docs/Web/API/Body)

[Headers](https://developer.mozilla.org/en-US/docs/Web/API/Headers)

