---
title: 请谨慎使用nej框架提供的工具函数_$format
date: 2017-06-30
---
前阵子评论系统的后端开发同学反馈，偶尔会收到 NaN-00-00 00:00:00 这样格式的时间，希望我这边能协助排查一下。

根据提供的信息，很容易联想的格式化时间的时候出现了问题。评论系统页面并不多，并且查询条件也很类似，因此找了一个比较典型的页面开始分析。

<!-- more -->

从时间相关的参数下手，找到了 \_u._$format 函数的身影（代码太多了，不贴了），顺藤摸瓜，又牵扯数了\_$var2date的函数：

```javascript

_p._$var2date = function(_time){
    var _date = _time;
    if (_p._$isString(_time)){
        _date = new Date(
            _h.__str2time(_time)
        );
    }
    if (!_p._$isDate(_date)){
        _date = new Date(_time);
    }
    return _date;
};

```
接着，我们来看下_h.__str2time的使用姿势:

```javascript
NEJ.define([
    './global.js',
    '{platform}util.js'
],function(NEJ,_h,_p,_o,_f,_r){
    
})
```

请注意这里的{platform}，根据nej框架的[平台适配系统](https://github.com/genify/nej/blob/master/doc/PLATFORM.md)介绍，这是用于处理跨平台的配置参数。顺理成章，找到了util.js和他的兄弟util.patch.js。

util.js中都是正常的实现，util.patch.js则是针对非标准浏览器的实现。

大家应该都经历过，在使用new Date()时，需要先将'2017-05-25 08:20:30' 中的 '-' 替换为 '/'，否则，在一些浏览器上会存在兼容性问题。

在util.patch.js中，也找到了nej用于兼容旧版浏览器的解决方案。

```javascript

/**
 * YYYY-MM-DDTHH:mm:ss.sssZ格式时间串转时间戳
 * @param  {String} 时间串
 * @return {Number} 时间戳
 */
_h.__str2time = (function(){
    var _reg = /-/g;
    return function(_str){
        // only support YYYY/MM/DDTHH:mm:ss
        return Date.parse(_str.replace(_reg,'/').split('.')[0]);
    };
})();

```

一切都看似很完美，按说这样处理过就不会有问题了，但是，当我掏出那个不常用的IE10浏览器来验证时，还是出现了开头所说的问题。接着，在IE10的控制台上进行了验证，确认了IE10在使用new Date()时是不支持 '2017-05-25 08:20:30'这种格式的。

接着，经过调试，IE10并没有调用util.patch.js中的方法，那么问题就很明确了，nej的兼容性配置应该存在问题，并没有把IE10列入到需要打补丁的浏览器中。

仔细看了一遍util.patch.js的文件描述:

```javascript
/*
 * ------------------------------------------
 * 平台适配接口实现文件
 * @version  1.0
 * @author   genify(caijf@corp.netease.com)
 * ------------------------------------------
 */
NEJ.define([
    './util.js',
    'base/platform'
],function(_h,_m,_p,_o,_f,_r){
    // for ie8-
    NEJ.patch('TR<=4.0',function(){
    
    });
})();

```



至此，问题就有了答案了：
'for ie8-'、TR<=4.0'，这个补丁分明就是只是针对IE8以下的嘛，IE10（Trident/6.0）这种看似正常却又不完整的浏览器根本不在人家的考虑范畴内哇！

总结：使用nej提供的_$format函数时，请先确保传入的是时间对象或者已经转为标准格式的 '2017/05/25 08:20:30' 字符串，否则，IE10下会有兼容性问题。另外，nej框架内置的一些util函数在跨平台上并没有考虑的非常周全，需要谨慎使用。




<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
