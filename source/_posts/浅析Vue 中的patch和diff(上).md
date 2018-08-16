<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
## 疑问

1.当我修改了属性值时，vdom立即进行diff,重新渲染视图了吗？

2.如果1是对的，那重复修改，性能岂不是很差？如果不是，1是如何实现的？

3.我们的nextTick 具体的实现是怎样的？什么时候需要用到它？

4.vdom diff的过程是怎样的？

## 梗概

关于vue数据更新渲染的几个知识点，先列一下：

* 数据的更新是实时的，但是渲染是异步的。

* 一旦数据变化，会把在同一个事件循环event loop中的观察到的watcher 推入一个队列（相同watcher实例不会重复推入）

* DOM并不是马上更新视图的，2.4版本之后的是利用MicroTask或者MacroTask来批处理DOM更新视图的，那就要了解event loop

* 整个script是一个主任务, setTimeout 是一个macroTask, promise的回调是microtask, 顺序是

  主script（第一个主任务） —> microTask （全部执行完）—> UI渲染 —> 下一个macroTask

  ```javascript
  // 先不看结果，想一下你的输出
  console.log('script start');

  setTimeout(function() {
    console.log('setTimeout');
  }, 0);

  Promise.resolve().then(function() {
    console.log('promise1');
  }).then(function() {
    console.log('promise2');
  });

  console.log('script end');
  // result: 
  /**
  * script start
  * script end
  * promise1
  * promise2
  * setTimeout
  */
  ```


* 所以dom diff 这个过程是在microtask中去处理的（也有的是强制走macrotask, 本例子走microtask）

## 例子
接下来，所有的讲解都会围绕下面这个例子
```javascript
<!DOCTYPE html>
<html>
<head>
	<title></title>
    <style>
        .f-error {
            color: red;
        }
    </style>
</head>
<body>
    <div id="test">
    </div>
    <script type="text/javascript" src="./vue.js"></script>
    <script type="x-template" id="temp">
        <section>
            <div :class="{'f-error': a==2}">{{a+b}}</div>
        </section>
    </script>
    <script type="text/javascript">
        var data = {
            a: 1,
            b: 1
        }
        new Vue({
            el: '#test',
            template: temp,
            data: function() {
                return data;
            },
            mounted() {
                setTimeout(() => {
                    data.a = 2;
                    data.b = 3;
                    this.$nextTick(()=> {
                        console.log(document.querySelector('.f-error'));
                    })
                }, 500);
            }
        });
    </script>
</body>
</html>
```

## 入口

#### 当```vm._update(vm._render(), hydrating)```

经过上一次分享，我们知道通过```vm._render()```方法，我们会获得我们的vdom; 接下去我们进入_update方法；我们看下内部的细节。

```javascript
Vue.prototype._update = function (vnode, hydrating) {
    var vm = this;
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate');
    }
    ...
    // 初始时，我们是没有prevVnode的， 进入了patch方法
    if (!prevVnode) {
      // initial render
	  // 初始化渲染，我们看下细节	
      vm.$el = vm.__patch__(
        vm.$el, vnode, hydrating, false /* removeOnly */,
        vm.$options._parentElm,
        vm.$options._refElm
      );
      // no need for the ref nodes after initial patch
      // this prevents keeping a detached DOM tree in memory (#5851)
      vm.$options._parentElm = vm.$options._refElm = null;
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode);
    }
    activeInstance = prevActiveInstance;
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null;
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm;
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el;
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  };
```

#### 初始进入patch

```javascript
// 当我们初始进入patch时，会进入createElm 根据我们的vnode 创建节点
return function patch (oldVnode, vnode, hydrating, removeOnly, parentElm, refElm) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) { invokeDestroyHook(oldVnode); }
      return
    }
    var isInitialPatch = false;
    var insertedVnodeQueue = [];

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true;
      createElm(vnode, insertedVnodeQueue, parentElm, refElm);
    } else {
        ...
    }
}
```

## 数据更新时

假设我们的数据是```{a:1, b:1}```更新为了```{a:2, b:3}```, 我们下面看下细节

