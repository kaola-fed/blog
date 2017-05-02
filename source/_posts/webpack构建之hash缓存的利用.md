---
title: webpack构建之hash缓存的利用
date: 2017-04-26
---

# webpack构建之hash缓存的利用

众所周知，优化一个页面的方法之一就是利用浏览器的缓存机制，在文件名不改变的情况下，使客户端不用频繁向服务端请求和重复下载相同的资源，节约了流量的开销。

现在前端的主要构建工具就是使用webpack，现在就说说使用webpack时遇到的hash的坑。

<!-- more -->

在生产环境发布时，为了利用缓存，都会加一个标识，webpack打包时也可加入这个功能，对于一个基本的应用，最开始的用法估计这样，见如下代码

```
module.exports = {
    entry: {
        index: path.resolve(__dirname, '../src/js/index.js'),
        index2: path.resolve(__dirname, '../src/js/index2.js'),
        vendor: ['react', 'react-dom'] //第三方模块
    },

    output: {
        path: path.resolve(__dirname, '../dist'),
        filename: '[name].[hash:8].js',
        publicPath: '/'
    },
    
    module: {
        rules: [{
            test: /\.jsx?$/,
            use: ['babel-loader'],
            exclude: /node_modules/
        }
        //省略其他loader
        ]
    },
    
    plugins: [
    	 new CommonsChunkPlugin({
            name: 'vendor'
        })
        //省略其他plugin
    ] 
}
```
打包结果如下：
<img width="715" alt="hash1" src="https://cloud.githubusercontent.com/assets/5706155/25329966/83f95b50-2910-11e7-9376-af537aa9649c.png">

这里的hash是每次编译时所计算出来的一个值，因此当任何一个文件修改，hash都将会改变，而且所打包出来的文件名也将不一样了，这样对于需频繁发布的项目不是很友好，会造成每次有新版本，用户浏览器都将重新下载所有的文件，为了避免这种情况，webpack还提供了chunkhash（**注**：这里提取出的css文件hash值不一样，是因为使用的是extract-text-webpack-plugin插件，它提供了自己的一个contenhash，也是对于css文件建议的一种用法，保证了css有自己独立的hash，不会受到js文件的干扰）

将上述代码中的hash换成chunkhash后打包结果如下：
<img width="647" alt="hash2" src="https://cloud.githubusercontent.com/assets/5706155/25329965/83f6f932-2910-11e7-9a79-f326e6357080.png">
看到每个chunk有自己单独的hash值，此时只修改某一个模块里的文件，将不会影响到其他的模块打包出的hash值了，这样也就能充分利用hash缓存了。

----
你以为到此处就结束了吗，too young too simple！

你可以试试改动一个文件，打包后vendor的hash值每次都是在变化的，第三方模块是最不长改动的模块，更应该被缓存住，可为什么vendor的chunkhash总是变化，是因为webpack runtime由于entry对应的Id变化而发生了变化，chunkhash的计算又依赖于runtime，因此vendor的chunkhash也发生了变化。

为了解决这个问题，我们首先得保证chunkId的稳定，参考webpack2的文档[caching](https://webpack.js.org/guides/caching/)，可以使用HashedModuleIdsPlugin的插件(webpack1也可以使用，但需要将[HashedModuleIdsPlugin.js](https://github.com/webpack/webpack/blob/master/lib/HashedModuleIdsPlugin.js)自行引入)，然后就是创建一个额外的chunk来提取runtime，就是文档中说的manifest，部分代码如下

```
plugins: [
    new CommonsChunkPlugin({
	names: ['vendor', 'manifest'],
        minChunks: Infinity
    }),
    new webapck.HashedModuleIdsPlugin()
]
```
打包后将多出一个很小的manifest.js的文件，但保证了vendor的hash没有改变，对于manifest的内容我们需要优先引入，在此可以借助[inline-manifest-webpack-plugin](https://github.com/szrenwei/inline-manifest-webpack-plugin)将manifest内容内联进html文件中，以免多发一次js的请求，使用方式可直接参考文档。

至此对于hash的变化才算真正结束，才能达到利用hash的变化真正的控制住缓存。

*PS.* 建议不要用webpack-md5-hash，会有坑的，见[webpack-md5-hash-issue](https://github.com/erm0l0v/webpack-md5-hash/issues/7)

## 参考资料：

[webpack的cache](https://webpack.js.org/guides/caching/)

[extract-text-webpack-plugin](https://github.com/webpack-contrib/extract-text-webpack-plugin)

[webpack-md5-hash](https://github.com/erm0l0v/webpack-md5-hash)

[http缓存](http://harttle.com/2017/04/04/using-http-cache.html)

[inline-manifest-webpack-plugin](https://github.com/szrenwei/inline-manifest-webpack-plugin)

[webpack-md5-hash-issue](https://github.com/erm0l0v/webpack-md5-hash/issues/7)