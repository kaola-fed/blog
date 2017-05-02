---
title: 如何理解JS中的闭包
date: 2017-04-19
---

## 如何理解JS中的闭包？

对于函数是一等对象的语言来说，下面两种典型的情况，会存在变量如何取值的问题。

<!-- more -->

**case a**. 将函数当参数传递
```javascript
var x = 15;
function A () {
    console.log(x);
}

function B (fn) {
    var x = 20;
    fn();
}

B(A);  // 15, but not 20
```

**case b**. 将函数作为返回值
```javascript
function B () {
    var x = 10;
    function A () {
        console.log(x);
    }
    
    return A;
}
    
var x = 20;
B()();  // 10, but not 20
```

从语言实现层面来看，JS中闭包是解决上面两种情况下变量取值的一种机制。

对于闭包常见的描述，比如缓存变量等，只不过是对上面语言实现特性的典型应用。

JS中变量作用域使用的是`静态作用域(static scope)`，在JS引擎解析JS代码时，各个函数能访问到的作用域以及相应的变量已经定了，并且不会再改变。比如上面`函数A`定义的时候，其能访问到的变量已经定了。对于`case a`，`函数A`声明的时候已经决定了其只能访问到全局作用域，并且后续不会发生改变，即始终只能访问到全局变量x和全局函数声明B，访问不到`函数B`中的局部变量`x`，不管其后续如何执行，打印出来的结果总是全局变量x的值；对于`case b`，`函数A`在`函数B`执行的时候才被声明，其能访问到的作用域，首先是`函数B`的局部作用域(局部变量x)，然后是全局作用域(全局变量x和全局函数声明B)，所以`函数A`打印的始终是局部变量x的值，而不管其后续如何执行。

通过上面的描述，我们也可以推断*闭包名字的由来*

> 闭包（closure）就像是对作用域的一种闭合（closing），在函数定义的时候，其作用域已经决定了，并且后续使用不会再改变，即作用域已经闭合了。

通过`case a`和`case b`的对比，我们可以进一步给出闭包更加广义层面的定义，***JS中任何一个函数都是闭包***，而不仅仅是`case b`中那种教科书上面狭义的定义，全局函数也是闭包，是对全局作用域的一种闭合。

### Reference
- [Closure](http://dmitrysoshnikov.com/ecmascript/chapter-6-closures/)

### Written by [Hong](https://github.com/llwanghong/blog/blob/master/misc/%E5%A6%82%E4%BD%95%E7%90%86%E8%A7%A3JS%E4%B8%AD%E7%9A%84%E9%97%AD%E5%8C%85.md)