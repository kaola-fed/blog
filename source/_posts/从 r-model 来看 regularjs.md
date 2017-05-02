---
title: 从 r-model 来看 regularjs
date: 2017-03-27
---

### 从 r-model 来分析 regularjs

大家都知道 r-model 是一个指令，关于指令的[语法](http://regularjs.github.io/guide/zh/basic/directive.html)可以看源码中关于 r-model 的部分

<!-- more -->

```
Regular.directive("r-model", {
  param: ['throttle', 'lazy'],
  link: function( elem, value, name, extra ){
    var tag = elem.tagName.toLowerCase();                                                  // 标签名
    var sign = tag;                                                                 
    if(sign === "input") sign = elem.type || "text";                                       // 
    else if(sign === "textarea") sign = "text";
    if(typeof value === "string") value = this.$expression(value);

    if( modelHandlers[sign] ) return modelHandlers[sign].call(this, elem, value, extra);
    else if(tag === "input"){
      return modelHandlers.text.call(this, elem, value, extra);
    }
  }
})
```
关键的是 link 函数，elem 是绑定这个指令的 dom 节点， value 是指令后面的值，如果这个值是字符串那么 value 也是字符串，如果这个值是插值，那个这个 value 就是一个 expression 对象，这个对象是在 parse 过程中产生的， regular 在 parse 的时候每次解析到一个插值就会生成一个 expression 对象。   
expression 对象长这样，包含一个 get 方法和 set 方法  
```
expression {
	"type": "expression",
	"touched": "true",
	"once": "false",
	"get": function anonymous(c, e) {
		var d = c.data;e = e||'';return (c._sg_('testWatch', d, e))
	},
	"set": function (ctx, value, ext) {
		expr.set = new Function(_.ctxName, _.setName, _.extName, _.prefix + setbody)
		return expr.set(ctx, value, ext);
	}
}
```
分析 r-model 源码，主要是获取绑定 r-model 的 dom 节点，然后匹配到对应的方法上，这个 map 长这样  
```
var modelHandlers = {
  "text": initText,
  "select": initSelect,
  "checkbox": initCheckBox,
  "radio": initRadio
}
```
如果这个 value 的值是一个字符串，r-model 指令还会主动将字符串转换成 expression 对象，所以 r-model={test} 和 r-model="test" 是一样的。关于每种节点的类型需要处理的方法其实大同小异，找一个分析一下。  
```
function initSelect( elem, parsed, extra){
  var self = this;
  var wc = this.$watch(parsed, function(newValue){
    var children = elem.getElementsByTagName('option');
    for(var i =0, len = children.length ; i < len; i++){
      if(children[i].value == newValue){
        elem.selectedIndex = i;
        break;
      }
    }
  }, STABLE);

  function handler(){
    parsed.set(self, this.value);
    wc.last = this.value;
    self.$update();
  }

  dom.on( elem, "change", handler );
  
  if(parsed.get(self) === undefined && elem.value){
    parsed.set(self, elem.value);
  }

  return function destroy(){
    dom.off(elem, "change", handler);
  }
}
```
这个就是 r-model 对于 select 节点的处理方法。比如现在有一个` <select r-model={selectModel}> ... </select> `，执行 initSelect 方法。  
首先实现 model => view 的绑定，主要方法就是上面 $watch 的部分，watch 插值，一旦插值的值变化就执行方法遍历 select 的 option ，找到 option 的 value 等于插值的值的那个 index 然后将其设置成选中状态，如果是 input 就更简单，直接将 input 的 value 设置成插值的值。然后实现 view => model 的绑定，这部分主要是通过原生 dom 方法 onchange 来实现的，给绑定了 r-model 指令的 dom 节点绑定 onchange 事件，那么这个节点改变的时候就会触发方法，将 dom 的 value 赋值给对应的插值。  
就这样实现了双向的绑定，view => model 这部分实现比较简单。就不过多讲解。后面主要看看 model => view 中的 $watch 部分到底是怎么实现的  

首先来看看 $watch 这个函数长什么样子，有点长
```
$watch: function(expr, fn, options){
    var get, once, test, rlen, extra = this.__ext__; //records length
    if(!this._watchers) this._watchers = [];
    if(!this._watchersForStable) this._watchersForStable = [];

    options = options || {};
    if(options === true){
       options = { deep: true }
    }
    var uid = _.uid('w_');
    if(Array.isArray(expr)){
      var tests = [];
      for(var i = 0,len = expr.length; i < len; i++){
          tests.push(this.$expression(expr[i]).get)
      }
      var prev = [];
      test = function(context){
        var equal = true;
        for(var i =0, len = tests.length; i < len; i++){
          var splice = tests[i](context, extra);
          if(!_.equals(splice, prev[i])){
             equal = false;
             prev[i] = _.clone(splice);
          }
        }
        return equal? false: prev;
      }
    }else{
      if(typeof expr === 'function'){
        get = expr.bind(this);      
      }else{
        expr = this._touchExpr( parseExpression(expr) );
        get = expr.get;
        once = expr.once;
      }
    }

    var watcher = {
      id: uid, 
      get: get, 
      fn: fn, 
      once: once, 
      force: options.force,
      // don't use ld to resolve array diff
      diff: options.diff,
      test: test,
      deep: options.deep,
      last: options.sync? get(this): options.last
    }


    this[options.stable? '_watchersForStable': '_watchers'].push(watcher);
    
    rlen = this._records && this._records.length;
    if(rlen) this._records[rlen-1].push(watcher)
    // init state.
    if(options.init === true){
      var prephase = this.$phase;
      this.$phase = 'digest';
      this._checkSingleWatch( watcher);
      this.$phase = prephase;
    }
    return watcher;
  },
```
首先这个函数接收三个参数，在最上面 r-model 的方法里面也可以看到，第一个参数是 watch 的插值，类型是 expression 对象，第二个参数是 watch 的值发生改变之后要执行的函数，第三个参数是一些 option ，可以看到在 initSelect 方法里面这个参数传递的是一个对象 `{ stable: true }` ，看函数体，首先创建了一些变量，然后初始化了两个数组 '_watchersForStable', '_watchers'，经过一些判断， 第一个参数是否是数组，是否是函数，最后生成一个 watcher ，watcher 长这样
```
var watcher = {
  id: uid, 
  get: get, 
  fn: fn, 
  once: once, 
  force: options.force,
  // don't use ld to resolve array diff
  diff: options.diff,
  test: test,
  deep: options.deep,
  last: options.sync? get(this): options.last
}
```  
注意 watcher 里面有维护一个 last 字段，这个字段表示这个 watcher 上一次的值。还有一个 fn 字段，这个字段就是改变之后要执行的函数。
然后判断 option 里面的 stable 是否是 true ，如果是就 push 进 _watchersForStable，不是就 push 进 _watchers，猜测，前面那个数组是保存 r-model 的 watcher 对象，后面那个是保存用户自己写的 watcher 对象。然后 return 这个 watcher 对象，到此函数结束。  
$watch 主要做的事情就是把需要 watch 的 expression 对象 push 到 '_watchersForStable' 或者 '_watchers' 数组中。  
很明显这个 $watch 函数并不能检测到值的变化。真正检测到这个变化的是脏值检测的过程。  
触发脏值检测过程的函数是 $update ，函数长这样
```
$update: function(){
	var rootParent = this;
	do{
	  if(rootParent.data.isolate || !rootParent.$parent) break;
	  rootParent = rootParent.$parent;
	} while(rootParent)

	var prephase =rootParent.$phase;
	rootParent.$phase = 'digest'

	this.$set.apply(this, arguments);

	rootParent.$phase = prephase

	rootParent.$digest();
	return this;
},
```  
这个函数的主要作用就是找到根节点 rootParent ,然后 rootParent.$digest()  
脏值检测的主要过程在 $digest 函数中，这个函数长这样
```
$digest: function(){
    if(this.$phase === 'digest' || this._mute) return;
    this.$phase = 'digest';
    var dirty = false, n =0;
    while(dirty = this._digest()){

      if((++n) > 20){ // max loop
        throw Error('there may a circular dependencies reaches')
      }
    }
    // stable watch is dirty
    var stableDirty =  this._digest(true);

    if( (n > 0 || stableDirty) && this.$emit) {
      this.$emit("$update");
      if (this.devtools) {
        this.devtools.emit("flush", this)
      }
    }
    this.$phase = null;
},
```
这个函数首先将 $phase 字段置为 digest 代表在脏值检测的阶段，然后执行 _digest() 方法，来看看 _digest 方法
```
_digest: function(stable){
  var watchers = !stable? this._watchers: this._watchersForStable;
  var dirty = false, children, watcher, watcherDirty;
  var len = watchers && watchers.length;
  if(len){
    var mark = 0, needRemoved=0;
    for(var i =0; i < len; i++ ){
      watcher = watchers[i];
      var shouldRemove = !watcher ||  watcher.removed;
      if( shouldRemove ){
        needRemoved += 1;
      }else{
        watcherDirty = this._checkSingleWatch(watcher);
        if(watcherDirty) dirty = true;
      }
      // remove when encounter first unmoved item or touch the end
      if( !shouldRemove || i === len-1 ){
        if( needRemoved ){
          watchers.splice(mark, needRemoved );          
          len -= needRemoved;
          i -= needRemoved;
          needRemoved = 0;
        }
        mark = i+1;
      }
    }
  }
  // check children's dirty.
  children = this._children;
  if(children && children.length){
    for(var m = 0, mlen = children.length; m < mlen; m++){
      var child = children[m];
      if(child && child._digest(stable)) dirty = true;
    }
  }
  return dirty;
},
```  
这个函数首先判断是否传入一个 stable 来决定是用 _watchers 还是 _watcherForStable ，然后遍历数组里面的每一个 watcher 来看这个 watcher 是否是脏的，是否是脏的是用 _checkSingleWatch 这个函数来判断的，这个函数有点长不方便贴，主要是一个 diff 功能，根据 watcher 里面的 last 字段用来获取上次的值和 watcher.get() 方法用来获取当前的值，做 diff 之后如果是脏的就将 now 赋值给 last 然后执行 watcher 的回调函数 fn ，最后 return 一个 true。   
遍历完整个数组之后再遍历这个 rootParent 的 children 继续执行这个方法，将这整个 rootParent 遍历完之后，过程中如果有一个 watcher 是脏的，那么整个脏值检测的过程会返回脏，如果整个过程返回脏就会再来执行一遍脏值检测的过程直到整个过程返回不脏为止，如果20次之后还是返回脏就抛出异常，循环依赖。  
脏值检测的整个过程就是这样的。  
那么剩下最后一个问题了，$update 到底是在什么时候触发的。除了我们手动去执行 $update 以外，平常的一些操作，比如给按钮绑定一个 on-click 事件，在事件里面去改变 data 的值，比如 this.data.testWatch = 'xxx'，比如在 ajax 请求的回调里面去改变 data 的值，比如在 setTimeout 里面去改变 data 的值，在这些个过程中我们并没有手动去 $update 但是 regular 会自动的进入 $update ，那么问题来了，regular 到底是在什么时候进入的，他怎么知道我们改变了 data 的值呢？  
regular 在自定义事件，比如 on-xxx 的回调函数触发之前和之后都执行了一遍 $update 然后 regular 包装了一个 timeout 方法，就是把 setTimeout 之后加了一个 $update ，但是异步操作还是不会自动的去 $update 所以我们平时的工程里面都封装了一个 $request 这个函数除了封装了标准的 ajax 以外还触发了一遍 $update。思路就是 regular 接管了大部分可能改变 data 的入口，并主动的去触发 $update 。