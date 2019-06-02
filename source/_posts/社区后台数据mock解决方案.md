<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
### NEI常用功能介绍

[NEI(Netease Easy Interface)](https://github.com/NEYouFan/nei-toolkit/blob/master/doc/NEI%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5%E4%BB%8B%E7%BB%8D.md) 是一个为我们提供接口约定、维护的接口管理平台，它同时提供了[自动化构建工具](https://github.com/NEYouFan/nei-toolkit)。

简单介绍下NEI常用的几个功能：

- 有5种添加数据的方式：手动添加、从数据模型导入、从JSON文件导入、从在线接口导入、从JavaBean导入。
  ![](http://chuantu.biz/t6/331/1529567395x-1404764858.jpg)
- 自动化构建工具常用命令：
  - 在当前目录下构建 key 为 xyz 的项目：`nei build -k xyz`
  - 更新通过 `nei build` 构建的项目：`nei update`
  - 启动本地模拟容器：`nei server`



### 社区后台原有数据mock方案

![](http://chuantu.biz/t6/331/1529582674x-1404764858.png)



```json
// package.json
"scripts": {
    "dev": "node build/dev-server.js"
}
```



```javascript
// build/dev-server.js
var express = require('express')
var app = express()
app.use(require('./../mock')
```



```javascript
// /mock/index.js
var fs = require('fs')
var path = require('path')
var stripJsonComments = require('strip-json-comments')

var resolveMockDataPath = function(mockDir, filePath) {
    if (filePath.indexOf('/') === 0) {
        filePath = filePath.slice(1, filePath.length)
    }
    return path.resolve(mockDir, filePath)
}

var readFile = function(extname) {
    extname = extname || '.json'
    return function(filePath) {
        filePath += extname
        let exists = fs.existsSync(filePath)
        if (exists) {
            return fs.readFileSync(filePath, 'UTF-8')
        }
        return exists
    }
}

var readJSONFile = readFile()
var readMockData = function(filePath) {
    return readJSONFile(filePath)
}
var mockDir = path.resolve(__dirname, '../mock')

var getFilePath = require('./mockRouterMap').getFilePath

var initMockMiddleware = function(request, response, next) {
    var requestPath = request.path
    var method = request.method.toLowerCase()
    let mockDataPath = getFilePath(requestPath, method, request.xhr)
    if (mockDataPath) {
        let content = readMockData(resolveMockDataPath(mockDir, mockDataPath))
        if (content) {
            response.status(200).json(JSON.parse(stripJsonComments(content)))
        } else {
            var NO_FOUND_CODE = 404
            response.json(NO_FOUND_CODE, {
                code: NO_FOUND_CODE,
                msg: '接口数据未定义'
            })
        }
    } else {
        next()
    }
}

module.exports = initMockMiddleware
```



```javascript
// mockRouterMap.js
const path2Regexp = require('path-to-regexp')
const MOCK_DATA_DIR = './data'

const initMockRouterReg = function (map) {
	var regMap = new Map()
    for (var pathReg in map) {
        var keyArr = map[pathReg].split(/\s/)
        var pathInfo = {}, urlReg
        if (keyArr.length > 1) {
          	urlReg = keyArr[1]
          	pathInfo.method = keyArr[0].toLowerCase()
        } else {
          	urlReg = keyArr[0]
        }
        pathInfo.mockFile = MOCK_DATA_DIR + map[pathReg]
        regMap.set(path2Regexp(urlReg), pathInfo)
    }
    return regMap
}
var routeMap = {
    'get /api/user/userInfo': '/api/user/userInfo', // 用户信息
    'post /api/novel/list': '/api/novel/list', // 长文-已发布列表
    'post /api/novel/listDraft': '/api/novel/listDraft', // 长文-草稿列表
    'post /api/novel/edit/*': '/api/novel/edit', // 长文-编辑
    'post /api/novel/edit/10086/1': '/api/novel/edit/10086/1', // 长文-编辑-草稿
    'post /api/novel/edit/10086/2': '/api/novel/edit/10086/2', // 长文-编辑-长文
    'post /api/novel/delete': '/api/novel/delete', // 长文-删除草稿
    'post /api/novel/save': '/api/novel/save', // 长文-保存草稿
    'post /api/img/upload': '/api/img/upload', // 上传图片
    'post /api/article/goodsInfo': '/api/article/goodsInfo', // 长文-获取商品信息
    'post /api/novel/user': '/api/novel/user', // 长文-获取用户信息
    'post /api/novel/article': '/api/novel/article', // 长文-获取用户信息
    'get /api/novel/cell/permission': '/api/novel/cell/permission' // 长文-获取用户权限
}
const pathRegMap = initMockRouterReg(routeMap)
module.exports = {
    getFilePath (requestPath, method, isXhr) {
        var filePath = false
        pathRegMap.forEach(function (pathInfo, urlReg) {
            var limitMethod = pathInfo.method
            if (urlReg.test(requestPath)) {
                filePath = pathInfo.mockFile
                if (limitMethod && limitMethod !== method && isXhr) {
                    filePath = false
                }
            }
        })
        return filePath
    }
}
```

从上面代码可以看出，该方案使用本地mock文件存放接口的返回数据。其缺点非常明显：

1. 需要手动维护接口和mock文件的对应关系；
2. 需要手动添加mock文件和数据；
3. 与nei脱离（没有把已有的nei mock数据用起来）；
4. mock方式单一，只能使用本地mock，而不能使用nei线上提供的mock数据、也不能使用线上或测试环境数据。



### 原有数据mock方案与NEI有机结合

NEI作为一个定义、维护接口的平台，使用方便、非常便于接口管理。另外，QA使用的接口测试平台[gotest](https://gotest.hz.netease.com)与NEI对接，这就要求开发必须在NEI上维护接口约定。

那么如何把NEI与原有mock方案有机地结合起来呢？

社区后台的解决方案是：使用原有中间件，利用NEI提供的mock数据和自动化构建方案替换原来的手动mock（包括手动创建mock文件和数据、手动维护接口和mock文件的对应关系）方式，并增加使用线上NEI提供的mock数据功能 和 代理到线上/测试环境的功能。



![](http://chuantu.biz/t6/331/1529573440x-1404764858.png)



```javascript
// /mock/proxy.config.js
const proxy = require('http-proxy-middleware')
const NO_NEED_PROXY = process.env.NO_NEED_PROXY
const proxyTarget = 'http://content-kl.netease.com'
const proxyTable = NO_NEED_PROXY ? [] : [
    proxy('/api', {
        target: proxyTarget,
        changeOrigin: true,
    }),
    proxy('/community', {
        target: proxyTarget,
        changeOrigin: true,
    })
]

module.exports = {
    // 项目的nei唯一标识
    key: '07841b89b63b942b1bb0abcfd090685d',
    // 是否使用 nei 提供的在线 mock 数据
    neiOnline: true,
    // 是否代理到测试/线上环境，只有当neiOnline为false时才有效：true - 代理到proxy target，false - 使用本地mock数据
    useProxy: false,
    // 代理环境配置
    proxyTable
}
```



```javascript
// build/dev-server.js
const { neiOnline, useProxy, proxyTable } = require(path.resolve(__dirname, './../mock/proxy.config.js'))
var express = require('express')
var app = express()
if (neiOnline) {
    console.log('use nei mock data online')
    app.use(require('./../mock/nei-online.js'))
} else if (useProxy && proxyTable.length >= 0) {
    console.log('use proxy')
    app.use(proxyTable)
} else {
    console.log('user local mock')
    app.use(require('./../mock'))
}
```



#### 1. 使用nei提供的在线mock数据

nei本身提供了使用nei在线mock数据的方法：`nei server`可以启动本地模拟容器，设置 server.config.js 文件的 `online: true`就可以使用nei提供的在线mock数据了。

那么不使用`nei server`，该怎么实时拿到nei线上mock数据呢？剖析[nei-toolkit](https://github.com/NEYouFan/nei-toolkit)源码，发现nei上定义的每个接口都可以通过`https://nei.netease.com/api/mockdata?path=${requestPath}&type=3&key=${项目key}&method=${method}`请求来返回结果数据。（其中requestPath是接口url，type为3表示api接口、1表示页面接口，key是项目唯一标识码，method是请求方法如get或post）

所以我们方案是：

![](http://chuantu.biz/t6/331/1529583100x-1404764858.png)

_nei-online.js代码略。_


#### 2. 本地mock

原有的本地mock方案，是根据请求和mock文件的对应关系去取/mock/data下的相应mock文件，那么我们可以**根据nei提供的mock文件替换掉/mock/data下的文件**，**根据server.config.js自动生成接口和mock文件对应关系routeMap**，从而将原有本地mock中间件与NEI有机结合起来。

![](http://chuantu.biz/t6/331/1529585421x-1404764858.png)



```json
// package.json
"scripts": {
    "mock": "NO_NEED_PROXY=true node mock/nei-mock.js"
}
```



```javascript
// /mock/nei-mock.js
const exec = require('child_process').exec
const fs = require('fs')
const path = require('path')
const os = require('os')
const globule = require('globule')
const yargs = require('yargs')
const rimraf = require('rimraf')
const async = require('async')
const { key } = require('./proxy.config')

// 命令行参数
let argv = yargs
    .option('f', {
        alias: 'force',
        describe: 'force to pull data from nei',
        boolean: true,
        default: false
    })
    .help('h')
    .alias('h', 'help')
    .alias('v', 'version')
    .version('0.0.1')
    .usage('Usage: hello [options]')
    .example('npm run mock, npm run mock -- -f, npm run mock -- --force')
    .argv

const neiBaseDir = path.resolve(os.homedir(), 'localMock', key)
const copyTar = path.join(__dirname, './../mock/data')

// 判断文件/文件夹是否已存在
function fsExistsSync (path) {
    try {
        fs.accessSync(path, fs.F_OK)
    } catch (e){
        return false
    }
    return true
}

// 复制文件
let copyFile = (src, tar, cb) => {
    console.log('file update:', tar)
    let rs = fs.createReadStream(src)
    rs.on('error', (error) => {
        if (error) {
            console.log('file read error:', src)
        }
        cb && cb(error)
    })

    let ws = fs.createWriteStream(tar)
    ws.on('error', (error) => {
        if (error) {
            console.log('file write error:', tar)
        }
        cb && cb(error)
    })
    ws.on('close', (ex) => {
        cb && cb(ex)
    })

    rs.pipe(ws)
}

// 复制文件夹
let copyFolder = (srcDir, tarDir, cb) => {
    fs.readdir(srcDir, (error, files) => {
        if (error) {
            console.log('readdir error:', error)
            cb && cb(error)
            return
        }
        files.forEach((file) => {
            let srcPath = path.join(srcDir, file)
            let tarPath = path.join(tarDir, file)
            fs.stat(srcPath, (error, stats) => {
                if (error) {
                    console.log('stat error:', error)
                    return
                }
                if (stats.isDirectory()) {
                    console.log('mkdir:', tarPath)
                    fs.mkdir(tarPath, (error) => {
                        if (error && error.code !== 'EEXIST') {
                            console.log('mrdir error:', error)
                            return
                        }
                        // 无异常 或 已经存在的文件夹(error.code === 'EEXIST')，复制文件夹内容
                        copyFolder(srcPath, tarPath, cb)
                    })
                } else if (file === 'data.json') {
                    // 是文件，且文件名是 data.json
                    let newTarDir = tarDir + '.json'

                    if (!fsExistsSync(newTarDir)) {
                        copyFile(srcPath, newTarDir, cb)
                    } else {
                        console.log('file exist:', newTarDir)
                    }

                    // 删除data.json的上一级目录
                    rimraf(tarDir, (error) => {
                        if (error) {
                            console.log('rmdir error:', error)
                            return
                        }
                    })
                }
            })
        })
        // 为空时直接回调
        files.length === 0 && cb && cb('files is empty')
    })
}

let createMockData = (neiBaseDir) => {
    const copySrcGET = path.join(neiBaseDir, './mock/get')
    const copySrcPOST = path.join(neiBaseDir, './mock/post')
    copyFolder(copySrcGET, copyTar, (error) => {
        if (error) {
            console.log('copy get error:', error)
            return
        }
    })
    copyFolder(copySrcPOST, copyTar, (error) => {
        if (error) {
            console.log('copy post error:', error)
            return
        }
    })
}

// 从nei的server.config.js提取route map
let routeMap = (folderPath) => {
    let srcPath = path.resolve(folderPath, './server.config.js')
    let tarPath = path.join(__dirname, './routeMap.json')

    let serverContent = require(srcPath)
    let { routes } = serverContent

    // 将格式化后的数据写入tarPath所在文件
    fs.writeFile(tarPath, formatRoutes(routes), (error) => {
        if (error) {
            console.log('write file error:', error)
            return
        }
        console.log('update route map: success')
    })
}

// format server.config.js 的 routes，返回格式化后的对象
let formatRoutes = (routes) => {
    let result = {}
    for (let key in routes) {
        result[key] = key.split(' ')[1]
    }
    // JSON.stringify后两个参数可以让json文件换行、4空格缩进 格式化显示
    return JSON.stringify(result, null, 4)
}

let softUpdate = (cb) => {
    const neiServerConfig = path.resolve(neiBaseDir, './nei**')
    let configPathArr = globule.find(neiServerConfig)

    // 从nei拉取mock数据
    const neiBuild = `nei build -k ${key} -o ${neiBaseDir}`
    // nei update: 更新接口文件，但本地已存在的不覆盖；
    // nei update -w: 覆盖已存在的文件，但本地已存在、nei已删除的文件不处理（需要用户手动删除）。
    // const neiUpdate = `cd ~/localMock/${key} && nei update -w`
    const neiUpdate = `cd ~/localMock/${key} && nei update`
    const cmdStr = (configPathArr && configPathArr.length) ? neiUpdate : neiBuild
    console.log('nei exec start:', cmdStr)

    // 每次执行命令，总是先 nei build 或 nei update，然后更新本地的数据
    exec(cmdStr, (error, stdout, stderr) => {
        console.log('nei exec end')
        if (error) {
            cb && cb('cmd exec error')
            console.log('cmd exec error:', error)
            console.log('cmd exec stdout:', stdout)
            console.log('cmd exec stderr:', stderr)
            return
        }

        !configPathArr[0] && (configPathArr = globule.find(neiServerConfig))
        routeMap(configPathArr[0])
        createMockData(neiBaseDir)

        cb && cb()
    })
}

// 删除 ~/localMock/${key}文件
let removeLocalMock = (cb) => {
    console.log('remove localMock start:', neiBaseDir)
    rimraf(neiBaseDir, (error) => {
        if (error) {
            cb && cb('remove localMock error')
            console.log('remove localMock error:', error)
            return
        }
        console.log('remove localMock end')
        cb && cb()
    })
}

// 删除本工程mock/data下的文件
let removeProjectMockData = (cb) => {
    console.log('remove project mock data start')
    fs.readdir(copyTar, (error, files) => {
        if (error) {
            cb && cb('remove project mock data readdir error')
            console.log('readdir error:', error)
            return
        }
        files.forEach((file) => {
            let theFolder = path.join(copyTar, file)
            rimraf(theFolder, error => {
                if (error) {
                    cb && cb('remove project mock data error')
                    console.log('remove project mock data error:', error)
                    return
                }
                console.log('remove project mock data end')
            })
        })
        // 为空时直接回调
        files.length === 0 && console.log('project mock data is empty')
        cb && cb()
    })
}

let hardUpdate = () => {
    async.series([
        removeLocalMock, // 删除 ~/localMock/${key}文件
        removeProjectMockData, // 删除本工程mock/data下的文件
        softUpdate // 重新拉取
    ],
    (err, results) => {
        if (err) {
            console.log('async series error:', err)
        }
    })
}

let main = () => {
    if (argv.f) {
        // 强制从nei拉取数据、覆盖本地mock数据
        hardUpdate()
    } else {
        // 更新nei新增接口、保留本地mock数据
        softUpdate()
    }
}

main()
```

注意：nei拉取到本地的文件结构是在[nei工程规范](https://nei.netease.com/spec/list?s=web&l=all)中定义的。



mockRouterMap.js文件修改（只贴出修改的代码）：

```javascript
// mockRouterMap.js
const fs = require('fs')
const path = require('path')
const ROUTE_MAP = './routeMap.json'

// 删除原先的routeMap
let routeMapPath = path.join(__dirname, ROUTE_MAP)
let routeMap = JSON.parse(fs.readFileSync(routeMapPath))
```



#### 3. 代理到线上/测试环境

通过http-proxy-middleware把请求代理转发到其他服务器，从而响应得到其他服务器上该请求的返回数据。

关键代码：

```javascript
// mock/proxy.config.js
const proxy = require('http-proxy-middleware')

const proxyTarget = 'http://content-kl.netease.com'
const proxyTable = [
    proxy('/api', {
        target: proxyTarget,
        changeOrigin: true,
    }),
    proxy('/community', {
        target: proxyTarget,
        changeOrigin: true,
    })
]

module.exports = {
    // 代理环境配置
    proxyTable
}
```



```javascript
// build/dev-server.js
const { proxyTable } = require(path.resolve(__dirname, './../mock/proxy.config.js'))
app.use(proxyTable)
```



### 线上/测试环境代理到本地debug

当我们需要定位线上或测试环境的问题时，通常的做法是拦截资源请求、使其走本地资源（如使用Fiddler），这样就可以在本地定位问题了。社区组娄涛同学写了个[proxy-local](https://note.youdao.com/group/#/42540264/(folder/180948433//full:md/199088480))（[github](https://github.com/466023746/proxy-local)），可以在不使用代理工具的情况下让线上/测试环境请求本地资源。



### 下一步

- 为了提高此方案的可移植性，计划将其发布到[NPM](https://www.npmjs.com/)平台；
- 进一步封装以提高可用性；
- 整合娄涛的[proxy-local](https://github.com/466023746/proxy-local)（[github](https://github.com/466023746/proxy-local)）；
- 持续迭代。



### 总结

我们拿到一个需求之后，为了达到前后端分离的高效开发方式，开发各阶段都需要不同的数据mock方式。

- NEI数据[生成规则](https://github.com/NEYouFan/nei-toolkit/blob/master/doc/NEI%E5%B9%B3%E5%8F%B0%E7%B3%BB%E7%BB%9F%E9%A2%84%E7%BD%AE%E7%9A%84%E8%A7%84%E5%88%99%E5%87%BD%E6%95%B0%E9%9B%86.md)能为我们造出需要的数据，所以**开发之初**使用**NEI提供的线上mock数据**能减少我们造数据的时间，同时也为团队维护接口数据提供极大便利；
- 在**开发自测阶段**，更多的同学喜欢在**本地**维护自己的**mock数据**，这样可以随意构造出自己想要的数据；
- **联调阶段**可以**代理到测试环境**或后端机器上，从而拿到更真实的数据、模拟更真实的环境，同时可以随时修改问题；
- 当线上或测试环境出现问题时，可以使用@娄涛的[proxy-local](https://github.com/466023746/proxy-local)（[github](https://github.com/466023746/proxy-local)）进行debug。

![](http://chuantu.biz/t6/331/1529572262x-1566657657.jpg)



### 参考

1. https://note.youdao.com/group/#/12651257/(full:collab/113122602)?gid=12651257&filterState=true&noPush=true
2. https://github.com/NEYouFan/nei-toolkit
3. https://github.com/NEYouFan/nei-toolkit/blob/master/README.md
4. https://note.youdao.com/group/#/42540264/(full:md/190946293)?gid=42540264&filterState=true
5. https://www.cnblogs.com/zhoujie/p/nodejs2.html
6. https://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000/001434501497361a4e77c055f5c4a8da2d5a1868df36ad1000
7. http://javascript.ruanyifeng.com/nodejs/fs.html
8. http://nodejs.cn/api/
9. http://www.cnblogs.com/rubylouvre/archive/2011/11/28/2264717.html
10. https://itbilu.com/nodejs/core/E1Abosjbe.html
11. http://ourjs.com/detail/59a53a1ff1239006149617c6
12. https://itbilu.com/nodejs/core/4JGAlesbl.html
13. http://www.ruanyifeng.com/blog/2016/10/npm_scripts.html
14. http://www.webmxx.com/2017/06/13/package-json-script/
15. http://www.ruanyifeng.com/blog/2015/05/command-line-with-node.html