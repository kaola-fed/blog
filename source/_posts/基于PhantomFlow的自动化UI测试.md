---
title: 基于PhantomFlow的自动化UI测试
date: 2017-03-27
---

### 基于PhantomFlow的自动化UI测试

本文的目录结构：

1. 自动化测试的意义
2. 可测试方向分析
3. 竞品分析&技术选型
4. PhantomFlow介绍
5. 持续集成

<!-- more -->

#### 自动化测试的意义
 - 一个项目最终会经过快速迭代走向以维护为主的状态，在合理的时机以合理的方式引入自动化测试能有效减少人工维护成本。
 - 自动化的收益 = 迭代次数 * 全手动执行成本 – 首次自动化成本 – 维护次数 * 维护成本
 - 另一方面，当我们需要对代码进行重构或者完善，在修改结束时我们如何确定项目仅仅是被重构了，而不是被改写了？此时测试将是一根救命稻草，它是一个衡量标准，告诉开发人员这么做是否将改变结果。


#### 可测试方向分析
前端自动化测试的方向有：

 - 单元测试
 - UI回归测试
 - 功能测试
 - 性能测试

##### 单元测试

 - 在计算机编程中，单元测试（Unit Testing）又称为模块测试, 是针对程序模块（软件设计的最小单位）来进行正确性检验的测试工作。
 - 单元测试已经有非常完善的工具体系，借用2016 JavaScript 之星的图，常用的单元测试框架有：
 
 ![](http://haitao.nos.netease.com/dd76bbe4b7184124955a77ce4cb7ed97.png)

##### UI回归测试

UI回归测试通常采用的方法是像素对比：

 - 像素对比基本的思想认为，如果网站没有因为你的改动而界面错乱，那么在截图上测试页面应当跟正常页面保持一致。
 - 像素对比比较出名的工具是PhantomCSS，它结合了 CasperJS 截图和 ResembleJs 图像对比分析。从易用性和对比效果来说是很不错的。

###### 像素对比 - PhantomCSS

 初次运行的时候，会截图并作为baseline，后面再运行的时候，再生成截图，并与baseline比较，生成diff结果。

 ![](http://haitao.nos.netease.com/0395a077309043299724f6479a3b6ed5.png)

像素对比需要注意的事项：

 - 推荐对某些区域进行测试而不是整个页面。图像越大对比越慢。
 - 如果测试区域内有动态元素，可以通过选择器来隐藏。
 - 界面对比只是一个环节，需与其他测试相结合，合理结合才是关键。

##### 功能测试

 - 仅仅对界面进行测试是不够的，即使界面正确，功能不正确也是断然不能接受的。
 - 最直接的功能测试就是通过模拟用户操作流程来判断页面的展现是否符合预期。
 - 有时，我们需要浏览器处理网页，但并不需要浏览，比如生成网页的截图、抓取网页数据等操作。PhantomJS的功能，就是提供一个浏览器环境，你可以把它看作一个“虚拟浏览器”，除了不能浏览，其他与正常浏览器一样。它的内核是WebKit引擎，不提供图形界面，我们可以用它完成一些特殊的用途。

###### PhantomJS和CasperJS
 - CasperJS是对PhantomJS的封装，提供了更加易用的API, 增强了测试等方面的支持。
 - 如下图，很方便的实现了一个百度贴吧自动发帖的功能。

 ![](http://haitao.nos.netease.com/466dba6d10c34d37bfd27b7f2280e837.png)

##### 性能测试
 - 性能测试通常来测试网站的性能，如白屏时间、首屏时间等。
 - 通常的工具有：chrome devtool，PageSpeed等在线测试网站。

 考虑到我们主题是nek-ui组件库的测试，性能测试的部分，这里不做赘述。

#### 竞品分析&技术选型

我们的测试对象是NEK-UI组件库，这一部分分析了其他组件库的测试方法并选择了最终的测试方案。

##### RegualrUI测试方案分析：
RegularUI使用的测试方案是karma + mocha的黄金搭档

![](http://haitao.nos.netease.com/0f28ada2d3634f72a5fd727aea122ef6.png)

这种方式存在的问题：

 - 没有UI部分的测试，这也就是单元测试与UI测试的差别。
 - 虽然可以通过调用组件的某些方法，达到用户操作同样的效果，但是跟真实的用户操作还是有差别的。比如，这个时候，template的这个方法根本没绑定，或者传参错误，这种情况是覆盖不到的。

##### Ant-design测试方案分析：
 - Ant-design是蚂蚁金服的一套企业级的 UI 设计语言和 React 实现，目前是Github上一个很火的项目：
 ![](http://haitao.nos.netease.com/e0603d5c703c4d8581415543e981b67a.png)

 - Ant-design作为一个基于react的组件库，使用的测试框架是同样出自Facebook的Jest。

 - Ant-design使用的是Jest中称为snapshot testing的测试方案。

 - Jest的官方文档上介绍到，Jest的[Snapshot Testing](http://facebook.github.io/jest/docs/snapshot-testing.html#content)与典型的snapshot test不同，不是生成截图并比较图片的差异，而是直接输出React tree 的最终渲染dom结构。
 
 Snnpshot Testing介绍：

 ![](http://haitao.nos.netease.com/3f1fb57eee1c4983bc55ac824d41b9dd.png)

再来看看Ant-design中的实际使用：

![](http://haitao.nos.netease.com/2283561036b544248593ca0e3bab7fd9.png)

测试某个组件的时候，就会引入改组件文件夹里demo文件夹下的所有md文件，这个md文件是组件的各种示例，同时也用于ant-design的官方文档。然后，使用enzyme和enzyme-to-json提供的方法经过render->renderToJson->toMatchSnapshot, 第一次运行的时候会输出如下的.snap文件：

![](http://haitao.nos.netease.com/9117fbd4c0124353bf66534c266022cd.png)

这个文件要随着代码一起提交到仓库，下次运行测试的时候，就和这个.snap文件做比较。

当然仅仅测试dom结构不变是不够的，ant-design的测试里，还有模拟用户操作的测试。如下两个文件，demo.test.js是上面的snapshot部分，index.test.js是模拟用户操作部分。

![](http://haitao.nos.netease.com/9ce2990a7e864b2f97cbb1fd7c4e3c01.png)

Index.test.js里做了什么呢？

![](http://haitao.nos.netease.com/7b9c34694caa42f6afbd94497484ea92.png)

在组件上绑定事件方法，然后模拟事件，判断方法是否被调用。

这种方式存在的问题：

 - DOM结构不变并不完全等于样式不变。
 - 很多相关工具都是React专用。

分析完了2个组件库的测试方案，那么我们期望的测试方案应该包含什么呢？

 - 组件库同一般的纯JS库不用的地方，使得单纯的单元测试是不够用的，最好要包含UI测试的部分。
 - 有模拟用户操作的部分。
 - 能方便的管理test case。

 基于此，我们最终选择了PhantomFlow。


#### PhantomFlow介绍
 - PhantomFlow是基于决策树(decision tree)的ui test 框架，是对PhantomJS、CasperJS、PhantomCSS的包装。
 ![](http://haitao.nos.netease.com/0873824ae7e949c1a65a57d5be351055.png)

 - PhantomFlow假定如果页面正常，那么在相同的操作下，每次页面所展现的应该是一样的。基于这点，使用者只需要定义一系列的操作流程和决策分支，然后利用PhantomCSS进行截图和图像对比。最后将测试结果在一个可视化报表中展现出来。

这里采用倒序的方式先来看一下PhantomFlow生成的测试报告，再介绍具体的使用：

这是PhantomFlow的母公司Huddle在他们实际的业务中使用的报告截图：

![](http://haitao.nos.netease.com/a184a33735b1486cb3788f75b19b4759.png)

同时PhantomFlow也提供了单独查看某一个操作流的功能：

![](http://haitao.nos.netease.com/7b401b5c042b48f09ae1362982103a47.png)

图中的每一条线代表一个用户操作流。绿色的点表示截图对比通过，红色的点表示截图对比失败，灰色的点表示这仅仅是PhantomFlow流程中的一步，并没有真正的操作。
黄色的表示是一个操作，但是操作里面并没有进行截图。我们只要关心其中绿色的点和红色的点。

PhantomFlow是基于决策树的，那么什么是决策数呢？没必要吧它想的那么神秘，我们可以认为它就是普通的流程图。

![](http://haitao.nos.netease.com/de751a9169224fb9aea6a794ae44d174.png)

##### PhantomFlow方法介绍

 - flow (string, callback)：初始化一个test  suite，回调函数中可以包含step， chance 和 decision。

 - step (string, callback)：一个单独的步骤，回调函数中可以包含PhantomCSS的截图，CasperJs的操作事件和断言

 - decision (object)：定义一个用户的决定，参数是一个对象，key用来描述decision的名称，value是一个function，里面可以包含后续的decision， chance和step

 - chance (object)：功能上同decision一样，只是在语义上区分decision，用来描述不是用户主动的行为。

step对应决策树中的矩形，表示用户具体的某一个操作。decision和chance对应决策树中的菱形，表示用户的选择。

这是用PhantomFlow描述用户喝咖啡的一个场景：

![](http://haitao.nos.netease.com/bba6eb94ccb545bfbb62e8400c22cb2d.png)

##### PhantomFlow在NEK-UI组件测试中的使用

 - 以ui.select组件为例：

![](http://haitao.nos.netease.com/d741f773f53743029e5e7d197d071fce.png)

PhantomFlow提供了简单的方法来描述用户的操作流，具体的操作使用回调函数里的CasperJS来完成：

```javascropt
    function goToPage() {
        casper.thenOpen("http://localhost:9001/test/index.html", function() {
            this.echo('PageTitle: ' + this.getTitle());
            phantomCSS.turnOffAnimations();
        });
    }

    function injectModule(json) {
        casper.evaluate(function(json) {
            console.log(JSON.stringify(json));
            new NEKUI.UISelect(json).$inject('#module');
        }, json);
        casper.onConsoleMessage = function(msg) {
            console.log(msg);
        }
    }

    function goToModule() {
        casper.waitForSelector(
            '#module .u-select2',
            function success() {
                phantomCSS.screenshot('#module .u-select2');
                casper.test.pass('Should see the uiselect module' );
            },
            function timeout() {
                casper.test.fail('Should see the uiselect module');
            }
        )
    }

    function clickModule() {
        casper.click('#module .dropdown_hd');
        casper.waitForSelector(
            '#module .dropdown_bd',
            function success() {
                phantomCSS.screenshot('body');
                casper.test.pass('Should see the options of module');
            },
            function timeout() {
                casper.test.fail('Should see the options of module');
            }
        )
    }

    function selectAnOption(optionIndex) {
        casper.click('#module .m-listview li:nth-child(' + (optionIndex+1) + ')');
        phantomCSS.screenshot('body');
    }

```

###### 测试报告使用介绍：

![](http://haitao.nos.netease.com/080b87e7ede04c93af3faf2d3b72e7be.gif)


###### 运行测试的常用参数：

在npm test后带上如下参数即可

 - report：打开浏览器，生成测试报告。

 - debug：输出更多的log信息，强制切换到单线程运行。

 - earlyexit： 默认为false，设置为true的话，遇到第一个failure就会终止测试。

 - threads：设置多线程来运行测试，默认为4。


##### 常用CasperJS方法介绍

 - casper.thenOpen(String location[, mixed options]): 用来打开一个地址，当网页加载完成之后，执行一个方法。

 - casper.waitForSelector(String selector[, Function then, Function onTimeout, Number timeout])：等到DOM里有一个元素匹配选择器，可以传入成功的方法和失败的方法，和等待的毫秒数（默认5000）。

 - casper.click(String selector, [Number|String X, Number|String Y])：在匹配选择器的第一个元素上执行一次click

 - casper.mouseEvent(String type, String selector, [Number|String X, Number|String Y])：在匹配选择器的第一个元素上触发鼠标事件。支持的事件有：mouseup、mousedowm、click、dblclick、mousemove、mouseover、moustout、mouseenter、mouseleave and contextmenu

 - casper.getHTML([String selector, Boolean outer])：获取匹配选择器里的元素的内容。

 - casper.evaluate(Function fn[, arg1[, arg2[, …]]])：在打开的当前页面环境下执行方法。

 - casper.test.fail(String message)：添加一个fail test。

 - casper.test.pass(String message)：添加一个pass test。

 - casper.test.assertEquals(mixed testValue, mixed expected[, String message])：断言两个值严格相等。

##### 使用PhantomFlow要注意的地方

 - 数据确定性：同样的测试用例在组件上运行多次，产生的结果应该相同。如果测试方法里面包含有Date.now()这种“数据不确定”的因素，会导致每次运行测试，页面显示的都不相同，这个时候可以引入sinon，用它的stub来托管数据不确定的方法。
 - 适当的添加断言：截图测试的特性决定了baseline一定要正确。假如首次运行的时候截图就错误，后面的运行错误一样是不会报错的。因此需要添加一些dom取值断言。


#### 持续集成
 - 经常手动执行npm test？很麻烦有没有
 - 别人项目里的这两个徽章怎么来的？这两个徽章是项目可靠度的体现。

![](http://haitao.nos.netease.com/dbc2f843133d49b7824ee69536c5e10a.png)

 - 持续集成（Continuous integration，CI），一种软件工程流程，指工程师将自己对于软件的复本，每天集成数次到主干上。在测试驱动开发（TDD）的做法中，通常还会搭配自动单元测试。

 ![](http://haitao.nos.netease.com/5718899ab3e04d18a6a08cfc26ad6882.png)

##### Travis CI
Travis-ci是一款持续集成服务，它能够很好地与Github结合，每当代码更新时自动地触发集成过程。[Travis CI](https://travis-ci.org/)

打开Travis CI的官网，用Github账号登录。

![](http://haitao.nos.netease.com/f0ba429ecf8c4915baef067d9d39f5cc.png)

选择需要打开Travis CI服务的仓库：

![](http://haitao.nos.netease.com/fed147eab6cc4fe7ad309522173a6459.png)

开通了服务的仓库，每当有push代码的时候，Travis CI就会为我们执行相关的操作。这里可以查看运行的进度和结果等。

![](http://haitao.nos.netease.com/8e6ef08d9f74422f86ea3b573e03dbc4.png)

在Github提交记录里也会显示CI运行的结果。

![](http://haitao.nos.netease.com/2b2d8aae37a040cfbd16e9867f7bd3e3.png)

要告诉Tracvis执行什么，需要在我们的项目里添加一个.travis.yml文件，其最简单的配置如下：

![](http://haitao.nos.netease.com/e2d2b67dd16b48f8977d1d0c76366e22.png)

这里指定了CI运行的语言，语言版本，哪些分支，install执行npm install, script是具体的操作部分，这里让CI执行 npm test。

Travis CI在执行完之后，会将结果邮件通知给用户, 默认规则如下：

```text
By default, email notifications are sent to the committer and the commit author when they are members of the repository, 
that is they have

 - push or admin permissions for public repositories.
 - pull, push or admin permissions for private repositories.
Emails are sent when, on the given branch:

 - a build was just broken or still is broken.
 - a previously broken build was just fixed.

```

关于travis ci的生命周期等更多配置可以查阅[这里](https://docs.travis-ci.com/)


##### Coveralls 代码覆盖率托管平台
 - 代码覆盖率通常被用来衡量测试好换的指标。Coveralls就是将测试导出的覆盖率文件进行分析，以可视化的形式展现出来的一个工具。
 - 使用Coveralls的项目包括：React、Express、Gulp、Ant-design等等。

同样的使用Github账号登陆：

![](http://haitao.nos.netease.com/f3812e83530648c59b39b5e75878c534.png)

选择开启服务的仓库：

![](http://haitao.nos.netease.com/69efb9f82c6c4fc58f3c5c673eb7ccf0.png)

在项目的package.json文件script里添加一条coverage的命令， 即将istanbul等覆盖率工具生成的lcov文件给coveralls：

![](http://haitao.nos.netease.com/4fb61c8b6adb494287b40afbeee45975.png)

在travis.yml文件的after_script中运行npm run coverage，告诉CI服务器执行这条命令：

![](http://haitao.nos.netease.com/157c7103a9ec4d40bb268a2e5781436a.png)



#### 总结
 - 自动化测试不仅能有效的减少人工维护成本，同时为代码的维护迭代提供保障。

 - 前端自动化测试的方向有：单元测试、UI回归测试、功能测试、性能测试。

 - RegularUI采用karma+mocha的单元测试，ant-design使用Jest的snapshot测试与模拟用户的功能测试相结合的方式。

 - PhantomFlow是基于决策树的，对PhantomJS, CasperJS, PhantomCSS的包装。以简单的方式描述用户操作流。并配以CasperJS的页面操作，PhantomCSS的截图，达到非常好的自动化测试效果。

 - 测试时要保证数据的确定性和添加适当的断言。

 - CI是一种好的软件工程思想。Travis CI简单易用，解放了开发人员手动运行测试，非常值得在项目中引入。

