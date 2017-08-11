---
title: Promise：从懵逼开始摸爬滚打
date: 2017-07-30
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

### 前言
是的，Promise长期让我处在懵逼状态。就像高中学不会化学，可是总有人跟你说：化学，不是很简单吗？（再见）

找了几篇文章看，把我拉出懵逼状态的是同事的一句话和一篇文章，特此记录。

#### 话：“.then()参数就是向前面的异步函数注册成功和失败的回调”

#### 文： [你不懂JS: 异步与性能 第三章: Promises](https://github.com/getify/You-Dont-Know-JS/blob/1ed-zh-CN/async%20&%20performance/ch3.md)

## 我们经常写的异步回调出了什么问题？
1. 不好写不好看不好管理——引发对异步回调反感的直接原因。缺乏顺序性和可靠性。
2. callback代码流程是基于副作用的：一个函数会附带调用其他函数。
3. 失去了对主流程的控制权，控制权在那些结果不可预测不可靠的代码块里。

#### 如果异步代码只是告诉我们它是否完成，而能由我们把握流程，拿回调用回调的权利，那事情就会靠谱很多。

## 什么是Promise
### 未来的值
文中的例子我觉得很棒，再此简单重复：


买汉堡 | 请求与回调
---|---
去麦当劳，要了板烧堡套餐，27块给收银员 | 发请求
收银员给我号码牌198，保证我会得到汉堡 | 得到Promise
等待叫号，发了票圈“等会儿吃板烧堡了开心”（我还没有得到汉堡，但是号码牌已经代表了汉堡，相当于一个占位符，与什么时候得到它并没有关系） | 等待并处理其他事物(可以使用这个*未来*的结果)
“198号，您的套餐齐了” | *未来的值*已经返回，许诺换回了真值
“不好意思，我们的板烧堡卖完了” | 返回*未来的值*可能成功也可能失败
WTF, 我的号码永远不被叫到 | 还有可能回调永远被搁置

### 现在和稍后的值
1. 现在的值
    ```
    var x, y = 2;
    console.log(x + y);
    ```
    第二步的打印是基于x,y都已经被解析，是*现在值*
2. 稍后的值
    
    如果x,y解析需要时间，解析之后再进行处理，这就需要回调
    
    ```
    function add(getX, getY, cb){
        var x, y;
        getX(function(xVal){
            x = xVal;
            if (y != undefined) {
                cb(x + y);
            }
        });
        getY(function(yVal){
            y = yVal;
            if (x != undefined) {
                cb(x + y);
            }
        });
    }
    
    add(fetchX, fetchY, function(sum){
        console.log(sum);
    });
    ```
    从外部看，add()不关心现在x,y是否可用，内部将它们都当作未来的值。**泛化**了“现在”和“稍后”：所有的操作都变成异步的
3. Promise值

    用Promise来表达上面的例子
    ```
    function add(xPromise, yPromise){
        return Promise.all([xPromise, yPromise])
        .then(function(values){
            return values[0] + values[1];
        });
    }
    
    // fetchX(), fetchY()分别为它们的值返回一个Promise
    add(fetchX(), fetchY())
    .then(function(sum){    // 链在add()里的then之后
        console.log(sum);
    })
    ```
    上面代码中有两层Promise
    1. fetchX(), fetchY()被直接调用传入add()，返回值都是Promise，表示在*现在*或者*稍后*准备好，都将行为泛化为与时间无关。
    2. add()创建返回Promise。用.then()等待x, y被解析出来
    
    #### 注意：Promise一旦被解析，就是外界不可变的，将解析值传给任何其他模块都是安全的，两个模块之间对解析值的监听互不干扰——不可变性。

4. 完成事件

    ```
    function foo(x){
        // do async things
        return listener;
    }
    
    var evt = foo('result');
    
    evt.on('completion', function(val){
        // do next tings width val
    });
    
    evt.on('failure', function(err){
        // log wrong
    });
    ```
    将程序的控制权归还给调用方代码，一个耗时的函数在完成时发出通知，而不必关心外界的监听和处理
    
