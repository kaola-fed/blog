---
title: 不同加载方式的js脚本执行顺序的观察
date: 2017-07-26
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

首先思考几个问题:

- 1.内联脚本和标签引入的脚本, 哪个先执行?

```
<script src="a.js"></script>
<script>
console.info('this is inline');
</script>
```
<!-- more -->

- 2.用内联脚本动态向页面中append一个脚本, 并给脚本注册onload回调, 脚本的内容和onload哪个先执行?

```
<script type="text/javascript">
            console.info('this is inline');
            var script = document.createElement('script');
            script.src = 'a.js';
            script.onload = function() {
                console.info('onload');
            }
            document.body.appendChild(script);
</script>

// a.js
(function() {
    console.info('this is a');
}())
```



- 3.向页面中append一个脚本后, 再把这个脚本remove掉, 脚本会执行吗? 

- 4.如果写一个第三方库, 要在库加载的时候获取一些信息, 这些信息可以写到引入库的script标签上吗?


下面提供三个代码, 解答前三个问题:

page.html

```
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title></title>
    </head>
    <body>
        <script type="text/javascript">
            console.info('this is inline');
            var script = document.createElement('script');
            script.src = 'a.js';
            script.onload = function() {
                console.info('onload');
            }
            script.id = 'abc';
            document.body.appendChild(script);
            console.info('append');
        </script>
        <script src="b.js"></script>
        <script>
          console.info('inline2');
        </script>
    </body>
</html>

```


a.js

```
(function() {
    console.info('this is a');
    var script = document.querySelector('#abc');
    console.info(script);
}())

```


b.js

```
(function() {
    console.info('this is b');
    var script = document.querySelector('#abc');
    document.body.removeChild(script);
    console.info('removed');
}())
```

结论: 

1. 内联脚本和标签脚本的执行顺序, 由脚本在页面中的位置确定;
2. 动态加载的脚本, 脚本内容会先执行, onload后执行;
3. 动态加载的脚本会在所有标签脚本执行后执行, 只要append了脚本, 即使马上removeChild把脚本所在的标签删掉, 脚本也会执行; 只是脚本无法获取到自己所在的标签了.