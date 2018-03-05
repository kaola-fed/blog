<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

某个后台工程, 用了ts+regularjs+es6, webpack 打包非常耗时和耗内存.

首先使用DllPlugin替换commonChunkPlugin, 将一次打包分成连个独立的打包过程: 第三方库打包和业务逻辑打包. 这样做之后, 内存消耗和耗时有所降低. 

对于DllPlugin和CommonChunkPlugin的区别, [Stack Overflow有一个解答](https://stackoverflow.com/questions/41890855/webpack-common-chunks-plugin-vs-webpack-dll-plugin), 对于我们这个项目的打包来说, 使用DllPlugin带来的好处是可以将整一个打包的过程拆成为几个打包过程, 然后可以比较清晰的看出究竟是哪一个过程消耗的性能比较大.

然后从[这个文章](https://survivejs.com/webpack/optimizing/build-analysis/)找到[这个git repo](https://github.com/webpack/analyse)和[这个网站](http://webpack.github.io/analyse), 
执行以下命令, 可以在打包时生成一个打包过程的数据文件, 
```
webpack --profile --json > stats.json
```
然后上传到该网站上, 网站会分析此文件, 然后给出一些优化建议.

经过分析, 主要给出了两个性能消耗大的原因:
### Module in multiple chunks
Check if it is a good idea to move modules into a common parent. You may want to use require.include or insert them into the parents require.ensure array.

### Long module build chains
Use prefetching to increase build performance. Prefetch a module from the middle of the chain.

对于第一个问题, 给出的建议是使用`require.include`或者`require.ensure`, 这俩东西是啥呢? 在[这个文章](https://www.cnblogs.com/lvdabao/p/5953884.html)中找到一句

> 在commonjs中有一个Modules/Async/A规范，里面定义了require.ensure语法。webpack实现了它，作用是可以在打包的时候进行代码分片，并异步加载分片后的代码。用法如下：
```js
require.ensure([], function(require){
    var list = require('./list');
    list.show();
});
```
这样看来, 使用这种优化方式需要把所有引用了公共模块的地方用`require.ensure`把代码包起来, 而这对于使用es6的import写法加载模块的代码来说, 改动量太大了. 接着查了一下有没有用es6的方式使用这个优化方案的方法, 结果是: 不同于commonjs, es6是静态解析, 因此无解.

从公共模块的类型上看, 除了第三方的库, 还有一些业务代码里的公用方法和基类的文件, 于是把这部分业务逻辑的代码也单独拿出来作为一个dll来打包. 这样就产生了三个打包过程: 
1. 第三方库的dll 打包
2. 公共业务逻辑代码的dll打包
3. 普通业务逻辑代码的打包

从打包耗时看, 比较耗时的是第二和第三个打包过程. 

第一个优化建议无法执行, 来看第二个优化建议. 第二个优化建议是说构建链过长, 说白了就是一个文件A内部import了文件B, B又import了文件C, 形成的依赖链条. 这个链条到webpack打包时就是一个构建链, 如下:

> A => B => C

针对此种情形, 给出的建议是从构建链的中间预获取一个模块. 找到了一个[PrefetchPlugin](https://webpack.js.org/plugins/prefetch-plugin/)以及[使用示例](https://github.com/FrendEr/webpack-optimize-example/blob/master/prefetch-plugin/webpack.config.js)

在webpack配置中使用了一下如下:
```js
plugins: [
...
    new webpack.PrefetchPlugin(path.join(__dirname, './mcss/reset.mcss')),
...
]
```

下图是使用前和使用后的对比

![image](https://user-images.githubusercontent.com/2340296/36789210-abf74b6c-1ccb-11e8-8978-db0e84f022f4.png)

![image](https://user-images.githubusercontent.com/2340296/36789325-17c34f94-1ccc-11e8-977a-ba2e4f960c95.png)


一下是使用PrefetchPlugin之前和之后的构建链id列表: (构建链个数没变)
使用前: 
- 17, 22, 32, 31, 38
- 17, 22, 32, 31, 37
- 17, 22, 32, 31, 30
- 17, 22, 32, 31, 39, 40
- 17, 22, 32, 31, 39
- 17, 23, 36
- 17, 23, 34
- 17, 22, 33
- 17, 22, 35
- 17, 22, 32, 31

共42个节点

使用后:
- 22, 32, 31, 38
- 22, 32, 31, 37
- 22, 32, 31, 30
- 28, 17, 26, 24
- 28, 17, 27, 18
- 28, 17, 23, 34
- 28, 17, 23, 36
- 28, 17, 23, 35
- 28, 17, 23, 33
- 22, 32, 31, 39
共40个节点


从截图中的一来关系来看, 把`reset.mcss`添加到prefetch中后, 原来构建链中一来reset.mcss的第一项没有了, `reset.mcss`变成了第一项;
从构建链的长度来看, 之前长度为5和6的构建链, 变成了4, 而长度为3的构建链, 也变成了4;
从构建链节点数来看, 使用了prefetch之后, 构建链的总节点数少了两个

接下来, 再添加一个prefetch, 如下:
```js
plugins: [
    new webpack.PrefetchPlugin(path.join(__dirname, './mcss/reset.mcss')),
    new webpack.PrefetchPlugin(path.join(__dirname, './javascript/component/base-new.ts')),
]
```

构建链如下:
- 22, 32, 31, 37
- 22, 32, 31, 30
- 22, 32, 31, 38
- 17, 26, 24
- 17, 27, 18
- 17, 23, 34
- 17, 23, 33
- 17, 23, 36
- 17, 23, 35
- 22, 32, 31, 39, 40

构建链依然是10个, 总节点数变成了35个

虽然构建链的节点数每次都在减小, 但是从分析表上的每条构建链的耗时来看, 却是一次比一次增加: 4126ms => 5000ms => 5325ms

为了再次对比, 我把--profile参数去掉了, 直接从webpack打印到终端的日志来看打包时间, 注释掉prefetchPlugin之前和之后的公共业务逻辑代码打包耗时分别为 7478ms 和 7448ms
将dist目录删掉, 重新打包所有dll(包括第三方库的打包和公共业务逻辑的打包), 两次打包公共代码的耗时分别为7858ms 和 7555ms
所以, 加了prefetchPlugin, 对打包的性能只优化了300多毫秒, 可以说是比较没什么卵用.

再想一想, 前面的第三方库打包为什么那么快呢? 第三方库的代码量也不少, 它和业务代码有什么区别呢? typescript 和mcss! 业务逻辑里用到了typescript和mcss. 是不是ts-loader和mcss-loader打包的速度慢呢? 

接下来, 把公共逻辑代码中引用到的mcss文件改成css文件, 再打包, 打包速度没有明显提升; 把公共逻辑代码中引用了样式文件的公共文件从打包中去掉, 只打util类的文件, 打包速度依然没有明显提升. 这说明mcss并不是影响打包速度的因素. 

下一步, 公共逻辑代码中的ts文件全部注释掉, 只保留js文件进行打包 (共10个ts文件, 3个js文件) 
打包时间从原先的5000多毫秒一下减少到了480多毫秒! 打开一个注释掉的ts文件`action.ts`再打包, 打包时间又回到了5000多毫秒. 至此, 可以得出结论, 打包过程中大量的性能是消耗在对ts文件的处理过程中.

而可笑的是, 这个唯一被加入打包处理的文件, 全部内容只有如下几行:
```typescript
function type(type: any): any {
    return {
        type,
    }
}

export {
    type,
}

```

为了继续将这个ts文件简化, 我将文件内容改成了如下这样: 
```typescript
function a(b: any): any {
    return b
}

export {
    a,
}

```

再次打包时, 得到了一些报错, 报错是从一些引用了`action.ts`的文件中抛出的, 大意是说`action.ts`并没有导出一个叫`type`的方法. 当然没有这个方法了, 因为被我改掉了嘛. 

但是等等, 我们打包公共代码的时候, 为什么ts(或者说ts-loader)会去检查打包文件以外的文件呢? 性能的消耗是否就是从这里引入的?

带着这个疑问, 在网上查到了这个问题: [WebPack ts-loader compiling all files when I only want it to run in one folder/file](https://stackoverflow.com/questions/41289265/webpack-ts-loader-compiling-all-files-when-i-only-want-it-to-run-in-one-folder-f#) 这不就是我目前面对的问题么! 答案也很简单:只要在ts-loader里加个配置就好了, 如下:
```js
{
    test: /\.ts$/,
    loader: 'ts-loader',
    options: { onlyCompileBundledFiles: true }
},
```

加了这个配置后, 再讲刚才注释掉的所有ts文件打开, 执行打包, 打包时间果然大幅降低: `Time: 2086ms`

以为一切大功告成, 但是又发现打包普通业务逻辑的耗时依然很大, 在10秒左右. 看来加了`onlyCompileBundledFiles`参数只解决了公共ts打包的性能问题, 还有其他地方存在性能瓶颈.

在上面提到的性能分析网站上传打包日志后, 给出的建议还是上面提到的两个建议

继续拆分. 这次将普通业务逻辑中用来遍历生成entry的文件夹中, 只保留一个entry, 即将原先的17个entry删掉17个. 耗时对比如下:

|entry个数 | 耗时(ms)|
|--| --|
|17  |  10316  |
|1    | 7683    |

从上表看出, 砍掉超过90%的代码, 打包时间只减少了约25%.说明这里还有一些问题. 这和刚才打DLL时的现象太相似了! 是不是这次写的`onlyCompileBundledFiles`没有生效?
 
我将一个理论上已经不再当前打包范围内的ts文件(其entry已经被删除, 所以不会被webpack扫描到)里写上一个语法错误, 再进行打包. 如果`onlyCompileBundledFiles`生效, 这个语法错误应该不会影响打包. 

![image](https://user-images.githubusercontent.com/2340296/36846398-6608d88a-1d95-11e8-9414-bd2c3c27f704.png)

见证奇迹的时刻! 果然报错了!

![image](https://user-images.githubusercontent.com/2340296/36846388-5eed628c-1d95-11e8-9f06-fac6babf903a.png)

在happypack的repo里找了找, 又找到一个参数`transpileOnly`, 也是一个提升打包速度的参数, 但是没看明白和onlyCompileBundleFiles有什么区别, 而且这个参数在设置了`happyPackMode: true`的情况下, 是开启的. 也就是说, 目前我在webpack里已经将这两个参数打开了. 为什么在打dll的时候, 设置`onlyCompileBundleFiles: true`会忽略其他不相干的ts文件, 而这次在打包普通业务逻辑时就不管用了呢?
很奇怪.

在ts-loader的repo里发现了这个issue: [Tries to compile sources that are not supposed to be loaded](https://github.com/TypeStrong/ts-loader/issues/267), 

经过老夫的不懈努力, 发现了问题所在. 如果在plugins里加入了 `new ForkTsCheckerWebpackPlugin()`, 那么ts就会检查其他不相干的文件的语法. 如果把这个插件去掉, 那么ts就会忽略不在打包范围的文件, 打包时间也大幅度减小, 从7000多毫秒降低到了4000多毫秒, 再讲之前删掉的entries加回来, 再次打包, 打包时间为7000多毫秒. 

因此, 在plugins里去掉了一个`ForkTsCheckerWebpackPlugin`, 将打包时间从10000多毫秒降低到了7000多毫秒. 

以上这些加加减减, 虽然将整个工程的打包耗时降低了几秒, 但是总时间还是很大. 总时间还十几秒. 接下来试试awesome-typescript-loader, 看能否在此基础上继续降低打包耗时(估计够呛).

在试atl之前, 需要再次确定一下是否打包的大部分时间浪费在了typescript部分. 要做到这一点, 需要知道我们配置的webpack中, 每一个loader耗费的时间. 找了一下, 找到这一个 [speed-measure-webpack-plugin](https://www.npmjs.com/package/speed-measure-webpack-plugin). 使用方式很简单, 把整个导出的webpack配置用这个plugin的实例包裹一下, 就可以了, 如下:
```javascript
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin');
const smp = new SpeedMeasurePlugin();
module.exports = smp.wrap(webpackConfig) 
```

再次执行普通业务逻辑打包(公共业务逻辑代码打包时会报ts找不到namespace的错, 具体原因有待调查), 这次打包的时间比原来变长了两三倍(21s左右). 看了这个工具本身也消耗很多的内存. 打包之后, 得到如下输出:
![image](https://user-images.githubusercontent.com/2340296/36954249-a4ad09b2-205b-11e8-85ad-05892e0fbcc6.png)

从输出可以看出, HtmlWebpackPlugin占用了绝大部分的时间.
把webpack中用到HtmlWebpackPlugin的代码注释掉, 重新打包, 发现打包前后消耗的时间差异只有500ms不到. 说明这个鬼测试工具不仅没有什么卵用, 而且还会给出错误的结论.

突然想到一个点, 因为在windows系统vscode无法解析tsconfig文件中的`compilerOptions.paths`配置项. 所以在一些引用的依赖存在自定义路径名的文件中, 加入了大量的`/// <reference path="xxx" />`, 在[typescript的文档](https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html)中说道:

> Triple-slash references instruct the compiler to include additional files in the compilation process.

可以想见, 这个东西是会损耗性能的. 于是把所有文件中的三斜线注释去掉, 发现打包时间从7秒多减小到了5秒多

又想到一点, 既然ts开启了js和ts混合编译的选项, 是不是把babel去掉, 直接都用ts来编译会快一些呢? 
经过试验, 答案是否定的. 把`test: /\.ts$/`改成`test: /\.(j|t)s$/`, 然后删掉babel的loader等配置, 再进行打包, 打包时间又从5s多变成了7s多.

### 总结
#### 结论 ts打包是会消耗比较大的内存, 经过一系列优化, 打包时间缩减了5秒左右

使用的优化步骤:
1. 去掉`fork-ts-checker-webpack-plugin`;
2. 公共业务逻辑代码单独打包;
3. 公共的样式文件由mcss改成css;
4. 去掉ts文件中的三斜线注释;
5. 使用`DllPlugin` 和 `DllReferencePlugin` 替代`CommonChunkPlugin`;
6. 使用webpack4 (其实使用webpack4并没有带来明显的性能提升, 但是考虑到很多plugin和loader都更新到适配webpack4的版本了, 索性一并升级了)