对于软件开发而言，单步调试一直是定位问题的神器，一步一步确认排查可以让 bug 无所遁形。

本文简述 Egg 远程调试方式及实现原理。

## 最简单的 egg 调试流程
关于 egg debug 的命令和具体的命令行参数，可以查看 [使用 egg-bin 调试](https://egg-docs.implements.io/zh-cn/core/development.html#%E6%B7%BB%E5%8A%A0%E5%91%BD%E4%BB%A4-3)。

一条最简单的 egg debug 命令 是:
```bash
$ egg-bin debug
```

你会发现命令行中会输出 console:
```bash
$ Debug Proxy online, now you could attach to 9999 without worry about reload.
$ DevTools → chrome-devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=127.0.0.1:9999/__ws_proxy__
```

将这条链接在浏览器打开，即刻调试 Egg 的 worker 进程

⚠️注意点
* debug 模式打开，守护进程只会开启一个 worker 进程；
* 如果要本地调试测试环境，请把 127.0.0.1:9999 替换成测试环境 ip；

## 如何定制 egg 多个进程的 debug 端口

由于测试机资源问题，可能会在一台云主机上部署多个 Node 应用，此时会出现 debug 端口冲突的问题，可以使用以下方式进行灵活地配置各个进程的 debug 端口号

```bash
$ EGG_AGENT_DEBUG_PORT=7878 egg-bin debug --proxy=9999 --inspect=7879
```

环境变量与参数说明：
* inspect
 * master 进程调试端口直接取自 inspect 设置的值，默认 9229
 * worker 进程调试端口会对 inspect 字段累加后取可用端口
* EGG_AGENT_DEBUG_PORT - agent 调试端口设定，默认 5800
* proxy - worker 进程调试端口设定（转发 websocket 协议到真实的 worker 进程调试端口），默认 9999

## 如何调试 agent 进程
对于一般的用户而言，不太需要关心 `agent` 进程的存在，但是对于插件开发者，可能需要对 `agent` 进程进行调试或者做性能问题排查，此时我们需要点亮 `agent` 进程调试的技能。

执行以下脚本启动应用：

```
$ npm run debug
```

找到第二条 debug 启动日志（master、 agent、worker的启动顺序）
![image](https://user-images.githubusercontent.com/10825163/43468710-3fe86518-9517-11e8-9bab-2826f215179e.png)


默认值为 5800

使用 chrome 浏览器打开 `chrome://inspect`，点击 ` Discover network targets` 👉的 **configure** 按钮
![image](https://user-images.githubusercontent.com/10825163/43468727-4876a8de-9517-11e8-8bb5-6c2f32fa58e1.png)


配置 agent 进程的 debug 端口号
![image](https://user-images.githubusercontent.com/10825163/43468735-4e2fe970-9517-11e8-9a9f-6e3832558d66.png)


点击 done 之后，你会发现 `chrome://inspect` 多了一个 agent 调试的 target
![image](https://user-images.githubusercontent.com/10825163/43468741-524bc81c-9517-11e8-985d-ae4e0cd4c303.png)

## 小结
以上为 egg 应用的一些简单的调试方法，对于 egg 为何需要 proxy debug 协议，以及如何调试任意的 Node 进程，后面再做介绍