5. Promise“事件”
    
    evt的监听能力是一个Promise的类比。

    在基于Promise的方式中，foo()会返回一个Promise实例。.then()注册了异步函数完成和拒绝事件
    
## Promise的信任

Promise是为什么，以及如何被设计为来解决*控制倒转*的信任问题

当你传递一个回调给一个工具函数foo()，它可能：
1. 调用回调太早
2. 调用回调太晚（或者根本不调）
3. 调用回调太多或太少
4. 没能传递必要的参数和环境
5. 吞掉了任何可能发生的错误/异常

### 调的太早
即便是立即完成的Promise也不能被同步地监听
```
new Promise(function(resolve){
    resolve(42);
});
```
即使在Promise调用then的时候已经被解析了，then中的回调也总是被异步地调用

### 调得太晚
在resolve()或reject()被Promise创建机制调用的时候，注册在.then上的监听回调将会自动地被排程。这些被排程好的回调将在下一个异步时刻被可预测地触发。

当一个Promise被解析的时候，所有注册在then()上的回调都会被**立即**、按顺序地在下一个异步机会时被调用，且 没有任何在这些回调中发生的事情可以影响/推迟其他回调的调用

```
var p = new Promise(function(resolve, reject){});

p.then(function(val){
    p.then(function(){
        console.log('c: ', val);    // c不能干扰并优先于b
    });
    console.log('a: ', val);
});

p.then(function(val){
    console.log('b: ', val);
});
// a b c
```
#### Promise排程的怪象
链接在两个分离的Promise上的回调之间的相对顺序是无法可靠预测的。

如果两个Promise p1和p2都准备好被解析了，那么p1.then(); p2.then()应当归结为首先调用p1的回调，然后调用p2的回调。但是有时候情况是下面这样的
```
var p3 = new Promise(function(resolve, reject){
    resolve('A');
});

var p1 = new Promise(function(resolve, reject){
    resolve(p3);
});

var p2 = new Promise(function(resolve, reject){
    resolve('B');
});


p1.then(function(v){
    console.log(v);
});

p2.then(function(v){
    console.log(v);
});
// B A
```
p1不是被一个立即值所解析的，而是由另一个promise p3所解析，p3被一个立即值“A”解析。这种行为将p3展开到p1，但是是异步的。在异步队列中，p1的回调位于p2的回调之后。

这是很微妙的噩梦，不应该依靠任何跨Promise的回调顺序/排程

### 根本不调回调
Promise有几种解决方式
1. 没有任何东西可以阻止Promise通知你它的解析。如果在Promise上注册了完成和拒绝回调，解析完成之后总有一个会被调用
2. 如果Promise本身无论如何没有被解析呢？Promise使用了一个成为“竞赛(race)”的高级抽象
    ```
    // 一个使Promise超时的工具
    function timeoutPromise(delay) {
    	return new Promise(function(resolve,reject){
    		setTimeout(function(){
    			reject( "Timeout!" );
    		}, delay);
    	});
    }
    
    // 为`foo()`设置一个超时
    Promise.race( [
    	foo(),					// 尝试调用`foo()`
    	timeoutPromise( 3000 )	// 给它3秒钟
    ] )
    .then(
    	function(){
    		// `foo(..)`及时地完成了！
    	},
    	function(err){
    		// `foo()`不是被拒绝了，就是它没有及时完成
    		// 那么可以考察`err`来知道是哪种情况
    	}
    );
    ```
    这种超时模式有细节待考察，但是可以确保有delay的信号作为foo的结果，防止其无限地挂起我们的程序
    
### 调太少或太多次

太少：不被调用，上面已经说过
太多：Promise只能被解析一次。如果Promise的创建代码试着调用resolve()或reject()许多次，或者尝试用时调它们两个，Promise只接受第一次解析，而无声地忽略后面的尝试

因为Promise只能被解析一次，所以任何在then()上注册的回调函数将仅被调用一次。

### 没能传入任何参数/环境

Promise可以拥有最多一个解析值（完成或拒绝）。

如果没有用一个值明确地解析它，解析值就是undefined。不管是什么值都会被传入所注册的（且适当的：完成或拒绝）回调中，不管是现在还是未来。

