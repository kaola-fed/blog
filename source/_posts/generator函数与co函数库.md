---
title: generator函数与co函数库
date: 2017-06-27
---

## generator 函数

### 语法
参考：http://es6.ruanyifeng.com/#docs/generator  
> 每次调用next方法，内部指针就从函数头部或上一次停下来的地方开始执行，直到遇到下一个yield表达式（或return语句）为止。换言之，Generator 函数是分段执行的，yield表达式是暂停执行的标记，而next方法可以恢复执行。当执行到done为true时，这个generator对象就已经全部执行完毕，不要再继续调用next()了。

<!-- more -->

### 注意点
(1) 遇到yield表达式，就暂停执行后面的操作，将紧跟在yield后面的表达式的值作为返回对象的value属性值
(2) 如果没有遇到yield会一直运行到return 语句为止
(3) 注意非常重要的一点（看个例子）：
  * yield XXX 表达式本身并没有返回值，或者说总是返回undefiend。
  * 可以把yield理解为一个带有暂停功能的return
  
```javascript
function* gen(x){
  var y = yield x + 2;
  return y;
}

var g = gen(1);
g.next() // { value: 3, done: false }
g.next() // { value: undefined, done: true }
```

### 小例子
```javascript
var fs = require('fs');
var readFile = function(fileName) {
  return new Promise(function (resolve, reject){
  fs.readFile(fileName, function(error, data){
    if (error) reject(error);
    resolve(JSON.parse(data.toString()));
  });
 });
}
var gen = function\* () {
  var f1 = yield readFile('package.json'); //Promise
  var f2 = yield readFile(f1);
  f2.value.then(function(data) {
  console.log(data);
 });
}

// generator 的问题就是要一步一步的写
var g = gen();// 初始化了指针
var r1 = g.next(); // 开始读第一个文件
r1.value.then(function(data) {
  var r2 = g.next(data.name);
  g.next(r2);
  g.next();
});
```

### 使用generator函数不便利之处？
generator 函数的不便利支持在于我们要不断的next，并且通过参数去传递需要用的值。所以TJ大神写了一个co库，用于自动的执行generator函数

## co 函数库
### co函数库用法   
co接受一个generator函数作为参数，并自动化地执行异步操作后返回promise对象。yield值可以是一个generator，promise，数组，对象或函数。
### 小例子
```javascript
var co = require('co'),
fs = require('fs');

var readFile = function(fileName) {
  return new Promise(function (resolve, reject){
    fs.readFile(fileName, function(error, data){
      if (error) reject(error);
      resolve(JSON.parse(data.toString()));
    });
 });
}

co(function* (){
  var a = yield readFile('./package.json');
  var b = yield readFile('./'+a.name);
  return yield Promise.resolve(b);
}).then(function(val){
  console.log(val); // { content: '从package.json获取参数后读取到的最终文件' }
}).catch(function(error){
  console.log(error);
});
```
### 为什么co函数库可以自动执行next?
基于Promise实现的co，看下源码片段
```javascript
https://github.com/tj/co/blob/master/index.js
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1);//截取参数

  return new Promise(function(resolve, reject) {
    if (typeof gen === 'function') gen = gen.apply(ctx, args);// 执行gen得到指针
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    onFulfilled();

    /**
     * @param {Mixed} res
     * @return {Promise}
     * @api private
     */

    function onFulfilled(res) {
      var ret;
      try {
        ret = gen.next(res);// 执行next
      } catch (e) {
        return reject(e);
      }
      next(ret);// 重点在这个函数
      return null;
    }
    
    function next(ret) {
      if (ret.done) return resolve(ret.value);//如果done为true 结束
      var value = toPromise.call(ctx, ret.value);// 转为Promise
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);//返回结果后又会进入onFullfilled方法--->再次进入next
      return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
        + 'but the following object was passed: "' + String(ret.value) + '"'));
    }
    });
}

/**
 * Convert a `yield`ed value into a promise.
 *
 * @param {Mixed} obj
 * @return {Promise}
 * @api private
 */

function toPromise(obj) {// 这也是一个重点的函数 转为Promise的包装函数
  if (!obj) return obj;
  if (isPromise(obj)) return obj;
  if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
  if ('function' == typeof obj) return thunkToPromise.call(this, obj);
  if (Array.isArray(obj)) return arrayToPromise.call(this, obj);
  if (isObject(obj)) return objectToPromise.call(this, obj);
  return obj;
}
```
接下来可以看看ES7 async functions....

## 参考资料
[1]http://es6.ruanyifeng.com/#docs/generator
[2]http://www.ruanyifeng.com/blog/2015/04/generator.html
[3]http://www.ruanyifeng.com/blog/2015/05/thunk.html
[4]http://fengliner.github.io/2016/11/28/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3ES6%E5%BC%82%E6%AD%A5%E7%BC%96%E7%A8%8B/

