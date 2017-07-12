---
title: 解读 Project Guidelines
date: 2017-07-07
---

解读Project Guidelines

今天早晨在刷Github Trending的时候发现All language的第一名是个名为`project-huidelines`的JavaScript项目，看项目名字和描述可以知道是一个JavaScript项目最佳实践的文章，	内容比较精炼，值得一读。这里不会全部翻译一遍，会有所删减，有兴趣的可以看[原repo](https://github.com/wearehive/project-guidelines)。

<!-- more -->

## 1. Git

#### 1.1 Git Workflow
一些git的操作，不做介绍，是平时用的一些常规操作。
#### 1.2 Some Git Rules
列举了一些应遵守的git规范：
- 在feature分支进行开发工作
- 向`develop`分支提交pull request
- 永远不要向`develop`和`master`推代码
- 在推feature代码之前更新`develop`分支并进行rebase
- 在rebase时解决潜在冲突
- merge后删除本地和远程的feature分支
- 在创建PR前，确保代码测试通过(test & code style check)
- 使用合适的.gitignore文件
- 保护`master`和`develop`分支

这些规则和我参与的数据营销项目的做法基本一致，`master`和`develop`分支是被保护的，新的开发任务从`develop`checkout出feature分支，线上bug从`master`checkout出hotfix分支，feature，hotfix分支均在merge到develop和master上后删除。code style通过eslint保证，不过在合并代码时没有使用`rebase`，在看commit history时会有一些麻烦。

#### 1.3 Writing good commit messages
主要是如何写好commit message的几条建议：
- 将摘要和主体用一个空行隔开
- 将摘要控制在50字符内
- 摘要行用大写
- 摘要结尾不要加句号
- 摘要中使用适当的动词
- 主体部分每行72字符
- 主体部分用来阐述www(what, why, how)

这是引用[How to Write a Git Commit Message
](https://chris.beams.io/posts/git-commit/#end)的内容，事实上做到写好commit message是一件不容易的事情（词穷是硬伤）。规范的commit message是很重要的，写commit message是我们每天都在做的事情，所以要养成写好commit message的习惯，永久受益。

## 2. 文档
- 提供了一个[README 模板](https://github.com/wearehive/project-guidelines/blob/master/README.sample.md)
- 当一个项目由多个repo组成时，在README中注明链接
- 保持README不断更新
- 为代码添加注释，使每个主要模块更易懂
- 为你认为不易理解的小代码块添加注释
- 注释要随着代码的更新而更新

总结一下就是写好`README`和注释，这两点对于项目交接或新成员参与很有帮助。读到这里时我检查了一下数据营销项目的`README.md`，发现有些部分已经过时，比如初始化项目时eslint使用的是`recommend standard`，现在已经重新接入`eslint-config-kaola`规范了。而且`npm scripts`部分新增的功能，如bundle analyze没有在`README`中标注。没有及时更新的`README`可能导致合作者被旧的内容误导，或是对新feature毫不知情，无论哪种都可能要通过增加额外的沟通成本才能解决。

## 3. 环境
- 根据项目大小，分别定义`development`, `test`和`production`环境
- 通过环境变量加载部署时的特定配置，永远不要将它们作为常量写在业务代码中
- 配置项需要从应用代码中分离出来，因为你的程序随时可能会开源。使用`.env`存储一些变量并将这个文件添加到`.gitignore`中
- 建议在启动应用前检查环境变量

这里主要是说明了环境的重要性，可以让程序针对不同环境加载不同的插件。在`development`环境下启用devServer，`production`环境进行资源压缩等。

#### 3.1 一致的开发环境
- 在`package.json`中设置`engines`指定你的项目运行的node版本
- 另外，使用`nmv`，并在项目下创建`.nvmrc`
- 也可以用`preinstall`脚本检查node和npm版本
- 或者在不会让事情变复杂的前提下使用docker镜像
- 使用本地模块而非全局模块

一致的开发环境可以很好地解决"你重启下试试呗"，"我这是好的啊"此类问题。

## 4. 依赖
依赖部分简单说是以下几点：
- 使用靠谱的包，可以通过维护者，更新频率，版本号等来判断
- 保持依赖一致性，使用`package-lock.json`，`npm-shrinkwrap.json`或`yarn`，也能解决"我这里是好的"一类问题。

## 5. 测试
- 如果有必要的话，增加测试模式
- 将模块测试文件紧跟被测试模块后，用`*.test.js`或`*.spec.js`
- 其余的测试文件单独放在test文件夹下
- 书写可测试的代码，避免副作用，写纯函数
- 不要写过多判断类型的测试代码，使用静态类型检查完成这部分工作
- 在提交任何PR前执行测试
- 记录测试，并附上说明

## 6. 项目结构和命名
围绕产品 功能/页面/组件 组织文件，而不是角色

```
// BAD
.
├── controllers
|   ├── product.js
|   └── user.js
├── models
|   ├── product.js
|   └── user.js

```

```
├── product
|   ├── index.js
|   ├── product.js
|   └── product.test.js
├── user
|   ├── index.js
|   ├── user.js
|   └── user.test.js
```

- 测试代码和实现代码放在一起（应该是指可复用模块
- 其他测试文件单独放在test文件夹避免困惑
- 使用`./config`目录，配置文件被使用的值通过环境变量决定
- 将脚本放在`./scripts`目录下，包括用于同步数据库，构建或其他的bash，node脚本
- 使用`camelCase`命名文件和目录名，只有组件使用`PascalCase`
- `CheckBox/index.js`应该包含`CheckBox`组件，或写作`CheckBox.js`但**不应该**是`CheckBox/CheckBox.js`或`checkbox/CheckBox.js`，太多余了。
- 理想情况下，目录名和`index.js`的默认导出一致

这部分主要是目录结构的规范，不一定适用于所有情况。

## 7. Code style
- 构建过程中引入代码检查
- 使用eslint强制统一code style
- 使用Airbnb代码规范
- 使用flow进行类型检查
- 使用`.eslintignore`忽略不需要检查的文件/目录
- 在提交PR前移除所有禁止eslint的注释
- 使用`//todo:`标记未完成的任务
- 禁止`alert`
- e...

Code style就不多说了，组内形成统一规范更重要。

## 8. Logging
- 生产环境避免控制台打log
- 最好使用log库生成可读性好的log

## 9. API 设计
遵循基于资源的设计，分为资源，集合，URL三部分
- 资源包括数据，和其它资源的关系，以及对其进行操作的方法
- 一组资源成为一个集合
- URL标识资源的线上地址

另外操作资源应该使用正确的方法，GET获取，PUT修改，POST新增，DELETE删除。另外使用API version管理接口，一般开放平台接口都会这么做。

## 10. Licensing
确保自己有项目中每个模块的使用权，否则可能有法律问题。

## 总结
这个repo大概就这些内容，从开发流程，分支管理，代码质量，可扩展性，安全性都有所涉及。对比一下自己负责的项目，基于vue-cli创建，所以在各种环境分离，eslint，构建脚本等方面已经帮我们做的很好了，git flow也比较合理，开发起来还是很流畅的，需要改进的地方主要有`README`同步更新，commit message，锁死版本，测试等方面。


