---
title: generator基本概念
date: 2017-06-27
---

- [x] generator基本概念
- [x] generator流程控制
--------
生成器这个概念在其他编程语言比如C++,PHP,Javaj,Python等语言中都有广泛应用，于是es6也将这一特性引入。

<!-- more -->

>生成器是一种被用来控制循环中迭代行为的特殊程序。实际上，所有的生成器都是迭代器。生成器和一个返回了数组的函数很类似，在这里生成器有参数能够被调用，并且生成一系列值。然而和数组不同的是，生成器不是一次存储所有的值并且一次性返回完，而是每调用一次返回一次并且停止等待下一次调用。这样只需要很少的内存，且允许调用者立即处理开始的几个数据。简而言之，生成器看起来像函数，表现得像迭代器    
--wikipedia

要从整体上了解这一概念，需要了解一些与之相关的知识与概念。

## 可迭代协议(iterable protocols)

### 定义

`iterable`协议可以允许对象定义或者定制迭代行为，要能够可迭代，对象需要继承或者有`@@iterator`方法，`@@iterator`属性可以通过`Symbol.iterator`（是一个常量）获得，`@@iterator`方法是一个零参数并返回一个遵守`iterator`协议对象的函数。如果一个对象有`@@iterator`方法，但返回的对象不是迭代器，就会成为一个不正确的可迭代对象。

当一个对象是可迭代的并且**需要迭代的时候**，就会无参调用`@@iterator`，返回的迭代器用来获得迭代出来的值

### 需要迭代的时候

- `for...of`
- `spread operator`
- `destructuring assignment`
- `yield*`

### 内置可迭代对象

- `String`
- `Array`
- `TypedArray`
- `Map`
- `Set`

### 定制迭代行为

即重写或者添加`@@iterator`方法，所写方法遵守迭代器协议(`iterator protocols`)即可，否则为不正确的可迭代对象。
```
// 为对象添加@@iterator
var o = {};
o[Symbol.iterator] = function() {
    return {
        tick: 0,
        next: function() {
            return {
                value: this.tick > 3 ? undefined:this.tick++,
                done: this.tick > 4 ? true:false
            }
        }
    }
}
var a = o[Symbol.iterator](); // 获得迭代器

a.next(); // {value: 1, done: false}
a.next(); // {value: 2, done: false}
a.next(); // {valeu: 3, done: false}
a.next(); // {value: undefined, done: true}
```

---------------

## 迭代器协议(`iterator protocols`)

### 定义

`iterator`协议定义了一个标准方法(`next`)来产生序列值。

要成为迭代器，需要继承或实现`next`方法，`next`方法返回了两个属性，`done`和`value`

### 与可迭代对象的关系

- 某些迭代器也是可迭代的对象，比如generator对象

- 可迭代对象调用@@iterator返回迭代器(即协议的内容)

--------------------

## 生成器(`generator`)

### 生成器函数(`generator function`)

#### 定义生成器函数

- `function*`声明
- `function*`表达式
- `GeneratorFunction`构造器

以`function*`声明为例
```
function* myGenerator() {
    yield 1;
    yield 2;
    yield 3;
}
var g = myGenerator();  // 得到generator object
g.next(); // {value: 1, done: false}
g.next(); // {value: 2, done: false}
g.next(); // {valeu: 3, done: false}
g.next(); // {value: undefined, done: true}
/**
 *可以在调用next的时候向next方法中传入参数，
 *此时的参数会被传入generator function中，与内部的变量进行通信。
 *需要注意的是传值的方向是外部返回回内部的时候，而第一次调用next的时候，是遍
 *历的开始，就算传了值内部也无变量接收。
 *
*/
function* myGenerator2() {
    var a = yield 1;
    console.log(a);
    var b = yield 2;
    console.log(b);
}
var g2 = myGenerator2(); // 得到generator object
g2.next();  // {value: 1, done: false}
g2.next('a:1'); // 'a:1'  {value: 2, done: false} 
g2.next('b:2'); // 'b:2'  {value: undefined, done: true} 
```

### 生成器对象(`generator object`)

`generator`对象是需要'generator function`执行才能得到。

`generator`对象遵循`iterable`协议和`iterator`协议，所以`generator`对象既是可迭代对象又是迭代器

### 原型链方法

- next()

返回两个参数，`yield`表达式后面的值`value`和表示遍历是否结束的`done`;

- return()

遇到`return`，和一般函数一样，结束函数。下面如果还有`yield`，则无视。

- throw(exception)

为了调试方便，`exception`一般是`Error`的实例，返回的结构和和调用`next`一样


## 应用

### 流和无穷序列

由于`generator`这种需要的时候才会去取的特性，所以对于描述流很有用，比如不可能一次计算完或者过量消费`cpu`的序列。包括无穷序列和实时数据流。

```
// 经典的无穷序列
function* fibonacci() {
    let [pre, cur] = [0, 1];
    while(true) {
        yield cur;
        [pre, cur] = [cur, pre + cur];
    }
}
```
---------------------

参考文献：
- [MDN Iteration protocols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#The_iterator_protocol)
- [MDN Symbol.iterator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/iterator)
- [MDN function*](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*)
- [MDN Generator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator)
- [MDN Iterators and generators](https://developer.mozilla.org/en/docs/Web/JavaScript/Guide/Iterators_and_Generators)
- [WIKI Generator](https://en.wikipedia.org/wiki/Generator_(computer_programming))
- [WIKI Iterator](https://en.wikipedia.org/wiki/Iterator)
