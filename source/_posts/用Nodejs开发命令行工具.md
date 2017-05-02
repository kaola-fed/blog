---
title: 用Nodejs开发命令行工具
date: 2017-03-24
---

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [前言](#%E5%89%8D%E8%A8%80)
- [准备工作](#%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C)
- [Hello World](#hello-world)
- [内容详解](#%E5%86%85%E5%AE%B9%E8%AF%A6%E8%A7%A3)
  - [index.js](#indexjs)
  - [package.json](#packagejson)
  - [npm link命令](#npm-link%E5%91%BD%E4%BB%A4)
- [发布项目到npm官网供大家使用](#%E5%8F%91%E5%B8%83%E9%A1%B9%E7%9B%AE%E5%88%B0npm%E5%AE%98%E7%BD%91%E4%BE%9B%E5%A4%A7%E5%AE%B6%E4%BD%BF%E7%94%A8)
- [如何处理命令行参数](#%E5%A6%82%E4%BD%95%E5%A4%84%E7%90%86%E5%91%BD%E4%BB%A4%E8%A1%8C%E5%8F%82%E6%95%B0)
- [单元测试](#%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95)
- [让你的项目显得正规(github badges)](#%E8%AE%A9%E4%BD%A0%E7%9A%84%E9%A1%B9%E7%9B%AE%E6%98%BE%E5%BE%97%E6%AD%A3%E8%A7%84github-badges)
  - [持续集成（CI）和代码覆盖率](#%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%EF%BC%88ci%EF%BC%89%E5%92%8C%E4%BB%A3%E7%A0%81%E8%A6%86%E7%9B%96%E7%8E%87)
- [常用的库](#%E5%B8%B8%E7%94%A8%E7%9A%84%E5%BA%93)
- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
<!-- more -->

## 前言

> 找到合适的工具包，开发nodejs命令行工具是很容易的

## 准备工作

- nodejs v6.10.1
- npm v3.10.10

版本号是我当前使用的版本，可自行选择

## Hello World

分4步：
- index.js
- package.json
- 根目录下执行`npm link`
- 执行命令`nhw` => `hello world`

`touch index.js`创建一个**index.js**文件，内容如下：

```js
#! /usr/bin/env node

console.log('hello world')
```

用`npm init`创建一个**package.json**文件，之后修改成如下:
```js
{
    "name": "npmhelloworld", 
    "bin": {
        "nhw": "index.js" 
    },
    "preferGlobal": true,
    "version": "1.0.0",
    "description": "",
    "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1"
    },
    "author": "jerryni <jerryni2014@gmail.com>",
    "license": "ISC"
}
```

## 内容详解

### index.js

> \#! /usr/bin/env node

http://stackoverflow.com/questions/33509816/what-exactly-does-usr-bin-env-node-do-at-the-beginning-of-node-files

这句话是一个[shebang line](https://en.wikipedia.org/wiki/Shebang_%28Unix%29)实例, 作用是告诉系统运行这个文件的解释器是node；
比如，本来需要这样运行`node ./file.js`，但是加上了这句后就可以直接`./file.js`运行了

### package.json

```js
{
    // 模块系统的名字，如require('npmhelloworld')
    "name": "npmhelloworld", 
    "bin": {
        "nhw": "index.js" // nhw就是命令名 ，index.js就是入口
    },
    
    // 加入 安装的时候, 就会有-g的提示了
    "preferGlobal": true,


    // 去掉main: 'xxx.js'  是模块系统的程序入口
    "version": "1.0.0",
    "description": "",
    "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1"
    },
    "author": "jerryni <jerryni2014@gmail.com>",
    "license": "ISC"
}
```

### npm link命令

执行后，控制台里面会有以下输出：

```shell
/usr/local/bin/nhw -> /usr/local/lib/node_modules/npmhelloworld/index.js
/usr/local/lib/node_modules/npmhelloworld -> /Users/nirizhe/GitHub/npmhelloworld
```
解释：创建了2个软链接分别放到系统环境变量$PATH目录里，nhw命令和npmhellworld模块。`npm link`在用户使用的场景下是不需要执行的，用户使用`npm i -g npmhellworld`命令安装即可。

## 发布项目到npm官网供大家使用

设置npm用户名，没有的话先到[npm官方网站](https://www.npmjs.com/)注册一个：

```js
npm set init.author.name "Your Name"
npm set init.author.email "you@example.com"
npm set init.author.url "http://yourblog.com"

npm adduser
```

项目根目录运行：

> npm publish

注意：
> 每次发布需要修改package.json中的版本号，否则无法发布

## 如何处理命令行参数

[当下比较流程的几个工具对比](https://npmcompare.com/compare/commander,minimist,nomnom,optimist,yargs)

这边使用[yargs](http://yargs.js.org/)

> npm install --save yargs

请看之前我实战[一段代码](https://github.com/jerryni/chost/blob/master/index.js#L9-L32)

大概是这个样子：
```js
var argv = yargs
    .option('name', {
        type: 'string',
        describe: '[hostName] Switch host by name',
        alias: 'n'
    })
    .option('list', {
        boolean: true,
        default: false,
        describe: 'Show hostName list',
        alias: 'l'
    })
    .option('close', {
        type: 'string',
        describe: 'close certain host',
        alias: 'c'
    })
    .example('chost -n localhost', 'Switch localhost successfully!')
    .example('chost -c localhost', 'Close localhost successfully!')
    .example('chost -l', 'All host name list: xxx')
    .help('h')
    .alias('h', 'help')
    .epilog('copyright 2017')
    .argv
```

效果：
![yargs](https://www.evernote.com/l/AawRLN3neFtPnqh6JWMYorvfA2aK4rhTslEB/image.png)

## 单元测试

推荐使用[mocha](https://mochajs.org/)

```js
var assert = require('assert');
describe('Array', function() {
  describe('#indexOf()', function() {
    it('should return -1 when the value is not present', function() {
      assert.equal(-1, [1,2,3].indexOf(4));
    });
  });
});
```

执行
```shell
$ ./node_modules/mocha/bin/mocha

  Array
    #indexOf()
      ✓ should return -1 when the value is not present


  1 passing (9ms)
```

## 让你的项目显得正规(github badges)

实际效果就是这些小图标：

![badges](https://www.evernote.com/l/Aay3jpmq3M9ElLB4wZQn6T5dDJ3EmxOSebEB/image.png)

这里可以找到各式各样的badges:
https://github.com/badges/shields

### 持续集成（CI）和代码覆盖率

![travis](https://travis-ci.org/jerryni/chost.svg?branch=master)
[![Coverage Status](https://coveralls.io/repos/github/jerryni/chost/badge.svg?branch=master)](https://coveralls.io/github/jerryni/chost?branch=master)

[travis-ci](https://travis-ci.org/):

- 用github帐号登录，这时网站上列出你github上的项目
- 在项目根目录放`.travis.yml`这个文件， 并写好简单的配置
```
language: node_js
node_js:
  - "6"
```
- 在项目的package.json中添加测试脚本， 因为travis默认会执行npm test
```js
  "scripts": {
    "test": "mocha"
  }
```

在travis设置成功后，继续[覆盖率](https://coveralls.io)的处理：

- 安装2个依赖

> npm install istanbul coveralls --save-dev

- travis的配置文件中添加
```
language: node_js
node_js:
  - "6"
after_script:
  - npm run coverage
```
- 修改`package.json`中的测试脚本
```js
  "scripts": {
    "test": "node ./node_modules/.bin/istanbul cover ./node_modules/mocha/bin/_mocha",
    "coverage": "cat ./coverage/lcov.info | coveralls"
  }
```

## 常用的库

[shelljs](https://www.npmjs.com/package/shelljs)：
> Portable Unix shell commands for Node.js 
> 在nodejs里用unix命令行

[chalk](https://github.com/chalk/chalk)
> Terminal string styling done right
> 给命令行输出上色

## 参考资料

http://javascriptplayground.com/blog/2015/03/node-command-line-tool/
https://medium.freecodecamp.com/writing-command-line-applications-in-nodejs-2cf8327eee2#.3yg7f98dv
http://www.ruanyifeng.com/blog/2015/05/command-line-with-node.html
https://gist.github.com/coolaj86/1318304

