---
title: 写给老古董们看的现代JavaScript
date: 2018-01-02
---

> 译者按：</2017><2018>，前端圈又度过了喧嚣的一年，整个JavaScript的技术栈趋于稳定与成熟，越来越多的服务端知识与架构被引进与运用，也许不久的未来，人人都是全栈工程师。然而在这之前，还是让我们看看JavaScript发生了什么样的变化。本篇译自2017年medium的十月热文[Modern JavaScript Explained For Dinosaurs](https://medium.com/the-node-js-collection/modern-javascript-explained-for-dinosaurs-f695e9747b70)，相较于一般的技术类博文，它更像科普，作者以他的个人经历，缓缓讲述了JavaScript技术的变迁。

如果你不是一开始就接触现代JavaScript的话，那么学习它是一件十分困难的事情。整个JavaScript的生态系统变化的太快了，以至于初学者难以理解不同的工具所针对解决的问题。我（作者）从1998年就开始编程，但是开始认真学习JavaScript是在2014年。记得当时我首先接触的是[Browserify](http://browserify.org/)，它的标语是这样的。

> Browserify允许你在浏览器中通过require('modules')的方式将所有的依赖打包。

这句话的每一个单词我都不懂，并且我挣扎着想要搞明白这玩意儿对开发者有什么用。

我写这篇文章的目的就是想要站在历史的角度解释JavaScript工具是如何发展为如今在2017年的样子。我会从构建一个示例网站开始，这个网站就像过去人们写的那样--不使用任何工具，只有HTML和JavaScript。接着我会逐步的介绍不同的工具，看看它们是为了解决什么样的问题而出现。在这个历史背景下，你会更好地学习并适应未来不断变化的JavaScript格局。让我们开始这段旅程吧。

## “老派”的使用JavaScript方式

首先我们从仅使用HTML和JavaScript构建的“老派”网站开始，这其中涉及到的操作有手动下载文件并链接到文件中。如下展示的是一个简单的index.html文件，引入了一个JavaScript文件：

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>JavaScript Example</title>
  <script src="index.js"></script>
</head>
<body>
  <h1>Hello from HTML!</h1>
</body>
</html>
```

`<script src =“index.js”></ script>`这一行引用了当前目录下名为index.js的独立JavaScript文件：

```JavaScript
// index.js
console.log("Hello from JavaScript!");
```

这就是你做一个网站的全部！现在让我们假设你想添加一个其他开发者写的类似于moment.js的库(这是一个可以帮助用户格式化日期的库)。例如，你可以使用库中的moment函数，如下所示：

```JavaScript
moment().startOf('day').fromNow();       
// 20 hours ago
```

但是前提是你必须在页面中引入moment.js！在moment.js的官网上，您会看到如下描述：

![](https://cdn-images-1.medium.com/max/800/1*ef7OX37jr--Jc38ZxO97Iw.png)

嗯，右边的安装部分看起来要做很多事。不过目前我们还不需要管这些。我们将名为moment.min.js的文件下载到当前目录下，并将其引入到index.html中。

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Example</title>
  <link rel="stylesheet" href="index.css">
  <script src="moment.min.js"></script>
  <script src="index.js"></script>
</head>
<body>
  <h1>Hello from HTML!</h1>
</body>
</html>
```

请注意moment.min.js在index.js之前被引入，这样开发者就可以在index.js中使用moment函数，如下所示：

```JavaScript
// index.js
console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());
```

以上就是我们通过JS库来编写网站的方式！这种方式的好处在于它十分简单，便于理解；坏处则是，每次都要查找下载最新的代码库，十分恼人。

## 使用JavaScript包管理器(npm)

从2010年开始，出现了几个相互竞争的JavaScript包管理器，用来自动化从中央仓库中下载并升级库的过程。在2013年，[Bower](https://bower.io/)是当时最流行的一款工具，但它最终在2015年左右被[npm](https://www.npmjs.com/)取代。(值得注意的是，从2016年下半年开始，[yarn](https://yarnpkg.com/en/)作为npm的替代品，已经引起了很大的注意，但它的底层仍然基于npm。)

npm最初是一个专为node.js(运行在服务端的JavaScript运行时)设计的包管理器，并非为前端准备。所以对于前端开发者来说，npm是一个奇怪的选择，因为他们实际上希望的是一个用于管理在浏览器中运行的库的包管理器。

> 注意： 
使用包管理器一般会涉及到使用命令行，在过去，前端开发者并不需要掌握它。但如果你之前没有接触过这方面的知识，你可以通过阅读[本教程](https://www.learnenough.com/command-line-tutorial)以获得一个良好的开端。了解如何使用命令行是现代JavaScript的重要组成部分(同时为向其他领域发展打开了一扇门)

让我们看看是如何使用npm自动安装moment.js包而非手动下载。如果你安装了node.js，那么你已经安装了npm，这意味你可以通过命令行导航到包含index.html的文件夹，然后输入：

> $ npm init

命令行会提示你几个问题(一般默认“回车”就行)，接着生成一个名为package.json的新文件。这是npm用于保存所有项目信息的配置文件。默认情况下，package.json的内容应该如下所示：

```json
{
  "name": "your-project-name",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}
```

为了安装moment.js包，我们可以通过以下命令来进行：

>$ npm install moment --save

这条命令执行两个步骤。首先，将moment.js包中的所有代码下载到名为node_modules的文件夹中。其次，它会自动修改package.json文件以使moment.js作为依赖项进行跟踪。

```json
{
  "name": "modern-javascript-example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "moment": "^2.19.1"
  }
}
```

在与他人共用一个项目时，共享package.json文件比共享node\_moudles文件夹更有优势。因为node\_modules内的文件可能会很大。而通过共享package.json的方式，其他开发者就可以使用命令`npm install`自动安装所需的包。

现在我们不再需要从网上手动下载moment.js了，而是使用npm自动下载和更新它。打开node\_modules文件夹，可以看到node\_modules/moment/min目录中的moment.min.js。这意味着我们可以将moment.min.js的npm下载版本引入到index.html中，如下所示：

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>JavaScript Example</title>
  <script src="node_modules/moment/min/moment.min.js"></script>
  <script src="index.js"></script>
</head>
<body>
  <h1>Hello from HTML!</h1>
</body>
</html>
```

这么做的好处在于，通过命令行，就可以使用npm下载和更新包。然而还是有缺点，我们仍然需要在node_modules文件夹中定位每个包的位置，并手动将其引入到对应的HTML文件中。这么操作十分不方便，接下来让我们看看这个过程是如何自动化的。

## 使用JavaScript模块绑定器（webpack）

大多数编程语言提供了将代码从一个文件导入到另一个文件的方式。JavaScript起初并不具有这个特性，因为JavaScript只能在浏览器中运行，无法访问系统的文件系统（出于安全因素）。因此，在很长的一段时间内，为了在多个文件中组织JavaScript代码，需要使用全局共享变量的方式加载每个文件。

就像上面的moment.js例子——整个moment.min.js文件被引入到HTML文件中，它定义了一个全局变量，在它后面引入的文件都可以访问它(不管是否需要)。

在2009年，CommonJS项目启动，旨在浏览器之外为JavaScript指定一个生态系统。CommonJS的一个重要组成部分就是它的模块规范，它允许JavaScript像大多数编程语言一样实现跨文件导入和导出代码，而不需要使用全局变量。CommonJS规范最着名的实现便是node.js.

![](https://cdn-images-1.medium.com/max/800/1*xeF1flp1zDLLJ4j7rDQ6-Q.png)

如前所述，node.js是在服务端上运行的JavaScript运行时。下面的例子展示了如何使用node.js模块。不像在script标签中将moment.min.js全部载入，开发者可以直接在js文件中导入moment：

```JavaScript
// index.js
var moment = require('moment');
console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());
```

这就是模块如何在node.js中加载的方式。因为node.js本来就是一个可以访问计算机文件系统的服务器端语言，所以模块系统能在其中很好的运行。Node.js知道每个npm模块的路径，所以不需要写出require('./node_modules/moment/min/moment.min.js)的代码，而只需要写require('moment')就行了。

require模式在node.js中可以顺利运行，但是如果开发者试图在浏览器中运行上面的代码，则会得到require字段没有定义的错误。因为浏览器无法访问文件系统，这意味着以这种方式加载模块非常棘手——加载文件必须动态地执行，要么是同步（这会减慢运行速度），要么是异步执行（可能会产生时间问题）。

这种时候模块绑定器就粉墨登场了。JavaScript模块绑定器通过构建步骤（可以访问文件系统）最终创建一个浏览器兼容的输出（不需要访问文件系统）。在这种情况下，我们需要一个模块打包器来查找所有require语句（这是不合法的浏览器JS语法），并将其替换为每个所需文件的实际内容。得到的最终结果是一个捆绑的JavaScript文件（没有require语句）！

原来最受欢迎的模块打包程序是2011年发布的Browserify，它率先在前端使用了node.js风格的require语句（这实质上是使npm成为通用的前端程序包管理器的原因）。大约在2015年，webpack最终成为了使用最为广泛的模块打包程序（受React前端框架的普及推动，其充分利用了webpack的各种功能）。

让我们看看一个使用webpack打包满足上诉要求并能够在浏览器中工作的例子。第一步就是将webpack安装到项目中。Webpack本身就是一个npm包，所以我们可以从命令行安装它：

>$ npm install webpack --save-dev

请注意--save-dev参数，它的作用是将webpack保存为开发依赖项，代表webpack是在开发环境中需要的，但不在生产服务器上的软件包。更新后的package.json文件如下：

```json
{
  "name": "modern-javascript-example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "moment": "^2.19.1"
  },
  "devDependencies": {
    "webpack": "^3.7.1"
  }
}
```

现在我们将webpack安装为node_modules文件夹中的其中一个包。可以通过命令行使用webpack，如下所示：

>$ ./node_modules/.bin/webpack index.js bundle.js

此命令会运行安装在node_modules文件夹中的webpack工具，从index.js文件开始，查找任何require语句，并用适当的代码替换它们以创建一个名为bundle.js的单个输出文件。 这意味着我们不再使用浏览器中的index.js，因为它包含无效的require语句。相反，在浏览器中使用打包好的bundle.js文件，引入到index.html文件的形式如下：

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>JavaScript Example</title>
  <script src="bundle.js"></script>
</head>
<body>
  <h1>Hello from HTML!</h1>
</body>
</html>
```

刷新浏览器后，会看到之前的一切照旧工作。

需要注意每次更改index.js时，都要运行webpack命令。这一步单调乏味，尤其当我们使用webpack的更高级的功能（比如生成源代码映射来帮助调试源代码中的原始代码）会变得更乏味。 Webpack可以从名为webpack.config.js的项目的根目录中的配置文件中读取选项，在我们的例子中，这个选项如下所示：

```JavaScript
// webpack.config.js
module.exports = {
  entry: './index.js',
  output: {
    filename: 'bundle.js'
  }
};
```

每次当我们改变index.js文件时，我们都可以使用下面的命令运行webpack：

>$ ./node_modules/.bin/webpack

不需要指定index.js和bundle.js选项，因为webpack正在从webpack.config.js文件加载这些选项。这种方式稍微好那么一点，但是对于每个代码更改，输入命令行仍然很乏味，在后面我会使这个步骤更简单一点。

总的来说，虽然看起来做的并不多，但是这个工作流程有一些巨大的优势。我们不再通过全局变量加载外部脚本。任何新的JavaScript代码库都将在JavaScript中使用require语句添加，而不是在HTML中添加新的`<script>`标签。同时一个单独的bundle.js文件意味着更好的性能。现在，我们添加了构建步骤，还有一些其他强大的功能可以添加到我们的开发工作流程中！

## 新语言功能的编译代码（babel）

编译代码意味着将一种语言的代码转换为另一种相似语言的代码。这是前端开发的一个重要组成部分——由于浏览器添加新功能的速度很慢，那些具有实验性功能的新语言需要被编译成浏览器兼容的语言。

对于CSS来说，有Sass，Less和Stylus等等。对JavaScript来说，以前最受欢迎的转换器是CoffeeScript（2010年左右发布），而现在大多数人使用babel或TypeScript。 CoffeeScript是一种专注于通过显著改变语言来改进JavaScript的语言包括可选的括号，重要的空白等等。Babel不是一种新的语言，而是一种将所有浏览器（ES2015及更高版本）尚未提供的功能转换为下一代JavaScript的转译程序以得到更老更兼容的JavaScript（ES5）代码。Typescript是一种与下一代JavaScript基本相同的语言，同时也增加了可选的静态类型。许多人选择使用babel，因为它最接近vanilla JavaScript。

让我们看看如何在现有的webpack构建步骤中使用babel的例子。 首先，我们将从命令行安装babel（这是一个npm包）到项目中：

>$ npm install babel-core babel-preset-env babel-loader --save-dev

注意，我们正在安装3个独立的软件包作为开发依赖关系——babel-core是babel的主要部分，babel-preset-env是一个预定义哪些新的JavaScript功能来进行转换的包，而babel-loader是一个使babel与webpack一起工作的包。通过编辑webpack.config.js文件来配置webpack以使用babel-loader，如下所示：

```JavaScript
// webpack.config.js
module.exports = {
  entry: './index.js',
  output: {
    filename: 'bundle.js'
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['env']
          }
        }
      }
    ]
  }
};
```

这个语法可能会令人困惑（幸运的是，这不是我们会经常编辑的东西）。本质上，我们告诉webpack查找任何.js文件（不包括node_modules文件夹中的文件），并使用babel-loader与babel-preset-env来应用babel转换。你可以在(这里)[https://webpack.js.org/configuration/]阅读关于webpack配置语法的更多信息。

现在，一切都准备好了，我们可以开始在JavaScript中编写ES2015功能了！以下是index.js文件中ES2015模板字符串的示例：

```javascript
// index.js
var moment = require('moment');
console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());
var name = "Bob", time = "today";
console.log(`Hello ${name}, how are you ${time}?`);
```

我们也可以使用ES2015导入语句而不是require来加载模块，在很多代码库中都会看到的这种语法：

```javascript
// index.js
import moment from 'moment';
console.log("Hello from JavaScript!");
console.log(moment().startOf('day').fromNow());
var name = "Bob", time = "today";
console.log(`Hello ${name}, how are you ${time}?`);
```

在这个例子中，导入语法与require语法没有太大的区别，但是对于更高级的情况，导入具有额外的灵活性。由于我们改变了index.js，我们需要在命令行中再次运行webpack：

>$ ./node_modules/.bin/webpack

现在在浏览器中刷新index.html。在撰写本文时，大多数现代浏览器都支持所有ES2015功能，因此很难判断babel是否能够完成其工作。你可以在像IE9这样的老浏览器中测试它，或者你可以在bundle.js中搜索转译后的代码行：

```JavaScript
// bundle.js
// ...
console.log('Hello ' + name + ', how are you ' + time + '?');
// ...
```

在这里你可以看到babel将ES2015模板字符串转换成常规的JavaScript字符串连接，以保持浏览器的兼容性。虽然这个例子可能不是太令人兴奋，但是编译代码的能力是非常强大的。 还有一些令人兴奋的语言功能，如async/await，开发者现在就可以使用以编写更好的JavaScript代码。虽然转译有时繁琐和痛苦，但是因为人们是在今天测试以后的特征，所以在过去几年里，JavaScript语言取得了巨大的改进。

整个工程差不多已经完成，但在我们的工作流程中仍然有一些小缺陷。如果我们关心性能，我们还需要缩小包文件，这应该很容易，因为我们已经有了一个构建步骤。当我们改变JavaScript文件时，我们还需要重新运行webpack命令。所以接下来我们看到的是一些便利的工具来解决这些问题。

## 使用任务运行器（npm脚本）

既然我们已经通过构建步骤来处理JavaScript模块，那么任务运行器作为一个工具的意义就在于它能够自动化构建工程中的不同任务。对于前端开发来说，这些任务包括代码精简，图片优化，运行测试等等。

2013年的时候，Grunt是最受欢迎的前端构建工具，不久就被Gulp取代。两者都依赖于包含其他命令行工具的插件。目前最流行的选择是使用内置在npm包管理器本身的脚本功能，它不使用插件，而是直接与其他命令行工具一起工作。

为了让webpack更好用，我们还要再写一些npm脚本。只需要简单地更改package.json文件，如下所示：

```JavaScript
{
  "name": "modern-javascript-example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack --progress -p",
    "watch": "webpack --progress --watch"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "moment": "^2.19.1"
  },
  "devDependencies": {
    "babel-core": "^6.26.0",
    "babel-loader": "^7.1.2",
    "babel-preset-env": "^1.6.1",
    "webpack": "^3.7.1"
  }
}
```

在这里，我们添加了两个新的脚本，分别是构建和监视。要运行构建脚本，可以在命令行中输入：

>$ npm run build

运行webpack（使用我们之前配置的webpack.config.js）--progress选项来显示百分比进度，-p选项来最小化生产代码。运行监视脚本：

>$ npm run watch

通过--watch选项从而实现每次JavaScript文件更改时都会自动重新运行webpack，这对开发很有帮助。

请注意，package.json中的脚本可以运行webpack，而无需指定完整路径./node_modules/.bin/webpack，因为node.js知道每个npm模块路径的位置。通过安装webpack-dev-server，我们可以使事情变得更加简单，这是一个单独的工具，它提供了一个简单的Web服务器，可以实时重新加载。要将其作为开发依赖项安装，请输入以下命令：

>$ npm install webpack-dev-server --save-dev 

然后添加一个npm脚本到package.json：

```javascript
{
  "name": "modern-javascript-example",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "webpack --progress -p",
    "watch": "webpack --progress --watch",
    "server": "webpack-dev-server --open"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "moment": "^2.19.1"
  },
  "devDependencies": {
    "babel-core": "^6.26.0",
    "babel-loader": "^7.1.2",
    "babel-preset-env": "^1.6.1",
    "webpack": "^3.7.1"
  }
}
```

现在，你可以运行以下命令来启动开发者服务器：

>$ npm run server

这会自动打开浏览器中的index.html网站，地址为localhost：8080（默认）。每当index.js中的代码发生改变，webpack-dev-server都会重新打包JavaScript并自动刷新浏览器。这一特性非常节省时间，因为它使开发者可以保持对代码的关注，而不必不断地在代码和浏览器之间切换以查看新的更改。

这只是表面上的，webpack和webpack-dev-server（你可以在[这里](https://webpack.js.org/guides/development/)了解到）有更多的配置。当然，你也可以编写npm脚本来执行其他任务，例如将Sass转换为CSS，压缩图像，运行测试等任何有命令行工具的操作。

## 总结

以上便是现代JavaScript的简史。我们从简单的HTML和JS到使用包管理器来自动下载第三方包，创建单个脚本文件的模块打包器，使用未来JavaScript特性的转译器，以及使生成过程的不同部分自动化的任务运行器。对于初学者来说，这里面有很多不一样的东西。Web开发曾经是新手编程的一个很好的切入点，因为它很容易就跑起来; 但是由于工具迅速改变的趋势，这个过程开始变得艰巨。

尽管如此，它并不像看起来那么糟糕。事情正在稳定下来，特别是通过采用node生态系统作为与前端合作的可行方式。使用npm作为包管理器，导入模块使用node require或者import statement，结合用于运行任务的npm脚本。与一两年前相比，这是一个大大简化的工作流程！

对于初学者和经验丰富的开发人员来说，更好的是现在的框架通常都带有一些工具，以使这个过程更容易上手。Ember拥有ember-cli，其对Angular的angular-cli，React的create-react-app，Vue的vue-cli等等都有很大的影响力。所有这些工具都会建立一个包含你所需的一切的项目——你需要做的只是开始写代码。然而，这些工具并不是魔术，它们只是将所有东西都设置成一致的工作方式——你可能经常会遇到需要使用webpack，babel等进行额外配置的地方。所以，理解每一处干了什么的还是非常重要的，正如我在本文中介绍的那样。

现代JavaScript肯定会令人沮丧，随着它不断改变和快速发展。但是，即使有时似乎重新发明了轮子，JavaScript的快速发展也推动了诸如热重载，实时检测和时光旅行调试等创新。 对于开发者来说，这是激动人心的时刻，我希望我提供的信息能够指引你之后的旅程！

by (kaola-zhangxueai@corp.netease.com)