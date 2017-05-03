---
title: CORS 跨域请求
date: 2017-03-16
---

## 跨域资源共享 CORS(Cross-origin resource sharing)

CORS就是，允许浏览器向跨源服务器，发送XMLHttpRequest请求，并获取请求返回内容

<!-- more -->

#### 支持

CORS需要浏览器和服务器同时支持。  
目前，所有浏览器都支持该功能，IE浏览器不能低于IE10。

#### OPTIONS 预请求
检测 —— 客户端发出的请求是否被服务端允许

当客户端发送跨域请求时，浏览器会自发地先发送一个Method为OPTIONS的请求  
请求和你发送请求的URL一致，且不带任何参数  
OPTIONS请求通过，则发送真正的请求，否则将不再发送真正请求  

![optioons预请求](https://haitao.nosdn5.127.net/1489652433458options.jpg?imageView "optioons预请求")

## 跨域配置

### 基本功能配置

**客户端无需配置**

**服务端配置**

    String origin = request.getHeader("Origin"); //获取请求源
    response.setHeader("Access-Control-Allow-Methods", "GET, POST, OPTIONS"); //允许上述Method的跨域请求
    response.setHeader("Access-Control-Allow-Origin", origin); //允许当前源的跨域请求


### Cookie和TLS客户端证书配置
跨域请求默认不发送Cookie以及TLS客户端证书信息，需要手动配置  

**客户端配置**

    xhr.withCredentials = true

**服务端配置**

    response.setHeader("Access-Control-Allow-Credentials", "true");


## TIP

1、OPTIONS请求不会发送Cookie  
2、跨域请求发送的Cookie为请求域下的Cookie


## 问题解决
#### 登录权限判断
异步请求有时会需要根据Cookie来判断是否登录，而options请求是不发送cookie的。  
这时，就需要服务端仅针对options请求不做cookie校验。  
服务端可以通过下面两点来进行判断  
1、method == options  
2、Access-Control-Allow-Headers == X-Requested-With (判断是ajax，需要客户端添加)