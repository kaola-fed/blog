---
title: HTTP请求中的Form Data与Request Payload的区别
date: 2017-08-10
---


前端开发中经常会用到AJAX发送异步请求，对于POST类型的请求会附带请求数据。而常用的两种传参方式为：Form Data 和 Request Payload。

![121212](https://user-images.githubusercontent.com/5309877/29163614-9ac78ea4-7def-11e7-9067-77d73a5d841c.jpg)

![334343](https://user-images.githubusercontent.com/5309877/29163628-a3e89d48-7def-11e7-869a-1f818fd97149.jpg)

<!-- more -->

## GET请求
使用get请求时，参数会以key=value的形式拼接在请求的url后面。例如：

```
http://m.baidu.com/address/getlist.html?limit=50&offset=0&t=1502345139870

```

但是受限于请求URL的长度限制，一般参数较少时会使用get请求。


## POST请求

当参数数量较多，且对数据有一定安全性要求时，会考虑用post请求传递参数数据。POST请求的参数数据是在请求体中。


#### 方式一： Form Data形式

当POST请求的请求头里设置Content-Type: application/x-www-form-urlencoded(默认), 参数在请求体以标准的Form Data的形式提交，以&符号拼接，参数格式为key=value&key=value&key=value...

![3333](https://user-images.githubusercontent.com/5309877/29163839-45a49736-7df0-11e7-8f49-56b6744ca3fc.jpg)

![121212](https://user-images.githubusercontent.com/5309877/29163614-9ac78ea4-7def-11e7-9067-77d73a5d841c.jpg)


前端代码设置：

```
xhr.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');
xhr.send('a=1&b=2&c=3');
```

在servlet中，后端可以通过request.getParameter(name)的形式来获取表单参数。


#### 方式二：Request Payload形式

如果使用AJAX原生POST请求,请求头里设置Content-Type:application/json，请求的参数会显示在Request Payload中，参数格式为JSON格式：{"key":"value","key":"value"...}，这种方式可读性会更好。

![444](https://user-images.githubusercontent.com/5309877/29163866-5b0876ec-7df0-11e7-873b-44731e1becc1.jpg)

![334343](https://user-images.githubusercontent.com/5309877/29163628-a3e89d48-7def-11e7-869a-1f818fd97149.jpg)


后端可以使用getRequestPayload方法来获取。

## Form Data 和 Request Payload 区别
1. 如果请求头里设置Content-Type: application/x-www-form-urlencoded，那么这个请求被认为是表单请求，参数出现在Form Data里，格式为key=value&key=value&key=value...

2. 原生的AJAX请求头里设置Content-Type:application/json，或者使用默认的请求头Content-Type:text/plain;参数会显示在Request payload块里提交。


**参考文档：**
http://www.cnblogs.com/btgyoyo/p/6141480.html
http://xiaobaoqiu.github.io/blog/2014/09/04/form-data-vs-request-payload/
