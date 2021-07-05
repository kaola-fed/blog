<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
# Docker 在工程运维上的探索

标签（空格分隔）： docker nodejs webapp

---

### 背景
最近组内有同学用docker在项目中的应用，参加了设计分享，分享完后多数同学都是“我是谁，我从哪儿来，我到哪儿去”的一脸懵圈，这技术已经出现有些年头了，但在我们前端圈里有实践应用的还不多，而这也是我第一次参加docker技术在实践项目中的应用。余是就有了这一篇的学习记录。

#### Docker是啥？
- Docker Client:Docker提供给用户的客户端。Docker Client提供给用户的是一个终端，用户输入Docker提供命令来管理本地或远程服务器。
- Docker Daemon:Docker服务的守护进程。每台服务器上只要安装Docker的环境，基本上就跑有一个后台程序Docker Daemon, Docker Daemon会接收Docker Client发过来的指令，并对服务器进行具体操作。
- Docker Image:镜像。绿色安装程序。
- Docker Registry:是 Docker Image的仓库，就像git仓库一样，用来管理Docker镜像，提供Docker镜像的上传，下载，浏览等，也就Dock Hub.
- Docker Container:Docker 容器。Docker Container是跑项目程序，消耗机器资源，提供服务的地方，Docker Container 通过Docker Images 启动，在Docker Images的基础上运行代码。 Docker Container提供了系统硬件环境，然后使用Docker Images制作好的系统盘，再加上项目代码，就可以运行起来提供服务。


#### Docker怎么玩
- 下载、安装、初遇篇 
    [https://www.jianshu.com/p/d809971b1fc1] 这个链接，有比较完整的下载引导，适合新手直接安装docker，用这篇里如果完整随作者step by step下来，基本都把一个container跑起来

![此处输入图片的描述][1]
这张图比较好的说明了宿主机，docker主机，窗口终端三个载体
- 命令解析
```
docker-machine ssh default
```
Create and manage machines running Docker. Log into with SSH on default machine.

```
docker ps -a
```
![此处输入图片的描述][2]
```
docker --help
```
可以快速查看docker的命令

```
docker rm containerName
```
移除container名称
删除容器还有

- docker stop name
- docker kill name
- docker rmi 删除镜像

```
docker pull node
```
下载安装最新版本的node的linux系统

```
docker run --name koa -v /docker_study/koa-template:/app -p 3000:3000 -i -t node /bin/bash
```
docker run --help 可以查看docker run的参数命令

#### 实践
基于上面的命令的解释，开始一个实例
先在宿机上应射一个本地目录，在windows上的操作上面的那篇引导文章里有指出。然后拉代码到这个目录
```
git clone https://github.com/ltaoo/koa-template.git
```
1. 启动docker machine
```
docker-machine ssh default
```
2. mount命令把宿主机的目录应射到default的docker主机终端上
```
mount
```
3. 下载node的linus镜像
```
docker pull node
```
4. 启动容器,/docker_study/koa-template是代码目录，这个目录会应射到容器的app目录
```
docker run --name koa -v /docker_study/koa-template:/app -p 3000:3000 -i -t node /bin/bash
```
5. 在容器里进入app目录，安装应用依赖
```
npm i
```
6. 启动应用
```
node start.js
```
docker-machine的ip一般是192.168.99.100
所以上面的应用可以能过 http://192.168.99.100:3000 进行访问，同时修改源码里的内容，访问的内容就会修改

#### 基于上面的实践
我们可以总结出一套用于发布工程的方法
![此处输入图片的描述][3]

- 要发布工程时，源代码从指定的gitlab的分支如master上拉代码下来
- 把原来宿主机上的container 重启一下，工程就部署完成了
- 如果要新开一个测试环境，可以新做一个image,然后从指定分支拉代码，在测试容器里进行测试，效率很高

  [1]: https://upload-images.jianshu.io/upload_images/3531509-20b3237c2c4d815b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700
  [2]: https://haitao.nos.netease.com/4f7f82ea-a778-438b-b4dd-d2b4fb952f6b.jpg
  [3]: https://haitao.nos.netease.com/f9ab062d-b153-4137-8eb9-b61b228d49a6.jpg