---
title: svg图标在种草社区项目里的实践
date: 2017-07-29
author: luzhongfang
---
> 前端仔对svg这个词应该都不陌生，可缩放的矢量图形，意味着清晰、无损，想来该是越来越高清的手机端的宠儿，然而实际大公司里用的并不多，知乎上也有关于这个问题的[讨论](https://www.zhihu.com/question/26865508)。即便如此，svg在业界还是备受关注的，也有很多人对其看好，觉得是未来趋势。于是，在种草的最新一期需求里，小试了一把，来谈谈使用过程及感受。

先扔个扫盲贴，[SVG Sprite 使用简介](https://github.com/kaola-fed/blog/issues/36)一文类比了icon font，清晰的介绍了svg图标的使用方式。

### 项目中实践过程

- #### 收集图标

在业务功能都码的差不多了，开始想捣腾下svg了，就从设计师那里，要到了本项目的一些svg图标文件，将各个文件换成了英文名后放到了项目的static目录下。


- #### 拼合图标

查阅了一些文档，发现了一个可以[将svg图标进行拼合的工具](https://github.com/Hiswe/gulp-svg-symbols)，有点sprite精灵图的意思。于是在项目中安装了下

```
npm i gulp-svg-symbols --save-dev
```
注：使用此工具，需要结合gulp构建工具，确保项目中安装了gulp插件

然后设置gulp配置文件，gulpfile.js如下：

```
'use strict';

const {resolve, join} = require('path');

var gulp       = require('gulp');
var symbols = require('gulp-svg-symbols');
var wapPath = resolve(__dirname, 'src', 'main', 'resources', 'wap')
// svg 目录源
var svgPath = resolve(wapPath, 'static', 'svg')
// 拼合后的svg输出目录
var distPath = resolve(wapPath, 'dist', 'm', 'svg')


gulp.task('svgsprites', function () {
  return gulp.src(svgPath + '/*.svg')
    .pipe(symbols({'svgClassname':'svgicon'}))
    .pipe(gulp.dest(distPath));
});


```
并在package.json中，配置了此任务：

```
"scripts": {
    "svgsprites": "gulp svgsprites"
}
```

准备工作差不多了，跑下命令看看结果：
```
npm run svgsprites
```

果然，在输出目录下生成了一个`svg-symbols.css`文件和一个`svg-symbols.svg`文件，查看了下此svg文件，确认所用的几个图标代码都在此文件里了，就迫不及待的想在业务代码里使用起来：

```
<svg class="svg-arrow">
  <use xlink:href="/dist/m/svg/svg-symbols.svg#arrow-rt"></use>
</svg>

```

然后并没有在页面上如期看到图标，于是我又细看了下工具生成的这个svg，类比了其他能正常使用的文件，发现文件头缺少了XLink的定义：
```
xmlns:xlink="http://www.w3.org/1999/xlink"
```
看来此工具"年久失修"了，于是只好自己[fork了一份](https://github.com/lzf0402/gulp-svg-symbols)，改了此工具导出svg所用的模板（/templates/svg-symbols.svg），加上了这串定义，同时在此模板中加了use标签来显示各个图标：

```
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"<% if(svgClassname) {%> class="<%= svgClassname %>"<% } %>><% if(defs) {%>
  <defs>
  	<%= defs %>
  </defs><% } %>
  <% _.forEach( icons, function( icon ){ %>
  <symbol id="<%= icon.id %>" viewBox="<%= icon.svg.viewBox %>"<% if (icon.svg.originalAttributes.preserveAspectRatio) {%> preserveAspectRatio="<%= icon.svg.originalAttributes.preserveAspectRatio %>" <% }%>><% if (icon.title) {%>
      <title><%= icon.title %></title><% }%>
      <%= icon.svg.content %>
  </symbol><%
  }); %>
  <!-- 添加use标签 显示symbol中定义的图标 -->
  <% _.forEach( icons, function( icon, idx ){ %>
  <use xlink:href="#<%= icon.id %>" width="20" height="20" x="0" y="<%= idx*20 %>" fill="#666666"></use><%
  }); %>

</svg>

```
再把这个修改后的工具，发了个npm包—`gulp-svg-combine`，重新更新工程里的包名和gulp配置，再进行拼合，这次终于可以正常使用了。

### 使用

现在拼合后的svg文件已有，只需要在各个使用到的地方像图片一样的引用就可以了：


```
// 可以把所有图标的样式统一到单个文件里
.svg-arrow{
    width: 16px;
    height: 16px;
    fill: #666;
    margin-left: 3px;
    position: relative;
    top: 3px;
}

// ftl中可以直接标签使用
<svg class="svg-arrow">
  <use xlink:href="/dist/m/svg/svg-symbols.svg#arrow-rt"></use>
</svg>

// regular中使用，参考：SVG Sprite 使用简介
// 定义了r-xlink指令
<svg class="svg-zan">
  <use r-xlink:href="/dist/m/svg/svg-symbols.svg#zan"></use>
</svg>

// 指令动态设置标签的xlink定义
List.directive('r-xlink:href', function(elem, val) {
    if (val&& val.type === 'expression') {
        this.$watch(val, function(newVal) {
            elem.setAttributeNS('http://www.w3.org/1999/xlink', 'href', newVal);
        });
    } else {
        elem.setAttributeNS('http://www.w3.org/1999/xlink', 'href', val);
    }
});

```

至此，算是将svg图标在项目中，实际用起来了，从表现上来说，细看确实会发现对比图片图标或iconfont图标，都要清晰，相信放大之后清晰度的差异更明显，试了各大浏览器和微信中，均表现正常。


### 使用感受

下面谈谈一路折腾下来的感受，以及项目中实际使用的一点想法。


对于[Inline SVG vs Icon Fonts](https://css-tricks.com/icon-fonts-vs-svg/)一文中类比了svg和icon的优劣。
- svg 缩放无损不失真，毋庸置疑，几乎碾压png、iconfont，这也是TA最大的亮点。

- css属性方面，iconfont能做到的，svg基本也能做到，同时Ta还能控制图标的部分内容，但实际这种场景应该不多，如果真需要工程师去控制一个小图标里的部分样式，也是工作效率上的一个考验了。

- 定位方便，虽然svg标签就是其本身的大小，但实际使用中的感受是，图标如果要和文字对其的话，也需要相应的样式控制。

- 对于iconfont各种原因可能造成字体显示失败的问题，对用户来说确实是不友好的体验，而svg全赖浏览器支持，不支持无非也就不显示内容不会出现乱码之类的情况。目前IE8-以及Android 2.3默认浏览器是不支持SVG的，但可以做一些[fallback](一些SVG向下兼容优雅降级技术)。

- svg在语义和可访问性方面，确实比iconfont更胜一筹，毕竟svg标签代表着图片，但事实上很多小图标的语义我们
并不关心，也不需要浏览器或者屏幕阅读器去过分解读。

- 性能方面，svg渲染的成本比图片和字体都要来得高。

最后要谈谈使用体验了。

使用svg图标的时候，首先需要通过工具去合并。代码层面，
即需要在html中写结构（代码量上会增多），又需要通过选择器来给图标设置一些css属性。相对于使用iconfont，并无效率提升。

设计师直接导出的svg图片，如果不做优化，通过工具合并后的文件会比较大，代码量惊人。而且如果导出的图标被设定死了颜色（fill属性），不做处理的话就无法复用了。当然可以去要求设计师导出的时候遵循一定的标准，但一切人为的操作都是很难保证的。 作为前端，如果花太多精力在处理图标上，实在也不是好的体验。

svg虽然有向下兼容的降级方案，未免麻烦，PC端还是老实用iconfont吧。

以上是自己的一点感想，欢迎拍砖。

---
### 参考文档
- [从icon fonts到SVG icons](http://leungwensen.github.io/blog/2016/from-icon-fonts-to-svg-icons.html)
- [Inline SVG vs Icon Fonts](https://css-tricks.com/icon-fonts-vs-svg/)