```javascript
/**
mouted() {
    setTimeout(() => {
                    data.a = 2;
                    data.b = 3;
                    this.$nextTick(()=> {
                        console.log(document.querySelector('.f-error'));
                    })
                }, 500);
}
**/

new Vue({
		el: '#test',
		template: temp,
		data: function() {
			return data;
		},
		methods: {
			test() {
				data.a = 2;
				data.b = 3;
			}
		}
	});

// 当我们data.a 的值改变时，会进入它的setter
set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if ("development" !== 'production' && customSetter) {
        customSetter();
      }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      childOb = !shallow && observe(newVal);
      // 当值更新时，dep会通知watcher
      dep.notify();
    }

Dep.prototype.notify = function notify () {
  // stabilize the subscriber list first
  var subs = this.subs.slice();
  // 我们知道subs里面存放着我们的watcher实例， 进入watcher的update方法	    
  for (var i = 0, l = subs.length; i < l; i++) {
    subs[i].update();
  }
};

Watcher.prototype.update = function update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true;
  } else if (this.sync) {
    this.run();
  } else {
    // 一般情况下，没有其他配置会进入这里,将我们的watcher推入队列  
    queueWatcher(this);
  }
};

/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
function queueWatcher (watcher) {
  var id = watcher.id;
  // 判断这个watcher是否已经放入过队列
  if (has[id] == null) {
    has[id] = true;
    if (!flushing) {
      queue.push(watcher);
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      var i = queue.length - 1;
      while (i > index && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(i + 1, 0, watcher);
    }
    // queue the flush
    if (!waiting) {
      waiting = true;
      // 走到了这里, cb 是flushSchedulerQueue
      nextTick(flushSchedulerQueue);
    }
  }
}

// 进入了nextTick 方法，这里涉及到EventLoop相关的内容，后面会简单说一下
function nextTick (cb, ctx) {
  var _resolve;
  // 将flushSchedulerQueue塞入cb
  callbacks.push(function () {
    if (cb) {
      try {
        cb.call(ctx);
      } catch (e) {
        handleError(e, ctx, 'nextTick');
      }
    } else if (_resolve) {
      _resolve(ctx);
    }
  });
  if (!pending) {
    pending = true;
    // 注意这里，一般情况下使用microTask但某些情境下会强制使用macroTask  
    if (useMacroTask) {
      macroTimerFunc();
    } else {
     // 我们的例子会进入这里， microTimerFunc结果是什么呢？往下看
      microTimerFunc();
    }
  }
  // $flow-disable-line 
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(function (resolve) {
      _resolve = resolve;
    })
  }
}

// Determine MicroTask defer implementation.
/* istanbul ignore next, $flow-disable-line */
if(typeof Promise !== 'undefined' && isNative(Promise)) {
   var p = Promise.resolve();
    microTimerFunc = function() {
        // 重点注意这里是promise的cb, 是一个microTask, 是在主script执行完才会执行的
        p.then(flushCallbacks);
    } 
}
```

data.a执行完以后，开始走data.b, 流程都一样，只是当我们遇到watcher的update时有些区别

```javascript
queueWatcher(this);
function queueWatcher (watcher) {
  var id = watcher.id;
  // 判断这个watcher是否已经放入过队列, 当执行到data.b时已经放入过队列了， 所以不会继续往下走了(这个也很好理解)
  if (has[id] == null) {
    has[id] = true;
    if (!flushing) {
      queue.push(watcher);
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      var i = queue.length - 1;
      while (i > index && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(i + 1, 0, watcher);
    }
    // queue the flush
    if (!waiting) {
      waiting = true;
      // 走到了这里
      nextTick(flushSchedulerQueue);
    }
  }
}
```

然后进入```this.$nextTick```方法

```javascript
 Vue.prototype.$nextTick = function (fn) {
    return nextTick(fn, this)
  };

function nextTick (cb, ctx) {
  var _resolve;
  // 将我们的cb塞入callbacks	    
  callbacks.push(function () {
    if (cb) {
      try {
        cb.call(ctx);
      } catch (e) {
        handleError(e, ctx, 'nextTick');
      }
    } else if (_resolve) {
      _resolve(ctx);
    }
  });
  // 因为上一次pending 已经置为true,所以此时不符合条件
  if (!pending) {
    pending = true;
    if (useMacroTask) {
      macroTimerFunc();
    } else {
      microTimerFunc();
    }
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(function (resolve) {
      _resolve = resolve;
    })
  }
}
```

接下去，就到了我们之前讲到的，主script执行完了开始执行microTask, 进入flushSchedulerQueue方法。（tips: 推荐阅读[Tasks, microtasks, queues and schedules]( https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)更好地了解EventLoop)

```javascript
//  p.then(flushCallbacks);
function flushCallbacks () {
  pending = false;
  var copies = callbacks.slice(0);
  callbacks.length = 0;
  for (var i = 0; i < copies.length; i++) {
    copies[i]();
  }
}
// 开始遍历callbacks 执行其中的cb
// 第一个cb 是 flushSchedulerQueue
// 第二个cb 是 我们的 console.log(document.querySelector('.f-error')); 

// 第一个cb里面，watcher.run 最终会进入vdom的diff, 下一篇具体讲细节
function flushSchedulerQueue () {
  flushing = true;
  var watcher, id;
   ...
  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index];
    id = watcher.id;
    has[id] = null;
    watcher.run();
    // in dev build, check and stop circular updates.
    ...
  }

  // keep copies of post queues before resetting state
  var activatedQueue = activatedChildren.slice();
  var updatedQueue = queue.slice();
  resetSchedulerState();

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue);
  callUpdatedHooks(updatedQueue);

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush');
  }
}
// 然后执行了我们的cb, 此时视图已经更新
// <div class='f-error'>5</div>    
```

## 拓展

如果很好地理解了micoTask 与 macroTask之间的关系，那么也能很清楚的理解假设我们写成下面这样, 为什么不行了，自己试试喽！下一篇，会细致讲解vdom diff 的过程~

```javascript
mounted() {
                setTimeout(() => {
                    data.a = 2;
                    data.b = 3;
                    console.log(document.querySelector('.f-error'));
                }, 500);
            }
```


## 参考资料

1.[vue2.0 正确理解Vue.nextTick()的用途](http://www.cnblogs.com/minigrasshopper/p/7879545.html)
2.[从event loop规范探究javaScript异步及浏览器更新渲染时机](https://github.com/aooy/blog/issues/5)
3.[Promise的队列与setTimeout的队列有何关联？](https://www.zhihu.com/question/36972010/answer/71338002)
4.[JavaScript 运行机制详解：再谈Event Loop](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)
5.[Vue源码详解之nextTick：MutationObserver只是浮云，microtask才是核心！](https://github.com/Ma63d/vue-analysis/issues/6)
6.[Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
