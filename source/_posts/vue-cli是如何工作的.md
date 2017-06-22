---
title: vue-cli是如何工作的
date: 2017-06-16
---

vue-cli是Vue.js官方脚手架命令行工具，我们可以用它快速搭建Vue.js项目，vue-cli最主要的功能就是初始化项目，既可以使用官方模板，也可以使用自定义模板生成项目，而且从2.8.0版本开始，vue-cli新增了`build`命令，能让你零配置启动一个Vue.js应用。接下来，我们一起探究一下vue-cli是如何工作的。

<!-- more -->

#### 全局安装
首先，vue-cli是一个node包，且可以在终端直接通过`vue`命令调用，所以vue-cli需要全局安装，当npm全局安装一个包时，主要做了两件事：

1. 将包安装到全局的node_modules目录下。
2. 在bin目录下创建对应的命令，并链接到对应的可执行脚本。

看一下vue-cli的package.json，可以发现如下代码：

```
{
  "bin": {
    "vue": "bin/vue",
    "vue-init": "bin/vue-init",
    "vue-list": "bin/vue-list",
    "vue-build": "bin/vue-build"
  }
}
```
这样在全局安装vue-cli后，npm会帮你注册`vue`, `vue-init`, `vue-list`, `vue-build`这几个命令。

![](https://ooo.0o0.ooo/2017/06/16/59432fe15d720.png)

#### 项目结构
vue-cli项目本身也不大，项目结构如下：
```
.
├── bin
├── docs
├── lib
└── test
    └── e2e
```
`bin`目录下是可执行文件，`docs`下是新特性`vue build`的文档，`lib`是拆分出来的类库，`test`下是测试文件，我们着重看`bin`目录下的文件即可。

#### bin/vue
首先看`bin/vue`，内容很简短，只有如下代码：
```
#!/usr/bin/env node

require('commander')
  .version(require('../package').version)
  .usage('<command> [options]')
  .command('init', 'generate a new project from a template')
  .command('list', 'list available official templates')
  .command('build', 'prototype a new project')
  .parse(process.argv)
```
vue-cli是基于[commander.js](https://github.com/tj/commander.js)写的，支持[Git-style sub-commands](https://github.com/tj/commander.js#git-style-sub-commands),所以执行`vue init`可以达到和`vue-init`同样的效果。

#### bin/vue-init
接下来看`bin/vue-init`，`vue-init`的主要作用是根据指定模板生成项目原型。文件首先是引入一些依赖模块和lib中的辅助函数，因为init命令需要接收至少一个参数，所以`vue-init`第一个被执行到的就是检验入参的[`help`函数](https://github.com/vuejs/vue-cli/blob/master/bin/vue-init#L54)，如果没有传入参数，则打印提示，传入参数则继续运行。

再向下是解析参数的过程：
```
var template = program.args[0]
var hasSlash = template.indexOf('/') > -1
var rawName = program.args[1]
var inPlace = !rawName || rawName === '.'
var name = inPlace ? path.relative('../', process.cwd()) : rawName
var to = path.resolve(rawName || '.')
var clone = program.clone || false

var tmp = path.join(home, '.vue-templates', template.replace(/\//g, '-'))
if (program.offline) {
  console.log(`> Use cached template at ${chalk.yellow(tildify(tmp))}`)
  template = tmp
}
```
`template`是模板名，第二个参数(`program.args[1]`)`rawName `为项目名，如果不存在或为`.`则视为在当前目录下初始化(`inPlace = true`)，默认项目名称`name`也为当前文件夹名。`to`是项目的输出路径，后面会用到。`clone`参数判断是否使用git clone的方式下载模板，当模板在私有仓库时用得上。`offline`参数决定是否使用离线模式，如果使用离线模式，vue-cli会尝试去`~/.vue-templates`下获取对应的模板，可以省去漫长的`downloading template`的等待时间，但是模板是不是最新的版本就无法确定了。

前面在处理参数时会得到一个变量`to`，表示即将生成的项目路径，如果已存在，则会输出警告，让用户确认是否继续，确认后执行`run`函数：
```
if (exists(to)) {
  inquirer.prompt([{
    type: 'confirm',
    message: inPlace
      ? 'Generate project in current directory?'
      : 'Target directory exists. Continue?',
    name: 'ok'
  }], function (answers) {
    if (answers.ok) {
      run()
    }
  })
} else {
  run()
}
```

[run函数](https://github.com/vuejs/vue-cli/blob/master/bin/vue-init#L103)主要检查了模板是否是本地模板，然后获取或下载模板，获取到模板后执行`generate`函数。

generate函数是生成项目的核心，主要代码：
```
module.exports = function generate (name, src, dest, done) {
  var opts = getOptions(name, src)
  // Metalsmith读取template下所有资源
  var metalsmith = Metalsmith(path.join(src, 'template'))
  var data = Object.assign(metalsmith.metadata(), {
    destDirName: name,
    inPlace: dest === process.cwd(),
    noEscape: true
  })
  opts.helpers && Object.keys(opts.helpers).map(function (key) {
    Handlebars.registerHelper(key, opts.helpers[key])
  })

  var helpers = {chalk, logger}

  if (opts.metalsmith && typeof opts.metalsmith.before === 'function') {
    opts.metalsmith.before(metalsmith, opts, helpers)
  }
  // 一次使用askQuestions, filterFiles, renderTemplateFiles处理读取的内容
  metalsmith.use(askQuestions(opts.prompts))
    .use(filterFiles(opts.filters))
    .use(renderTemplateFiles(opts.skipInterpolation))

  if (typeof opts.metalsmith === 'function') {
    opts.metalsmith(metalsmith, opts, helpers)
  } else if (opts.metalsmith && typeof opts.metalsmith.after === 'function') {
    opts.metalsmith.after(metalsmith, opts, helpers)
  }
  // 将处理后的文件输出
  metalsmith.clean(false)
    .source('.') // start from template root instead of `./src` which is Metalsmith's default for `source`
    .destination(dest)
    .build(function (err, files) {
      done(err)
      if (typeof opts.complete === 'function') {
        var helpers = {chalk, logger, files}
        opts.complete(data, helpers)
      } else {
        logMessage(opts.completeMessage, data)
      }
    })

  return data
}
```
首先通过[`getOptions`](https://github.com/vuejs/vue-cli/blob/master/lib/options.js#L14)获取了一些项目的基础配置信息，如项目名，git用户信息等。然后通过`metalsmith`结合`askQuestions`,`filterFiles`,`renderTemplateFiles`这几个中间件完成了项目模板的生成过程。**[metalsmith](https://github.com/segmentio/metalsmith)**是一个插件化的静态网站生成器，它的一切都是通过插件运作的，这样可以很方便地为其扩展。
通过generate函数的代码，很容易看出来生成项目的过程主要是以下几个阶段。

![](https://ooo.0o0.ooo/2017/06/16/594331ee7a506.png)

每个过程主要用了以下库：
- getOptions: 主要是读取模板下的`meta.json`或`meta.js`，`meta.json`是必须的文件，为cli提供多种信息，例如自定义的helper，自定义选项，文件过滤规则等等。该如何写一个自定义模板，可以参考[这里](https://github.com/vuejs/vue-cli#writing-custom-templates-from-scratch)
- 通过Metalsmith读取模板内容，需要注意的是，此时的模板内容还是未被处理的，所以大概长这样:

```
/* eslint-disable no-new */
new Vue({
  el: '#app',
  {{#router}}
  router,
  {{/router}}
  {{#if_eq build "runtime"}}
  render: h => h(App){{#if_eq lintConfig "airbnb"}},{{/if_eq}}
  {{/if_eq}}
  {{#if_eq build "standalone"}}
  template: '<App/>',
  components: { App }{{#if_eq lintConfig "airbnb"}},{{/if_eq}}
  {{/if_eq}}
}){{#if_eq lintConfig "airbnb"}};{{/if_eq}}
```
- 获取自定义配置: 主要是通过[async](https://github.com/caolan/async)和[inquirer](https://github.com/SBoudrias/Inquirer.js)的配合完成收集用户自定义配置。
- filterFiles: 对文件进行过滤，通过[minimatch](https://github.com/isaacs/minimatch)进行文件匹配。
- 渲染模板：通过[consolidate.js](https://github.com/tj/consolidate.js)配合[handlebars](https://github.com/wycats/handlebars.js/)渲染文件。
- 输出：直接输出

`vue-init`的整个工作流程大致就是这样，`vue-cli`作为一个便捷的命令行工具，其代码写的也简洁易懂，而且通过分析源码，可以发现其中用到的很多有意思的模块。

#### bin/vue-list
[vue-list](https://github.com/vuejs/vue-cli/blob/master/bin/vue-list)功能很简单，拉取[vuejs-templates](https://api.github.com/users/vuejs-templates)的模板信息并输出。

#### bin/vue-build
[vue-build](https://github.com/vuejs/vue-cli/blob/master/bin/vue-build)则是通过一份webpack配置将项目跑起来，如果是入口仅是一个`.vue`组件，就使用默认的`default-entry.es6`加载组件并渲染。

#### 其他
在看vue-cli源码时，发现了[user-home](https://github.com/sindresorhus/user-home)这个模块，这个模块的内容如下：
```
'use strict';
module.exports = require('os-homedir')();
```
`os-homedir`这个包是一个`os.homedir`的polyfill，在[Why not just use the os-home module?](https://github.com/sindresorhus/user-home#why-not-just-use-the-os-home-module)下，我看到了[Modules are cheap in Node.js](https://github.com/sindresorhus/ama/issues/10#issuecomment-117766328)这个blog。事实上[sindresorhus](https://github.com/sindresorhus)写了很多的One-line node modules，他也很喜欢One-line node moduels，因为模块越小，就意味着灵活性和重用性更高。当然对于One-line modules，每个人的看法不一样，毕竟也不是第一次听到**“就这一个函数也tm能写个包”**的话了。我认为这个要因人而异，sindresorhus何许人也，很多著名开源项目的作者，发布的npm包1000+，大多数他用到的模块，都是他自己写的，所以对他来说，使用各种“积木”去组建“高楼”得心应手。不过对于其他人来说，如果习惯于这种方式，可能会对这些东西依赖性变强，就像现在很多前端开发依赖框架而不重基础一样，所以我认为这种“拼积木”开发方式挺好，但最好还是要知其所以然。但是我感觉One-line modules的作用却不大，就像user-home这个模块，如果没有它，`const home = require('os-homedir')();`也可以达到目的，可能处于强迫症的原因，user-home才诞生了吧，而且像[negative-zero](https://github.com/sindresorhus/negative-zero)这样的One-line modules,使用场景少是其一，而且也没带来什么方便，尤其是2.0版本，这个包直接使用Object.is去判断了:
```
'use strict';
module.exports = x => Object.is(x, -0);
```
不知道大家对One-line modules是什么看法？
