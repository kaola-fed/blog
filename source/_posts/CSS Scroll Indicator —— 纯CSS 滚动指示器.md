---
title: CSS Scroll Indicator —— 纯CSS 滚动指示器
date: 2017-05-31
---

# CSS Scroll Indicator —— 纯CSS 滚动指示器

前段时间拜读阮老师的 《ECMAScript 6 入门》 ，看到官网上每个章节的页面都有一个类似进度条的东西，出于好奇，上网查了一下，发现这个东西叫```Scroll Indicator```.

Scroll Indicator：滚动指示器。通俗来说，就是当前可视区域距离页面顶部的占比，效果如下：

![](http://olz3b8fm9.bkt.clouddn.com/17-5-31/83536113.jpg)

<!-- more -->

### JavaScript的做法

在还没有捞源码之前，自己先大致想了一下实现思路：

* 页面加载完成之后，获取到页面文档高度（DH）、当前可视区域高度（VH）、可视区域距离页面顶部的高度```scrollTop```（TH）；
* (DH - VH)就是需要滚动的值；
* 监听页面 ```scroll``` 事件，```(TH / (DH - VH)) * 100%```，便是当前的占比；
* 当然，需要考虑```节流```、  ```防抖```。

核心代码如下：

```
var dh = $(document).height();
var vh = $(window).height();
var sHeight = dh - vh;
$(window).scroll(function() {  
    var perc = $(window).scrollTop() / (dh - vh);
    $('.j-scroll-indicator').css({width: perc * 100 + '%'});
)};
```
这是实现之后的效果：

[DEMO](https://codepen.io/realign/pen/rmXmJx)

然后去捞了一下阮老师es6官网的源码：

```
(function() {
      var $w = $(window);
      var $prog2 = $('.progress-indicator-2');
      var wh = $w.height();
      var h = $('body').height();
      var sHeight = h - wh;
      $w.on('scroll', function() {
        window.requestAnimationFrame(function(){
          var perc = Math.max(0, Math.min(1, $w.scrollTop() / sHeight));
          updateProgress(perc);
        });
      });

      function updateProgress(perc) {
        $prog2.css({width: perc * 100 + '%'});
        ditto.save_progress && store.set('page-progress', perc);
      }

    }());
```

![](http://olz3b8fm9.bkt.clouddn.com/17-5-31/97007173.jpg)

不出所料，做法是一样的。

然后，就开始瞎琢磨，之前用 ```css``` 搞过瀑布流，那可以用 ```css``` 搞滚动指示器吗？


### CSS的做法

使用  ```CSS```来实现滚动指示器，难点在于：如何实时去获取当前可视区域在文档流中的位置。

上述问题，如何去考虑呢？我们从分解滚动指示器的动作中来找答案：

* 黑色表色进度条
* 红色表示文档高度
* 绿、蓝、橙表示视窗滚动
![](http://olz3b8fm9.bkt.clouddn.com/17-5-31/39951947.jpg)

也就是说：```(TH / (DH - VH)) ```公式中的 ```TH``` 可以不用知道，只需要```DH - VH``` 的高度，即直角三角形的高度便OK。

现在问题转化为：如何求出VH，这时候该 ```vh``` 这个长度单位登场了，[vh](http://www.zhangxinxu.com/wordpress/2012/09/new-viewport-relative-units-vw-vh-vm-vmin/) 是基于视窗单位的排版计量。vh可以获取当前视窗的高度。嗯，现在看来，应该是可以一写了。

[DEMO](https://codepen.io/realign/pen/bWXRYV)

```
<body>
    <div class="main">
        <h1>滚动鼠标</h1>
    </div>
</body>
```

```
.main {
   margin: 0;
   padding: 0;
   display: block;
   height: 30000px;
   text-align: center;
   line-height: 100px;
}

body {
   margin: 0;
   padding: 0;
   background: linear-gradient(to right top, #369 50%, #fff 50%);
   background-size: 100% calc(100% - 99vh);
}

body:before {
   content: '';
   position: fixed;
   top: 4px;
   bottom: 0;
   width: 100%;
   z-index: -1;
   background: #fff;
}
```

大致实现是没有问题的，但是有下面几个缺点：

* 文档内容太少（高度太小）的话，进度条呈箭头形，不美观（可考虑加毛玻璃效果来弱化）
* ```background-size: 100% calc(100% - 99vh);``` 中的99vh是相对值，若是视窗高度比较小，进度条会填不满进度条槽（可考虑加min-height来弱化）

以上