参数，如果你使用多个参数调用resolve()或者reject()，所有第一个参数之外的后续参数都会被无声忽略。如果想传递多个值，必须将它们包装在另一个单独的值array或者object

环境，JS函数总是保持他们被定义时所在作用域的闭包，所以它们理所当然地可以继续访问你提供的环境状态。

### 吞掉所有错误/异常
如果你用一个 理由（也就是错误消息）拒绝一个Promise，这个值就会被传入拒绝回调。

更重要的事：如果在Promise的创建过程中的任意一点，或者在监听它的解析的过程中，一个JS异常错误发生的话，比如TypeError或ReferenceError，这个异常将会被捕获，并且强制当前的Promise变为拒绝。
```
var p = new Promise( function(resolve,reject){
	foo.bar();	// `foo`没有定义，所以这是一个错误！
	resolve( 42 );	// 永远不会跑到这里 :(
} );

p.then(
	function fulfilled(){
		// 永远不会跑到这里 :(
	},
	function rejected(err){
		// `err`将是一个来自`foo.bar()`那一行的`TypeError`异常对象
	}
);
```
如果Promise完成了，在监听过程中（在then上注册的回调上）出现了异常错误也不会丢失，只是处理方式比较特别
```
var p = new Promise( function(resolve,reject){
	resolve( 42 );
} );

p.then(
	function fulfilled(msg){
		foo.bar();
		console.log( msg );	// 永远不会跑到这里 :(
	},
	function rejected(err){
		// 也永远不会跑到这里 :(
	}
);
```
foo.bar()发生的异常看起来被吞掉了，其实并没有。但更深层次的东西出问题了，也就是我们没能成功地监听他。

p.then()调用本身返回另一个Promise，是那个Promise将会被foo.bar()的TypeError异常拒绝。

为什么不能调用rejected(err)？因为违反了“Promise一旦被解析就**不可变**”的原则。p已经完成为值42，所以不能在监听p的解析时发生错误，在稍后又变成一个拒绝。

### 可信的Promise？

基于Promise模式建立信任，还有最后一个细节需要考察。Promise根本没有摆脱回调，只是改变了回调传递的位置。与将一个回调函数传入foo()相反，我们从foo()拿回某些东西，然后我们将回调传入这个东西。

为什么这比直接传入回调可靠？因为 Promise.resolve()

如果传递一个立即的、非Promise的、非thenable的值给Promise.resolve()，你会得到一个用这个值完成的Promise。下面两个Promise p1和p2的行为基本上完全相同：
```
var p1 = new Promise( function(resolve,reject){
	resolve( 42 );
} );

var p2 = Promise.resolve( 42 );
```
如果你传递一个纯粹的Promise给Promise.resolve()你会得到一个完全相同的Promise：
```
var p1 = Promise.resolve( 42 );

var p2 = Promise.resolve( p1 );

p1 === p2; // true
```
更重要的是如果你传递一个非Promise的thenable值给Promise.resolve()，它会试着展开这个值，而且直到抽出一个最终具体的非Promise值之前，展开会继续下去。
```
var p = {
	then: function(cb,errcb) {
		cb( 42 );
		errcb( "evil laugh" );
	}
};

p
.then(
	function fulfilled(val){
		console.log( val ); // 42
	},
	function rejected(err){
		// 噢，这里本不该运行
		console.log( err ); // evil laugh
	}
);
```
p是一个thenable但不是一个纯粹的Promise。我们可以将这两个版本的p传入Promise.resolve()，会得到一个期望的泛化、安全的结果：
```
Promise.resolve( p )
.then(
	function fulfilled(val){
		console.log( val ); // 42
	},
	function rejected(err){
		// 永远不会跑到这里
	}
);
```
Promise.resolve()会接受任何thenable，得到一个真正的、纯粹的Promise，一个可以信任的东西。反正过滤之后是没有坏处的。

