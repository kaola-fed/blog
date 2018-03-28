
最近听了狼叔关于如何正确学习node的live，进行了一些整理，希望可以对学习或者完善node知识体系的同学有所帮助

### 学习node的三个境界
- 打日志：console.log
- 断点调试：node debugger/node inspector/vscode
- 测试驱动开发(tdd|bdd) [node debug tutorial](https://github.com/i5ting/node-debug-tutorial)

### 如何学习Node
#### 基础学习

##### 个人学习和技术选型循序渐进
- 面向过程：定义一堆function然后调用
- 面向对象
- 函数式编程

##### JavaScript友好语言
- TypeScript

##### 再论面向对象
- 基于原型的写法
- 理解抽象、继承、封装、多态4个基本特征
- JavaScript设计模式

##### Node应用场景(模块维度)
- Api
- Api代理
- IM即时聊天
- 反向代理
- 前端构建工具
- 命令行工具
- 跨平台打包工具
- P2P
- 编辑器
- 物联网与硬件

#### Node核心：异步流程控制

##### 异步流程控制重点
- callback写法
- Promise
- Async函数

##### Api写法：Error-first Callback和EventEmitter
- 定义错误优先的回调写法
    - 回调函数的第一个参数返回error对象，如果error发生了，它会作为第一个err参数返回，如果没有，一般返回null
    - 回调函数的第二个参数返回的是任何成功响应的结果数据，如果结果正常没有error发生，err会被设置为null，并在第二个参数就返回成功结果数据

##### 中流砥柱：Promise
- 要点
    - 递归，每个异步操作返回的都是promise对象
    - 状态机：三种状态转换，只在promise对象内部可以控制，外部不能改变状态
    - 全局异常处理
- 学习资料
    - [Node.js最新技术栈之Promise篇](https://cnodejs.org/topic/560dbc826a1ed28204a1e7de)
    - [理解Promise的工作原理](https://cnodejs.org/topic/569c8226adf526da2aeb23fd)
    - [Promise迷你书](http://liubin.github.io/promises-book/)

##### 终极解决方案：Async/Await
- Async函数语义非常好
- Async不需要执行器，它本身具备执行能力，不像Generator需要co模块
- Async函数的异常处理采用try/catch和Promise的错误处理，非常强大
- Await接Promise，async函数没有纯并行处理机制，可用Promise里的all和race来补齐

##### 迷茫时学习Node.js最好的方法
- 每天看10个npm模块
- [小型库集合](https://github.com/parro-it/awesome-micro-npm-packages)

##### 系统学习Node
- 给别人贡献代码，学别人的习惯，网上有git标准工作流和提pr方法，要做的就是精研该模块代码，关注issue。多读几遍《深入浅出nodejs》，试着阅读node源码
- [《通过开源项目去学习》](https://github.com/i5ting/Study-For-StuQ)
- 跳出node范围，重新审视node的应用场景，对未来的技术选型和决策大有裨益
- 框架选型：个人学习求新，企业架构求稳，无非喜好与场景

### 大前端变化快，如何每日精进

##### 前端痛点
- 基础：oo，dp，命令，shell，构建等
- 编程思想上的理解(mvc、ioc，规约等)
- 区分概念
- 外围验收，如H5和hybird等
- 追赶趋势，如何学习新东西

##### 解决办法
- 玩转npm、gulp这样的前端工具类【前端
- 使用node做前后端分离【前端】
- express、koa这类框架
- jade、ejs等模板引擎
- nginx
- 玩转异步流程处理(promise/es6的generator|yield/es7的async|await)【后端】
- 玩转mongodb、mysql对应的Node模块【后端】
- 学习开源项目，掌握前端项目中Node相关的包管理、测试、ci、辅助模块

### 招聘角度Node开发需要具备的技能
- 基本的Node.js特性：事件驱动、非阻塞I/O、Stream等
- 异步流程控制相关，Promise必问
- 掌握1种以上Web框架，比如Express、Koa、Thinkjs、Restfy、Hapi等，会问遇到过哪些问题、以及前端优化等常识
- 数据库相关，尤其是SQL、缓存、Mongodb等
- 对于常见Node.js模块、工具的使用，观察一个人是否爱学习、折腾
- 是否熟悉linux，是否独立部署过服务器，有加分
- js语法和es6、es7，延伸CoffeeScript、TypeScript等，是否关注新技术，有加分
- 对前端是否了解
- 是否参与过或写过开源项目，技术博客，有加分


参考资料：[如何正确学习 Node.js - 知乎live](https://www.zhihu.com/lives/928687583372926976)