---
title: 如何把一个 RegularJS 组件打成 npm 包
date: 2017-07-04
---

本篇基于 RegularJS 热区组件，简单分享一下组件打包发布的全流程及主要遇到的问题。

## 目录

1. 项目初始化
2. 开发环境准备，安装基础依赖
3. 将组件打包输出成多种模式!important
4. 进入开发
5. 包装工作
6. 最终发布

<!-- more -->

## 1. 项目初始化

#### 1. 在 GitHub 上创建项目仓库，添加 README 和 License
    
没什么好说的，License 一般设置成 MIT（开源万岁），详细协议介绍可查：[HELP](https://choosealicense.com/)。

#### 2. clone 到本地，设置 git config
    
本地全局的 git config 文件一般设置为公司的邮箱和用户名。为了避免泄露信息，可在初始化时提前进行项目层面的 config 设置：

```shell
$ git config user.name "GitHub 用户名"
$ git config user.email "GitHub 邮箱"
```

这样提交代码时就以该用户名及邮箱作为用户信息了，此时执行查看命令可以看到：

```shell
$ cat .git/config
[user]
    name = GitHub 用户名
    email = GitHub 邮箱
```

#### 3. 执行 npm init，生成 package.json

按提示一步步来完成配置即可。

## 2. 开发环境准备，安装基础依赖

这里偷了个懒，直接使用了 vue-cli 的 [webpack-simple](https://github.com/vuejs-templates/webpack-simple) 模式生成的 webpack.config.js 和 package.json，并调整成实际需要的配置。

配置比较简单，可以直接 [**戳我**](https://github.com/Deol/regular-hotzone/blob/master/webpack.config.js) 看一下（具体配置解释直接查阅 [文档](https://webpack.js.org/configuration/output/#output-librarytarget)）。

## 3. 将组件打包输出成多种模式!important

既然是 RegularJS 组件，那么打包后的组件无论是直接以 `<script>` 标签形式引入，或者用 AMD / CommonJS 方式引入都应该可以使用。

### 第一部分，webpack 配置

与此相关的配置项是这三个：

 - output.library && output.libraryTarget

library 属性能让打包后的整个组件被当成一个全局变量使用。考虑命名污染及冲突，可以将 `library` 属性的值起得相对复杂些，如 `regularHotZone`。

另外，为了让组件在多种模式下都可运行，使用 `libraryTarget` 配置该组件的暴露方式为 **umd**。该模式意味着组件在 CommonJS、AMD 及 global 环境下都能运行：

```
output: {
    library: 'regularHotZone',
    libraryTarget: 'umd'
}
```

 - externals

这个配置是为了排除外部依赖，不将它们一起打包进去。对于 RegularJS 组件来说，并不需要把 RegularJS 也打包进去，此时就应该用 externals。

而配置中是这么写的：

```
externals: {
    regularjs: {
        root: 'Regular',
        commonjs: 'regularjs',
        commonjs2: 'regularjs',
        amd: 'regularjs'
    }
}
```

这是由于上述的 libraryTarget 设置为 umd，那么这里必须设置成这种形式，RegularJS 才能在 AMD 和 CommonJS 模式下通过 regularjs 被访问，但在全局变量下通过 Regular 被访问。

### 第二部分，package 配置

另一方面，我们需要在组件的 package.json 中需要将 RegularJS 设置为同伴依赖 (`peerDependencies`)：

```
// 建议：不同于一般的依赖，同伴依赖需要降低版本限制。不应该将同伴依赖锁定在特定的版本号。
"peerDependencies": {
    "regularjs": "^0.4.3"
}
```

因为 RegularJS 组件是 RegularJS 框架的拓展，它不能脱离于框架独立存在。

也就是说，如果需要以 npm 包形式引入 RegularJS 组件，那么 RegularJS 框架必须也被引入，不管是以 npm 包形式引入，还是用 `script` 标签引入并配置 externals。

**注意**：如果安装组件包时，找不到 RegularJS 或者其**不符合同伴依赖的版本要求**，终端将抛出警告：

```
`-- UNMET PEER DEPENDENCY regularjs@^0.4.3

npm WARN regular-hotzone@0.1.14 requires a peer of regularjs@^0.4.3 but none was installed.
```

npm 使用的版本规则「[**在此**](https://docs.npmjs.com/misc/semver)」查看。

可以知道，上面设定 RegularJS 版本为 `^0.4.3`，相当于 version >= 0.4.3 && version < 0.5.0。

## 4. 进入开发

跑个 `npm run startdev`，balabalabala...

## 5. 包装工作

 1. 完成整体开发后，修改 package.json 中的 version（版本介绍「[**在此**](http://semver.org/lang/zh-CN/)」，每次发布都必须修改，否则无法发布），并利用 `npm run build` 打包输出 dist 文件夹。

 2. 编写 Readme，可参考「[如何写好 Github 中的 readme？ - 知乎](https://www.zhihu.com/question/29100816/answer/68750410)」。

## 6. 最终发布

最终阶段，进入 https://www.npmjs.com/ 完成注册后，执行：

```
$ npm publish
```

完成登录后可能会发布失败，因为我们可能会将 npm 源设置为淘宝源，此时需要添加 `//` 暂时将其注释：

```
$ vi ~/.npmrc

//registry=https://registry.npm.taobao.org
```

保存后重新执行发布操作即可。

此时我们可以通过 npms.io 搜索 npm 包名，如（请忽略分数）：

![npms](https://user-images.githubusercontent.com/4961878/27834960-9bd1e728-610b-11e7-9de6-2e64a1c110e3.png)

并通过其[**分析**](https://npms.io/about)增强 npm 包的质量，最简单的可以有：

 - 完善 Readme、license、.gitignore 等；
 - 接入 [Travis CI](https://travis-ci.org/) 等，并确保覆盖率；
 - 去除过时依赖，减少依赖的脆弱性；
 - 增加专属站点，添加 Readme 上面的 icons；
 - 接入 ESLint，实现静态代码检查；
 - ...