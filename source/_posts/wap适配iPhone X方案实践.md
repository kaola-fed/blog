> **关于异形屏的安全区以及苹果官方的文档，网上有很详尽的博客，这里不再累述，本文偏实践。阅读本文前，[可点击了解safe-Area概念](http://www.zcool.com.cn/article/ZNTU1MTUy.html)**

# 适配iphone X方案：
## 1. 实现原理：
    > 首先我们需要了解一个属性：constant(safe-area-inset-bottom)/env(safe-area-inset-bottom)；这个的含义是，安全区底部距离屏幕底部的距离。因此，非异形屏下，这个值为0/或不解析；异形屏下，这个值为真实的距离，比如34px啥的。
    
接下来，我们需要做的有三点：
1. 找到有底部banner的页面
2. 将底部banner的父容器从bottom: 0; 升级成 bottom: 0;bottom: constant(safe-area-inset-bottom); bottom: env(safe-area-inset-bottom); // bottom: 0;是兼容非异形屏，bottom: constant(safe-area-inset-bottom);是为了让banner上移到安全区内。
3. 给 banner:after添加纯色遮罩bottom: 0; height: 0; height: constant(safe-area-inset-bottom); height: env(safe-area-inset-bottom); background: #fff;
（注：iOS11支持constant语法，future版本支持env语法）

然后我们发现底部的banner确实是在安全区内了，而且底部有纯色遮罩覆盖，不会有穿透效果。

然而。。。。我们是否忘记了，底部banner上移了，那页面里原有的内容区是不是被盖住了一部分？
所以，我们需要再给body:after加上height: 0; height: $safeArea（bottom）;把body也撑起一个高度，使得内容区不会被上移了的banner遮住。

## 2. 实现方式：
```css
/* _config.mcss */
$safeAreaHeight = ($height){
    height: $height;
    height: t('constant(safe-area-inset-bottom)') !important; 
    height: t('env(safe-area-inset-bottom)') !important;
}
$safeAreaBottom = ($height){
    bottom: $height;
    bottom: t('constant(safe-area-inset-bottom)') !important; 
    bottom: t('env(safe-area-inset-bottom)') !important;
}

```

```css
/* base.mcss */
body:after {
    display: block;
    content: '';
    width: 100%;
    $safeAreaHight(0);
}
```

```css
/* module.mcss */
.f-safeArea {
    $safeAreaBottom(0);
}
.f-safeArea:after {
    $safeAreaHeight(0);
    display: block;
    content: '';
    position: fixed;
    left: 0;
    bottom: 0;
    width: 100%;
    background: #fff;
}

```

```html
<!-- common.ftl -->
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover, minimal-ui">
```

## 3. 实现效果：
APP内：
![APP](https://haitao.nos.netease.com/57545adf-fe82-4053-99fd-442175882eaa.jpg?imageView&thumbnail=300x0)
Safari非全屏：
![Safari非全屏](https://haitao.nos.netease.com/19d67427-b483-4708-90d2-43f38a0ff028.jpg?imageView&thumbnail=300x0)
Safari全屏：
![Safari全屏](https://haitao.nos.netease.com/1583cfdd-87a8-4784-bbe5-6dcff82b6d7e.jpg?imageView&thumbnail=300x0)

总结：
1. wap中的底部banner没有一个通用组件的形式，因此没有通用方法去适配，只能将涉及到底部banner的页面进行针对性的修改。
2. 因为涉及到的页面较多，本方案采用在不侵入原有的代码的情况下，通过添加class的方案去兼容覆盖原有样式，达到兼容iPhone X的目的。因为不破坏原有的DOM结构，修改的成本较小。
3. 可以考虑以后在wap中写一个通用banner组件容器，baner中和业务逻辑相关的代码作为组件的$body，以后做兼容只需要修改通用组件，节省成本。