如果在调用一个foo()工具，而且不能确定我们能相信它的返回值是一个行为规范的Promise，但我们知道它是一个thenable。Promise.resolve()将会给我们一个可靠的Promise包装器来进行链式调用：
```
// 不要只是这么做：
foo( 42 )
.then( function(v){
	console.log( v );
} );

// 相反，这样做：
Promise.resolve( foo( 42 ) )    // 确保总是返回Promise，将函数调用泛化为一个行为规范的异步任务
.then( function(v){
	console.log( v );
} );
```

## 链式流程
Promise不仅仅是一个单步的 this and then的操作机制，而是构建块。可以将多个Promise串联在一起表达一系列的异步步骤。是一种代码结构和流程。

Promise的核心作用就在于拉平了callback hell的洋葱结构，错误处理和Promise链。规范了异步编程的标准。

Promise的两个固有行为：
1. 在一个Promise上调用then()的时候，都创建并返回一个新的Promise，可以在上面进行链接。
2. 无论从then()中的resolve()中返回什么值，都被作为链接的Promise的完成。

Promise.resolve()传递一个Promise或thenable时，会直接返回纯粹的Promise或是展开收到的thenable的值——并且递归地持续展开thenable。
```
var p = Promise.resolve( 21 );

p.then( function(v){
	console.log( v );	// 21

	// 创建一个promise并返回它
	return new Promise( function(resolve,reject){
		// 使用值`42`完成
		resolve( v * 2 );
		
		// 引入异步也是一样正常工作！
		// setTimeout(function(){
		//     // 使用值`42`完成
		//     resolve(v*2);
		// }, 100);
	} );
} )
.then( function(v){
	console.log( v );	// 42
} );
```

要是Promise链中的某一步出错了会怎样呢？一个错误/异常是基于每个Promise的，意味着在链条的任意一点捕获这些错误是可能的，而且这些捕获操作在那一点上将链条“重置”，使它回到正常的操作上来：
```
// 步骤 1:
request( "http://some.url.1/" )

// 步骤 2:
.then( function(response1){
	foo.bar(); // 没有定义，错误！

	// 永远不会跑到这里
	return request( "http://some.url.2/?v=" + response1 );
} )

// 步骤 3:
.then(
	function fulfilled(response2){
		// 永远不会跑到这里
	},
	// 拒绝处理器捕捉错误
	function rejected(err){
		console.log( err );	// 来自 `foo.bar()` 的 `TypeError` 错误
		return 42;          // 完成下一步（第4步）的promise，如此整个链条又回到完成的状态
	}
)

// 步骤 4:
.then( function(msg){
	console.log( msg );		// 42
} );
```

如果你在一个promise上调用then(..)，而且你只向它传递了一个完成处理器，一个假定的拒绝处理器会取而代之：
```
var p = new Promise( function(resolve,reject){
	reject( "Oops" );
} );

var p2 = p.then(
	function fulfilled(){
		// 永远不会跑到这里
	}
	// 如果忽略或者传入任何非函数的值，
	// 会有假定有一个这样的拒绝处理器
	// function(err) {
	//     throw err;
	// }
);
```
如你所见，这个假定的拒绝处理器仅仅简单地重新抛出错误，它最终强制p2（链接着的promise）用同样的错误进行拒绝。实质上，它允许错误持续地在Promise链上传播，直到遇到一个明确定义的拒绝处理器。完成处理器缺失也是一样的原理，会有一个默认的处理器取而代之直到遇到明确定义的完成处理器

复习一下：使链式流程控制成为可能的Promise固有行为
1. 在一个Promise上的.then()调用会自动生成一个新的Promise并返回。
2. 在完成/拒绝处理器内部，如果你返回一个值或抛出一个异常，新返回的Promise（可以被链接的）将会相应地被解析。
3. 如果完成或拒绝处理器返回一个Promise，它会被展开，所以无论它被解析为什么值，这个值都将变成从当前的.then()返回的被链接的Promise的解析。


## 参考
[你不懂JS: 异步与性能 第三章: Promises](https://github.com/getify/You-Dont-Know-JS/blob/1ed-zh-CN/async%20&%20performance/ch3.md)

[最后谈一次 JavaScript 异步编程](https://zhuanlan.zhihu.com/p/24444262)