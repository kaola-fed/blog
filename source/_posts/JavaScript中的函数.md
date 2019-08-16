---
title: JavaScript中的函数
date: 2017-12-27
---

起因是我在逛sf的时候看到了一个人的提问：

```js
为什么将函数c赋值给变量b，在函数体里面，给c赋值，为什么会失败？也就是这代码执行时为什么c打印出来的不是3

var  b = function c () {
    a=1, b=2, c=3;
    console.log(a);
    console.log(b);
    console.log(c);
}
b();

```

把上面这段代码在控制台中运行一下，得出的结果是：
```
1
2 
f c(){ ... }
```

看到其他的回答中有一个说c被声明成了全局变量，我当时是比较怀疑的，因为这种具名函数表达式是可以用函数名调用自身的，c会是函数本身的引用，所以我把`console.log(c)`改成了`console.log(window.c)`：
```js
var b = function c() {
    a=1, b=2, c=3;
    console.log(a);
    console.log(b);
    console.log(window.c);
}
b();

```
再执行，输出的结果是: `1 2 undefined`，很明显，`c`并没有声明到全局，所以说`c=3`这个语句只是静默失败了。在非严格模式下，JavaScript代码的很多行为都会静默失败，但在严格模式下就会暴露问题所在，然后我给这个函数加上`'use strict'`再运行一下：
```js
var b = function c() {
    'use strict';
    var a=1, b=2;
    c=3;
    console.log(a);
    console.log(b);
    console.log(window.c);
}
b();
```
果然报错:`Uncaught TypeError: Assignment to constant variable.`，说明这个c是不可变的，至于为什么不可变，就属于JavaScript函数本身的特点了。然后我在网上搜索了一些资料，在[ECMA-262-3 in detail. Chapter 5. Functions.](http://dmitrysoshnikov.com/ecmascript/chapter-5-functions/#feature-of-named-function-expression-nfe)找到了一点相关内容，文中提到了具名函数表达式(NFE)的一些特点，比如如下的例子：
```js
(function foo(bar) {
  
  if (bar) {
    return;
  }
  
  foo(true); // "foo" name is available
  
})();
  
// but from the outside, correctly, is not
  
foo(); // "foo" is not defined
```
其中一个特点是`foo`这个名字在函数内部可用，这个特点由NFE的工作方式决定：
当解释器在执行遇到具名函数表达式时，在创建函数表达式之前，它会创建一个特殊的辅助对象，并且加载当前的作用域链上。然后创建函数表达式本身，在该阶段函数获得[[Scope]]属性，即创建函数的上下文作用域链，然后函数表达式的`name`会作为唯一(unique)的属性添加到特殊对象上，属性的值就是函数表达式的引用。然后最后一步就是从父作用域链中将该特殊对象删除，整个过程的伪代码如下：
```js
specialObject = {};
  
Scope = specialObject + Scope;
  
foo = new FunctionExpression;
foo.[[Scope]] = Scope;
specialObject.foo = foo; // {DontDelete}, {ReadOnly} 注意这里，该属性不能删除，只读。
  
delete Scope[0]; // remove specialObject from the front of scope chain
```

对NFE工作原理理解过后，就很容易理解NFE的特性和文章开头的问题了。

END
