---
title: 初识mongoDB（一）
date: 2017-06-05
---

### 参考资料
1. [MongoDB官方网站](www.mongodb.org)：ww.mongodb.org
2. [MongoDB国内官方网站](www.mongoing.com)：www.mongoing.com
3. [MongoDBgithub地址](https://github.com/mongodb)：https://github.com/mongodb
4. [中文MongoDB文档地址](docs.mongoing.com)：docs.mongoing.com
5. [mongodb的jira](https://jira.mongodb.org/)：https://jira.mongodb.org/

<!-- more -->

### 一个mongoDB实例的组成结构如下：
![image](https://haitao.nos.netease.com/62dc5e04-9744-458c-8315-01f7fc3b01ad.png)

- **文档**：是基本单元（类似于关系型数据库中的行）JS中文档就是一个对象。
如：{"greeting" : "Hello, world!"} 就是一个文档

- **集合**：多个文档组成一个集合
如：{"greeting" : "Hello, world!"} {"foo" : 5} 可以存储在一个集合中，集合中一般放入同一种类型的文档。
 
  集合可以有子集合，如：在数据库shell中，db.blog代表blog集合，而db.blog.posts代表blog.posts集合。

- 多个集合可以组成数据库


MongoDB：3.4.2(3大版本，4偶数表示稳定版，2小版本号)

[下载链接](https://www.mongodb.com/download-center?jmp=nav#community)：
https://www.mongodb.com/download-center?jmp=nav#community

---

## OSX安装
**1、解压文件夹（文件内容如下）**
![image](https://haitao.nos.netease.com/63fd5641-295e-4136-a368-c83e1374b6f7.png)

**注：** bin目录下的文件简单介绍
![image](https://haitao.nos.netease.com/3092f303-9d21-4546-a705-1ffa9c251b21.png)

>- **mongod**: 数据库部署
>- **mongo**: 链接mongdb数据服务器的客户端
>- **mongoimport、mongoexport**: mongodb的导入、导出
>- **mongodump、mongorestore**: 导入导出二进制数据（用来做数据的备份恢复）
>- **mongooplog**: 记录操作记录的数据集合
>- **mongostat**: 查看mongodb服务器的各种状态

**2、新建文件夹data/conf/log**
![image](https://haitao.nos.netease.com/7ec995f2-cfc2-4c4a-be90-0b78b8847d38.png)

**3、conf文件夹内新建文件mongod.conf**
> mongod.conf配置文件内容如下：
```
port = 0412                 //端口
dbpath = data               //数据存储的目录
logpath = log/mongod.log    //日志文件
fork = true                  //启动一个后台进程
```

**4、启动**

**启动命令**：

```
mongod
```
```
从配置文件中启动 ./bin/mongod -f conf/mongod.conf
```


![image](https://haitao.nos.netease.com/a5a914db-9a1f-4c06-a40f-68833199123f.png)

**启动 shell**: 

```
mongo
```

![image](https://haitao.nos.netease.com/934041e9-0525-4db9-9539-d827f0d242fc.png)

附：

查看当前使用的数据库
```
db
```
![image](https://haitao.nos.netease.com/ae43ae28-68e2-4b5f-9dcf-88e1780b18aa.png)

可以看出默认使用的是test数据库


```
use kaola
```
切换数据库，没有默认新建一个
![image](https://haitao.nos.netease.com/7687b519-e84e-4ed9-9c80-45009d4fea27.png)

**5、关闭服务器**

```
use admin

db.shutdownServer()
```


### 可能会遇到的一些问题
解决WARNING: soft rlimits too low.

```
ulimit -f unlimited  
ulimit -t unlimited  
ulimit -v unlimited  
ulimit -n 64000  
ulimit -m unlimited  
ulimit -u 64000  
```
添加到 /etc/profile 

./bin/mongo 127.0.0.1:8989

https://www.mongodb.com/blog/post/mongodb-security-part-ii-10-mistakes-that-can 一些警告的解决办法
