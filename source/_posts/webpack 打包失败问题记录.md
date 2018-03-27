今天发生了一个奇怪的事情  

半个月之前已经上线没问题的一个工程，突然构建失败，看报错是 webpack 抛出来的，提示我没有传入有效的参数（或者 webpack.config.js 文件）

于是我把本地的 node 版本和 npm 版本降到了跟服务器一样的 6.9.2 和 3.10.9，删掉 node_modules 然后重新安装依赖，复现了这个错误。

为了定位这个问题
1. 到 webpack 内部去调试，断点不好打，直接改 webpack 的代码加一下 console.log 这种东西，发现根本没有进去 webpack 的函数里面。
2. webpack 是用 yargs 处理传入参数的，因为 webpack 可以不用配置文件，直接将需要的配置跟在 webpack 命令后面就可以运行了。发现在 yargs 阶段就报错了，相当于必填校验没过，进不去服务端。
3. 手动改掉 yargs 的参数，把必填校验给去掉，成功进去打包
4. 怀疑是 webpack 更新了，所以将 webpack 版本降下来了
5. 并没有任何卵用，照样报错，去 github 上看了下 webpack 近期（半个月）并没有发版
6. 这个时候感觉毫无头绪。。。
7. 突然想到还有另外一个工程跟这个工程的配置几乎一样
8. 去部署希望工程，发现希望工程竟然是好的！！！
9. 检查希望工程的 package.json 发现 webpack 的版本是一致的
10. 排除差异，对比十几个不一致的包，把不会引起错误的一些包给去掉，最后只有4个包版本不一致
11. 由于排除差异的过程中会一直尝试打包，一直失败。这个过程得重新安装依赖，超级浪费时间
12. 最后最后，排除到了问题所在，万万没想到是我出错工程里面依赖的一个 yargs 的版本用的 `^10.0.3`
13. yargs 这个东西不可能影响打包的啊，这个只用于起工程，起 mock 的时候的一个小命令行工具，为什么会影响到 webpack 打包呢
14. 调试了一下发现，webpack 自己也依赖 yargs ，但是依赖的版本是 `^8.x`，然而 webpack 用的是我外面高版本的 yargs ，在他自己的 node_modules 里面连 yargs 这个包都没有下下来
15. npm 更新到 5.2.0 就没有这个问题了，说到这里真的不得不吐槽一下我厂的生产测试环境，版本是真的低
16. 仔细研究一下坑爹的 npm 包管理机制，由于 npm3.x 的文档已经找不到了，只能看看目前的版本是怎么做的


### npm 打包机制
#### npm v2
如果我的项目依赖包A和包C，但是A依赖B@1.0，C依赖B@2.0，npm 会无脑的在依赖的A和C各自的 node_modules 里面再去下载各自依赖的包，那么是这样的
![image](https://www.npmjs.com.cn/images/how-npm-works/deps1.png)  
假如B依赖D，D又依赖E，不同的版本的依赖都不同，那么 node_modules 的层级会非常深

#### npm v3
像上面的那样肯定是有问题的，可以做一个简单的优化，把相同的依赖抽出来放在外面  

现在有一个包A@1.0依赖包B@1.0，那么 npm 在第一次去下载B的时候下载一个1.0版本的 B 放在外面
![image](https://www.npmjs.com.cn/images/npm3deps2.png)  
如果还有另外的包也依赖B@1.0的话就不需要额外再去下载了，假如另外的包C依赖的是B@2.0那么没办法只能再去下载一个B@2.0放在B自己的node_modules 里面去  
![image](https://www.npmjs.com.cn/images/npm3deps4.png)  
 
#### npm v3 Duplication 
在上面的基础上继续，如果有多个包依赖B@2.0，那么由于B@1.0在外面，只能多个包都在自己的 node_modules 里面下载一个 B@2.0
![image](https://www.npmjs.com.cn/images/npm3deps6.png)  
这个时候如果有一个E的包依赖B@1.0，那么就不需要再去下载了，用外面的包就可以了
![image](https://www.npmjs.com.cn/images/npm3deps8.png)  
假如A这个包依赖的B的版本更新成 2.0 了，那么A也得在他自己的 node_modules 里面下载一个 2.0 的包
![image](https://www.npmjs.com.cn/images/npm3deps10.png) 
假如这个时候E也更新成了依赖B@2.0，那么没人依赖B@1.0了，但是因为B@1.0还是在外面，所以还是只能在自己的 node_modules 里面下载一个包
但是这样明显就有问题了，我们可以用`npm dedupe`命令让 npm 重新调整一下  
![image](https://www.npmjs.com.cn/images/npm3deps13.png)  
![image](https://www.npmjs.com.cn/images/tree5.png)

#### 总结
npm 安装的时候感觉还是特别的死，根据包的顺序来做一个简单的处理，第一次加载某个依赖的时候会放外面，后面如果有依赖其他版本，无论依赖的次数是否比第一次依赖的次数多，都还是在自己的 node_modules 里面重新再安装。。感觉还是比较笨，还有很大的提升空间

