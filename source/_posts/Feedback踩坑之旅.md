---
title: Feedback踩坑之旅
date: 2017-06-07
---
目录

#### <a href="#section1">1. 重复的跨域头</a>

#### <a href="#section2">2. 打包后代码报错</a>

#### <a href="#section3">3. 获取客户端IP</a>

<!-- more -->

----------

### <p id="section1">1. 重复的跨域头</p>

之前的feedback方案是, 在宿主系统中嵌入一个iframe, 用户通过iframe向feedback提交信息. 这样导致feedback无法获取到宿主系统中当前页面的信息.

该来的总要来, 第二次迭代中, 问题暴露了. 在第二次迭代中, 要求打开反馈窗口后, 要在窗口中将当前页面的可见区域绘制出来, 并允许用户在绘制出的图像上进行标注.

这个需求的实现中使用了一个第三方的库, 将dom传到库提供的api里, 库把dom绘制到一个canvas里.跨域的话, iframe是无法拿到parent frame的dom内容的.所以提交信息的方式改为在宿主中打开普通的模态弹窗, 然后将保存的文本信息以及图像信息跨域提交到feedback系统中.

跨域提交数据, 需要在接收请求的server中对跨域做设置, 具体到node, 加上这几句:

```
res.header('Access-Control-Allow-Origin', '*');
res.header('Access-Control-Allow-Methods', 'GET,PUT,POST,DELETE,PATCH,OPTIONS');
res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization, Content-Length, X-Requested-With');
```
上面几句的意思显而易见, 分别是设置允许跨域请求来源的域名, 请求的方法, 请求的其他头信息.

理论上来说, 这样设置之后, feedback就会接收到所有域名来的跨域请求了, 但是在本地跑起来之后发现, 跨域请求还是会报错, 如下图:

![图1](https://raw.githubusercontent.com/yubaoquan/yubaoquan.github.io/master/images/feedback_1.png)

检查请求报文, 发现上面三句中涉及到的请求头都发了两份, 而且除了第一句中的`Access-Control-Allow-Origin`以外, 另外两个重复的头所对应的值并不相同, 如下图:

![图2](https://raw.githubusercontent.com/yubaoquan/yubaoquan.github.io/master/images/feedback_2.png)

由于请求头中存在两个`Access-Control-Allow-Origin:*`, 浏览器认为`Access-Control-Allow-Origin`的值是`*, *`, 所以跨域失败.

反复检查node代码后, 并没有找到其他地方设置了这个头, google了半天, 也没有找到有效的答案. 甚至尝试在node处理请求之前检查是否有重复, 都不能奏效. 所以当时的权宜之计是把第一句注释掉. 至于后两句的重复, 则不影响跨域请求.

但是发布到测试环境后, `Access-Control-Allow-Origin`头又不见了, 导致只能在开发过程中把第一句注释, 发布到线上的时候把第一句打开.

这样过了一段时间, 有一次开发feedback新功能的过程中, 偶然把跨域请求的域名改成了直接请求ip, 发现请求头重复的问题居然不见了, 换回域名请求后, 问题又重现出来. ip请求和域名请求, 中间有什么区别? ---- nginx.

是不是nginx对请求做了手脚? 打开nginx的配置文件, 果然发现如下三行:

```
add_header Access-Control-Allow-Origin *;
add_header Access-Control-Allow-Headers X-Requested-With;
add_header Access-Control-Allow-Methods GET,POST,OPTIONS;
```

这正好和浏览器中的报文吻合.

把这三行配置去掉, 重启nginx, 问题解决了.


### <p id="section2">2. 打包后代码报错</p>

feedback中的代码体积越来越大, 刚开始开发的时候, 只是把每个页面的文件打到一个js中, 后续开发中, 觉得这样处理显得很不美观. 于是把第三方库作为vonder单独拎出来, 这样打出来的包看起来结构清晰一点.

但是这又导致一个问题, 由于干掉了iframe, 所以要在其他系统中生成弹窗, 这样弹窗中的业务逻辑也要加到原先对外导出的js中. 而弹窗代码中, 也依赖了第三方库, 如`vue`, `vue-resource`等. 现在把第三方库拎出来, 难道要宿主系统再引入两个js吗? 这样显然不行.

目前想到的做法是, 把feedback系统内部使用的js用一个webpack打包, 对外暴露的js用另外一个webpack打包. 内部打包时把第三方库拎出来, 外部打包时把全部逻辑打到一起.

实践证明这样做是可行的, 但是在开发过程中, 大包后经常出现这样一个报错:
![图3](https://raw.githubusercontent.com/yubaoquan/yubaoquan.github.io/master/images/feedback_3.png)

重新执行一次打包, 报错就没了, 所以以为是webpack的bug或者是浏览器的缓存什么的, 并没多想.

但是放到测试环境之后, 发现这个bug100%发生, 重新构建部署也没用. 同样的代码, 同样的打包脚本, 为什么本地就能跑起来, 测试环境就跪了呢? 想了半天, 想不出名堂, 找到君羽帮忙, 把测试环境中打包后的文件和本地文件一个一个对比, 发现测试环境的第三方库文件比本地打包出来的文件多出来一大块代码, 而业务逻辑代码则没有区别.

又想了想, 想起来了, 原来, 两个打包脚本是先后执行的, 在两个打包脚本中, 第三方库对应的输出文件名设置成了一样的. 这导致后执行的打包脚本打出来的包, 覆盖了先执行的脚本打出来的包, 也就是不使用code split模式打出来的包把使用code split模式打出的同名文件覆盖掉了.

想着是这样, 那么真实的情况如何呢?

为了验证, 我把两个脚本中同名的输出文件名改成不同名, 再次到测试环境构建发布, 刷新页面, 报错没了, 页面正常了. 果然是这样.

### <p id="section3">3. 获取客户端IP</p>

业务中, 需要获取发起请求的客户端ip. 项目部署到测试环境后, 发现获取到的总是nginx的ip, 不是真实的客户端ip.

google了一下, 找到一个解决方案: 在nginx配置中, 加入如下一条配置:

```
proxy_set_header X-Real-IP $remote_addr;
```

这样, 当用户发起请求时, nginx会在请求头中添加一个`X-Real-IP`, 值就是客户端的真实ip.


by [Jerry Yu](http://yubaoquan.github.io)
