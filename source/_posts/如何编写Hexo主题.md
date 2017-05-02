---
title: 如何编写Hexo主题
date: 2017-05-02
---

最开始折腾Hexo的时候感觉这东西很神奇，通过他和github搭配就能生成免费的静态博客，而且还有丰富的主题可以选择，当我刚入Hexo的时候默认主题是`landscape`，后来又使用过`NexT`，是一款很漂亮的主题，但是除此之外，还有很多好看的主题，我很好奇这些主题都是怎么写出来的，于是乎就仿照`landscape`主题开始研究，写自己的主题，也就是[我自己的博客](http://www.showonne.com/)正在用的主题，项目地址在[这里](https://github.com/showonne/hexo_showonne)。

<!-- more -->

完成一个Hexo的主题其实很简单，和写静态页面差不多，只是内容部分通过Hexo的变量去获取，而且Hexo还内置了一些辅助函数帮你快速方便地完成繁琐的处理。

<!-- more -->

起步
---
在写代码之前要先把项目结构搭建好，一个Hexo主题的项目名就是主题名字本身，项目内的目录结构如下: (生成树形图是用的`tree`， mac上直接`brew install tree`就可以了，以前不写都不知道囧)
 
    .
    ├── _config.yml   //记录主题配置信息
    ├── layout        //存放布局模板文件
    │   └── _partial  //布局文件中可共用的模板
    └── source        //静态资源文件夹
        ├── css
        ├── fonts
        ├── js
        └── sass

项目结构搞好就可以开始写代码了!因为当初我是仿`landscape`写的，而且`ejs`也是我之前看nodejs时就接触过的，因此就直接用ejs写模板文件了，样式使用了`sass (scss`。

布局
---
#### 编写布局文件(layout.ejs)
模板文件在`layout`文件夹下，文件名对应Hexo中的模板名，有`index`,`post`,`page`,`archive`,`category`,`tag`几种，对于普通的`header + content + footer`的页面结构，`header`和`footer`往往是可以复用的，因此我们可以使用`layout.ejs`进行布局，动态的内容使用`body`变量去动态渲染，所以我的`layout.ejs`大概长这样:

    <!doctype html>
	<html lang="en">
	<head>
	    <meta charset="UTF-8">
	    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no"/>
	    <title><%= config.title %></title>
	    <%- css('css/style') %>
	</head>
	<body>
	    <%- partial('_partial/header') %>
	    <div class="main">
	        <%- body %>
	    </div>
	    <%- partial('_partial/footer') %>
	    <%- js('js/index.js') %>
	</body>
	</html>

`partial`,`js`和`css`是Hexo提供的辅助函数，后面再说。

#### 其他模板文件
每一个模板文件对应的是一种布局，当你使用`hexo new <title>`的时候，其实忽略了一个参数，完整的命令是`hexo new [layout] <title>`，这个`layout`就决定了文章使用何种方式布局，比如创建一个自己简介的About页面，`hexo new page "about"`其实就是使用了page布局。每种布局对应到我们的模板文件上就是`index.ejs(首页)`,`post.ejs(文章)`,`archive.ejs(归档)`,`tag.ejs(标签归档)`,`page.ejs(分页)`。

如果更直观一点，url和模板的对应关系是这样的:


Url                  | Description  | Layout      
 -------------------  |-----------| ----------|
 /                    | 首页          | index.ejs   
 /yyyy/mm/dd/:title/  | 文章          | post.ejs    
 /archives/           | 归档          | archive.ejs 
 /tags/:tagname/      | 某个标签的归档  | tag.ejs     
 /:else/              | 其他          | page.ejs    
    

#### index.ejs
首页一般是一些博文的摘要和一个分页器，通过Hexo的`page`变量拿到页面的数据渲染即可，这里我们不直接在`index.ejs`中写HTML结构，新建一个`_partial/article.ejs`，将文章数据传给子模板渲染，然后再额外传入一个参数`{index: true}`，对后面的`post.ejs`和`page.ejs`加以区分，让子模板能正确渲染。最后，index.ejs大致是这样的:

    //index.ejs
    <% page.posts.each(function(post, index){ %>
		<%- partial('_partial/article', {index: true, post: post}) %>
	<% }) %>
	<div class="pagination">
		<%- paginator({ total: Math.ceil(site.posts.length / config.per_page)}) %>
	</div>

#### post.ejs
文章模板和首页差不多，只是对应的是一篇具体的文章，所以就把文章传入，再额外传入`{index: false}`告诉子模板不要按首页的方式去渲染就好了。就一行代码(因为都在子模板里 XD

    //post.ejs
    <%- partial('_partial/article', {index: false, post: page}) %>

#### page.ejs
我个人对Page模板其实是有点懵逼的，在我自己的实践中是添加`about`(`hexo new page "about"`)页面后，访问`/about`会走分页布局，实际上这个页面对应的内容是`/source/about`里的`index.md`，也相当于对文章的渲染，因此我把Page模板也写成了和文章模板一样:

    //page.ejs
    <%- partial('_partial/article', {index: false, post: page}) %>

#### _partial/article.ejs
前面一共有三处共用了article模板，另外page和post的一样的，所以实际上只有两种情况:主页(`index: true`)和非主页(`index: false`)。对应的`_partial/article.ejs`里只要判断这个值就可以正确渲染了，基本结构如下：

    //_partial/article.ejs
    <% if(index){ %>
		//index logic...
	<% }else{ %>
		//post or page logic...
	<% } %>

#### tag.ejs
标签归档页内容很少，直接用Hexo的辅助函数`list_tags`生成一个标签的列表就ok了:

    //tag.ejs
    <%- list_tags() %>

归档页模板和首页差不多，归档页只需要展示文章标题和最后的分页器就好:

    //archive.ejs
    <div class="archive">
      <% var lastyear; %>
      <% page.posts.each(function(post){ %>
        <% var year = post.date.year() %>
        <% if(lastyear !== year){ %>
          <h4 class="year"><%= year %></h4>
          <% lastyear = year %>
        <% } %>
        <div class="archive_item">
          <a class="title" href="<%- url_for(post.path) %>"><%= post.title %></a>
          <span class="date"><%= post.date.format('YYYY-MM-DD') %></span>
        </div>
      <% }) %>
      <div class="pagination">
        <%- paginator({ total: Math.ceil(site.posts.length / config.per_page)}) %>
      </div>
    </div>
    
至此，模板文件就写好了，对于`category `模板就放弃了，感觉比较鸡肋。。。

变量
---
其实在模板文件中我们已经看到了`page.post`,`site.posts.length`,`config.per_page`等等，页面的内容就是根据这些变量获取的，由Hexo提供，拿来直接用，Hexo提供了很多变量，但不是都很常用，一般就用到以下变量:
- `site`: 对应整个网站的变量，一般会用到`site.posts.length`制作分页器
- `page`: 对应当前页面的信息，例如我在`index.ejs`中使用`page.posts`获取了当前页面的所有文章而不是使用`site.posts`。
- `config`: 博客的配置信息，博客根目录下的`_config.yml`。
- `theme`: 主题的配置信息，对于主题根目录下的`_config.yml`。

辅助函数(Helper)
---
制作一个分页器，我们需要知道文章的总数和每页展示的文章数，然后通过循环生成每个link标签，还要根据当前页面判断link标签的active状态，但是在Hexo中这些都不用我们自己来做了!Hexo提供了`paginator`这一辅助函数帮助我们生成分页器，只需要将文章总数`site.posts.length`和每页文章数`config.per_page`传入就可以生成了。

#### 其他的Helper:

- `list_tags([options])`: 快速生成标签列表
- `js(path/to/js)`, `css(path/to/css)` 用来载入静态资源，path可以是字符串或数组(载入多个资源)，默认会去`source`文件夹下去找。
- `partial(path/to/partial)` 引用字模板，默认会去`layout`文件夹下找。


样式
---
知道了Hexo的渲染方式，我们就可以使用HTML标签+CSS样式个性化我们的主题了，推荐大家使用CSS预处理语言的一种来写样式，这样就可以通过预处理语言自身的特点让样式更灵活。


其他
---

#### 添加对多说和Disqus的支持
评论是很常用的功能，不如就直接在我们的主题里支持了，然后通过配置变量决定是否开启，评论区跟在文章内容下面，对于这种三方的代码块，最好也以`partial`的方式提取出来，方便移除或是替换。
    
    //_partial/article.ejs
    <section class='post-content'>
        <%- post.content %>
    </section>
    //评论部分，post.comments判断是否开启评论，config.duoshuo_shortname
    和config.disqus_shortname来判断启用那种评论插件，这里优先判断了多说
    <% if(post.comments){ %>
        <section id="comments">
        <% if (config.duoshuo_shortname){ %>
                <%- partial('_partial/duoshuo') %>
            <% }else if(config.disqus_shortname){ %>
                <%- partial('_partial/disqus') %>
            <% } %>
        </section>
    <% } %>
    
再将多说和Disqus提供的js脚本代码放在`_partial/duoshuo.ejs`和`_partial/disqus.ejs`下就ok了~

#### 使用highlight.js提供代码高亮

highlight.js提供了多种语言的支持和多种皮肤，用法也很简单，载入文件后调用初始化方法，一切都帮你搞定，对于使用那种皮肤，喜好因人而异，我们干脆在主题的配置文件中做成配置项让用户自己选择:
    
    //showonne/_config.yml
    ...other configs
    
    # highlight.js
    highlight_theme: zenburn

对应的`layout.ejs`中:
    <link rel="stylesheet" href="//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.4.0/styles/<%= theme.highlight_theme %>.min.css">
    
样式文件通过CDN引入，因为不同皮肤对应不同的文件名，所以十分灵活。

最后
---
当初是对应着`landscape`照葫芦画瓢写的，最近回头来发现一些不合理的地方，所以就又改了改，也对应着写了这么一篇总结，接下来准备再把样式划分一下，对于颜色这类样式通过变量的方式提取出来，也变得可配置，能让主题更灵活一些。

#### 参考资源
[了解辅助函数](https://hexo.io/zh-cn/api/helper.html)
[模板](https://hexo.io/zh-cn/docs/templates.html)
[Hexo中的变量](https://hexo.io/zh-cn/docs/variables.html)
[Hexo主题列表](https://hexo.io/themes/)
[Hexo使用多说教程](http://dev.duoshuo.com/threads/541d3b2b40b5abcd2e4df0e9)
[How to use highlight.js](https://highlightjs.org/usage/)

