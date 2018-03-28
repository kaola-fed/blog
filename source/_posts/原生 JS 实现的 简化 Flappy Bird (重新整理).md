---
title: 原生 JS 实现的 简化 Flappy Bird (重新整理)
date: 2018-03-28
---

## .0 开始之前

之前曾经用 Html5/ JavaScript/ CSS 实现过 2048，用C ocos2d-html5/ Chipmunk写过一个 Dumb Soccer 的对战游戏，但没有使用过原生的 Canvas 写过任何东西，为了加深对 Canvas 的学习，就心血来潮花了将近一天的时间利用原生 Canvas 实现了一个简化版的 Flappy Bird，下面就总结一下开发的过程。

## .1 构造世界

### .1.1 html页面

首先需要在 `index.html` 中加载所需要的脚本，设置 `canvas` 标签，具体代码如下：

```html
<!doctype html>
<html>
<head>
  <style>
    .game_frame {
      margin: 20px auto;
      width: 240px;
      height: 400px;
    }
  </style>
</head>
<body>
  <div class='game_frame'>
      <canvas class='game_box' id='game_box' name='game_box' width='240px' height='400px'></canvas>
  </div>
  <script src="game.js" type="text/javascript"></script>
</body>
</html>
```

利用将 `canvas` 嵌入到一个 `div` 标签中，利用 `div` 标签来控制canvas的位置，目前是将 `canvas` 居中。

需要注意的是，必须要在 `canvas` 标签内部设置样式，否则 `Javascript` 中所绘制的图像的比例很发生严重失真（图像被拉伸，变形）。目前，暂时不考虑自适应屏幕大小的问题，首先把游戏实现了。

另外，脚本的加载最后放在 `body` 末尾，以免脚本获取元素的时候 `html` 页面并未加载完成。其实，更好的方法是利用 `window.onload` 设置页面加载完成后的动作，保证 Javascript 脚本不会在元素未加载完成的时候去读取元素。

### .1.2 game.js脚本

在 `game.js` 中定义一个 `World` 对象，`World` 负责游戏中的所有元素操控（创建、销毁、控制）、动画帧的循环、碰撞检测等工作，是整个游戏运作的引擎。下面是 `World` 中的所有方法和属性：

```javascript
var World = {

  // 保存Canvas
  theCanvas : null,

  // 游戏是否暂停
  pause: false,

  // 初始化并运行游戏
  init : function(){},

  // 重置游戏
  reset: function(){},

  // 动画循环
  animationLoop: function(){},

  // 绘制背景
  BGOffset: 0,    // scroll offset
  backgroundUpdate : function() {},

  // 更新元素
  elementsUpdate: function(){},

  // 碰撞检测
  collisionDectect: function(){},
  hitBox: function ( source, target ) {},
  pixelHitTest: function( source, target ) {},

  // 边界检测
  boundDectect: function(){},

  // 创建烟囱
  pipesCreate: function(){},

  // 清除烟囱
  pipesClear: function(){},

  // 小鸟出界检测
  isBirdOutOfBound: function(callback){},

};
```

通过这些方法，`World` 就可以运行游戏，实现对游戏中“小鸟”和“烟囱”的控制。

### .1.3 游戏的初始化：World.init

游戏通过 `World.init` 来初始化游戏并运行。利用 Html5 canvas 实现的游戏或动画的原理都是一样的，即以特定的时间间隔不断地更新 canvas 画布上的图像以实现动画。因此，游戏初始化时候必须要做的就是下面几件事情：

- 获取 DOM 中的 canvas 元素；
- 通过 canvas元素获取 context;
- 进入动画循环 (animationLoop);

在我们的game.js写下这样的代码:

```javascript
  World.init = function(){
    var theCanvas = this.theCanvas = document.getElementById('game_box');
    this.ctx = theCanvas.getContext('2d');
    this.width = theCanvas.width;
    this.height = theCanvas.height;
    this.bird = null;
    this.items = [];
    this.animationLoop();
  },
```

除了保存 canvas 元素及 context 以外，还将 canvas 画布的长宽也保存下，并创建了 `bird` 对象和 `items` 数组以保存游戏中的元素。然后就进入了 `animationLoop` 这个动画循环。


### .1.4 动画循环

动画循环以特定的时间间隔运行，负责更新游戏中每个元素属性，并将所有元素在canvas中绘制出来。一般游戏会以 `60fps` 或 `30fps` 的帧率运行，显然60fps的帧率需要更大的运算量，而画面也更为流畅，现在就暂时使用 `60fps` 帧率，那么每帧图像的时间间隔为 `1000ms/60 = 16.7ms`。另外，要注意的是元素的绘制顺序，一定要首先将背景绘制出来，然后再绘制其他游戏元素，否则这些元素就会被背景图所覆盖。由于帧间隔比较短，因此 `animationLoop` 中所允许的函数应当尽可能快。下面看看代码：

```javascript
animationLoop: function(){
  // scroll the background
  this.backgroundUpdate();

  // detect elements which is out of boundary
  this.boundDectect();

  // detect the collision between bird and pipes
  this.collisionDectect();

  // update the elements
  this.elementsUpdate();

  // next frame
  if(!this.pause){
    setTimeout(function(){
        World.animationLoop();
    }, 16.7)
  }
}

animationLoop 中的工作顺序是：

- 绘制背景；
- 边界检测，对出界的元素进行处理；
- 碰撞检测，执行相应的处理；
- 绘制游戏中的元素；
- 设置下一帧的定时；

### .1.5 绘制背景
在为游戏世界添加元素以前，先为这个世界创造一个背景。为此，从网上下载了一个 flappy 元素包，里面有一张整合了所有图片元素的 atlas 图集，为了提高游戏的加载速度，决定使用这种图集，尽管显式图片的时候可能稍微麻烦一点，游戏加载速度的提高效果很值得我们这么做。

首先，是 `atlas.png` 图片的加载，一般有两种方式在 `html` 页面中使用标签进行加载，这样图片就会在DOM的构建过程中加载完成，此时我们设置该标签的长宽为 `0`，令其不占据文本流的空间，同时也不会显示出来：

index.html:

```html
<img src="atlas.png" id='atlas' style='visibility:hidden' width="0" height="0">
```

game.js:

```javascript
var image = document.getElementById('atlas');
```

另一种是在Javascript脚本中动态加载，加载时间更加灵活：

```javascript
var image = new image();
image.src = 'atlas.png';
image.onload = function() {
  // wait for the loading
};
```

无论在哪种方式下都要等待图片加载完成才能使用图片，否则会出错。动态加载会使得图片运行更加脚本运行流程变得更复杂，由于在这个游戏中只需要加载一张图片，因此采用第一种方法。

图片加载完成以后，只要在 `backgroundUpadte` 函数中将其绘制出来，即可以顺利完成背景的绘制：

```javascript
backgroundUpdate : function() {
  var ctx = this.ctx;
  ctx.drawImage(image, 0, 0, 288, 512, 0, 0, 288, 512);
},
```

`drawImgae` 的第 2 个到第 5 个参数分别表示，目标图像在图源中的 x 坐标、y 坐标以及图源中图像的宽度和长度，最后四个参数是目标图像在画布中的x坐标、y 坐标以及在画布中的宽度和长度。

然而，一个静态的背景图太简陋了，缺乏活力，所以我们要令背景图卷动起来。实现原理非常简单，只需要将背景图绘制的 x 轴偏移量随着时间改变而改变。但由于我们所拥有的背景图太窄了，需要将其会绘制两次拼接出一张较宽的图片，实现的代码图下所示：

```javascript
backgroundUpdate : function() {
  var ctx = this.ctx;
  this.BGOffset--;
  if(this.BGOffset <= 0) {
    this.BGOffset = 288;
  }
  ctx.drawImage(image, 0, 0, 288, 512, this.BGOffset, 0, 288, 512);
  ctx.drawImage(image, 0, 0, 288, 512, this.BGOffset - 288, 0, 288, 512);
};
```
	
这时，图片还没有开始动，因为我们的世界还没有初始化！！！为了让世界在页面加载完成后初始化，在i `ndex.html` 的 `body` 末尾中嵌入脚本。

```html
<script>
window.onload = function(){
  console.log('start');
  World.init();
}
</script>
```

此时，用浏览器打开页面，就可以看到我们所创造的新世界了。

```html
<div style="padding:atuo"><img class="img-rounded" alt="image" src="/images/posts/2015-07-29-how_to_make_FlappyBird/1.png"></div>
```

## .2 在世界中添加元素
世界创造完成以后，就可以往世界里面添加元素了。游戏世界里面每个元素都可能不一样，有不同的大小、形状、属性、图片，但这些元素也有一些共性，如它们都需要有记录大小、位置、移动速度的属性，还需要有在元素中渲染该图片的方法。这里有些属性和方法是特有的，如大小属性，渲染方法，但同时这些元素也有共有的属性，如设置位置、速度的方法等。为此，我们将创建一个名Item函数对象，利用这个函数对象的prototype来保存一些公有的方法和属性，再创建`Bird` 类和 `Pipe` 类来创建构造 `bird` 和 `pipe` 对象。

### .2.1  基本元素：Item类
世界中每个元素都需要有的基本属性是：大小、位置、速度、重力（重力加速度），而这些属性每个具体的元素对象都可能不相同，因此它们不设置在 `prototype` 上，只在对象本身上创建。而 `prototype` 上有的是设置这些属性的方法，还有一个叫 `generateRenderMap` 的方法。这个方法是用来生成用于像素碰撞检测的数据的，暂时先不写。

```javascript
/*
* Item Class
*  Basic tiem class which is the basic elements in the game world
* @param draw, the context draw function
* @param ctx, context of the canvas
* @param x, posisiton x
* @param y, posisiton y
* @param w, width
* @param h, height
* @param g, gravity of this item
*/
var Item = function(draw, ctx, x, y, w, h, g) {
  this.ctx = ctx;
  this.gravity = g || 0;
  this.pos = {
      x: x || 0,
      y: y || 0
  };
  this.speed = {
      x: 0,        // moving speed of the item
      y: 0
  }
  this.width = w;
  this.height = h;
  this.draw = typeof draw == 'function' ? draw : function(){};
  return this;
};

Item.prototype = {
  // set up the 'draw' function
  setDraw : function(callback) {
    this.draw = typeof draw == 'function' ? draw : function(){};
  },

  // set up the position
  setPos : function(x, y) {
      // Handle: setPos({x: x, y: y});
      if(typeof x == 'object') {
          this.pos.x = typeof x.x == 'number' ? x.x : this.pos.x;
          this.pos.y = typeof x.y == 'number' ? x.y : this.pos.y;
      // Handle: setPos(x, y);
      } else {
          this.pos.x = typeof x == 'number' ? x : this.pos.x;
          this.pos.y = typeof y == 'number' ? y : this.pos.y;
      }
  },

  // set up the speed
  setSpeed : function(x, y) {
    this.speed.x = typeof x == 'number' ? x : this.speed.x;
    this.speed.y = typeof y == 'number' ? y : this.speed.y;
  },

  // set the size
  setSize : function(w, h) {
    this.width = typeof width == 'number' ? width : this.width;
    this.height = typeof height == 'number' ? height : this.height;
  },

  // update function which ran by the animation loop
  update : function() {
    this.setSpeed(null, this.speed.y + this.gravity);
    this.setPos(this.pos.x + this.speed.x, this.pos.y + this.speed.y);
    this.draw(this.ctx);
  },

  // generate the pixel map for 'pixel collision dectection'
  generateRenderMap : function( image, resolution ) {}
}
```

内部属性的初始化有Item函数实现，里面有设置简单的默认初始化。更加完善的初始化方法是首先检测输入参数的类型，然后再进行初始化。

`gravity` 影响的是垂直方向的速度，即 `speed.y`。而 `speed` 在每一次元素更新（动画循环）的时候，影响 `pos` 属性，从而改变元素的位置。

`update` 这个公有方法通过 `setSpeed`、`setPos` 改变元素的速度和位置，并调用 `draw` 方法来将元素绘制在 `canvas` 画布上。

元素初始化的时候，必须从 `World` 中获得 `draw` 方法，否则元素的图像是不会绘制到 `canvas` 画布上的。而绘制所需要的 `context` 也是从 `World` 中获取的，在初始化的时候获取，并保存到内部变量中。

####添加Item对象

通过 `Item` 来创建一个对象，就可以向世界中添加一个元素，在 `World.init` 中添加代码：

```javascript
World.init = function(){
  //  ...

  var item = new Item(function(ctx){
    ctx.fillStyle = "#111111";
    ctx.beginPath();
    ctx.arc(this.pos.x, this.pos.y, this.width/2, 0, Math.PI*2, true);
    ctx.closePath()
    ctx.fill()
  }, this.ctx, 50, 50, 10, 10, 0.2);
  this.items.push(item);    // 将元素放入到管理列表中

  // ...
}
```

通过上述代码，就可以往World中添加一个圆点。但此时世界中仍然不会显示圆点，那是因为 `World.elementsUpdate` 还没有实现。该方法需要遍历世界中的所有元素，调用元素的 `update` 方法，通过元素的 `update` 方法调用`draw` 方法从而实现元素在画布上的绘制。

```javascript
World.elementsUpdate = function() {
  // update the pipes
  var i;
  for(i in this.items) {
    this.items[i].update();
  }
}
```

刷新页面之后，就会看到一个小圆点在做自由落体运动。

```html
<div style="padding:atuo"><img class="img-rounded" alt="image" src="/images/posts/2015-07-29-how_to_make_FlappyBird/2.png"></div>
```

### .2.2 继承：extend函数

在创建其他类型的元素前，先来看看要如何实现类的继承。

####为何要使用继承？

在游戏中的元素都存在共性，它们都有记录大小、位置、速度的属性，也都需要有设置大小、位置、速度的属性，还必须要有一个提供给 `World.elementsUpdate` 方法调用的更新元素属性、在画布上绘制元素图像的接口。通过类的继承，在创建不同类型的时候就可以将来自早已定义好的基类——Item类——的属性或方法继承下来，简化了类的创建，同时也节省了实例占用的空间。

####如何实现类的继承？

要实现类的继承，最主要的是应用了 `constructor` 和 `prototype`。在子类构造器函数中，通过调用 `Parent.constructor.call(this)` 就可使用基类构造器为子类构造内部属性和方法；通过 `Child.prototype = Parent.prototype` 就可以继承基类的 `prototype`，这样子类的实例对象就可以直接调用基类 `prototype` 上的代码。JavaScript 里实现类继承的方法非常多，不同的方法能够产生不同的效果，更多详细的说明请翻阅相关的参考书，如 `《JavaScript面向对象编程指南》`，`《JavaScript设计模式》` 等。

####extend函数

在这里，我们采用一个简单的 `extend` 函数来实现继承。

```javascript
/*
* for deriving a new Class
* Child will copy the whole prototype the Parent has
*/
function extend(Child, Parent) {
  var F = function(){};
  F.prototype = Parent.prototype;
  Child.prototype = new F();
  Child.prototype.constructor = Child;
  Child.uber = Parent.prototype;
}
```


这个函数干了下面一些事情：

- 创建一个空函数对象F作为中间变量；
- 中间变量获取Parent的prototype；
- 子类从中间变量中继承原型并更新原型中的构造器，此时子类的原型和基类的原型虽然包含相同的属性和方法，但是已经两个独立的原型了，不会相互影响；
- 最后创建一个内部uber变量来引用Parent原型；
　
该方法参考自 `《JavaScript面向对象编程指南》`。使用这个方法，会复制基类原型链并继承之，并不会继承基类的内部属性和方法 `this.xxxx`。这样做的原因是，尽管子类和基类可能会有共同的元素，但是初始化构造时要执行的参数不一样，有些元素可能拥有更多内部属性，有些内部属性可能已经被一些子类元素抛弃了，但原型链上的公有方法则是子类想继承的。

利用内部 `uber` 属性引用基类原型链的原因在于，子类有可能需要重载原型链上的公有方法，这样就会把原有继承而来的方法覆盖掉，但有时又需要调用基类原有的方法，因此就利用内部属性 `uber` 保留对基类原型链的引用。


### .2.3 主角：Bird类

这个世界的主角是 `Bird`，尽管它是主角，但它也是这个世界的元素之一，与 `Item` 类一样拥有记录大小、位置、速度的内部属性，它将会继承来自 `Item` 类原型链上设置内部属性的方法，当然也有一个更重要的与众不同的 `fly` 方法。

但首先，要获取 `atlas` 中小鸟的图源参数。为了方便起见，创建一个对象将其记录下来。

```javascript
var atlas = {};
atlas.bird =[
  { sx: 0, sy: 970, sw: 48, sh: 48 },
  { sx: 56, sy: 970, sw: 48,sh: 48 },
  { sx: 112, sy: 970, sw: 48, sh: 48 }
];
```

`atlas.bird` 中记录了 `atlas` 图左下角三只黄色小鸟的信息，分别是表示三种状态。目前暂时 `atlas.bird[1]` 展示小鸟的滑翔状态。

```javascript
/*
* Bird Class
*
* a sub-class of Item, which can generate a 'bird' in the world
* @param ctx, context of the canvas
* @param x, posisiton x
* @param y, posisiton y
* @param g, gravity of this item
*/
var Bird = function(ctx, x, y, g) {
  this.ctx = ctx;
  this.gravity = g || 0;
  this.pos = {    x: x || 0,
                  y: y || 0
              };
  this.depos = {    x: x || 0, // default position for reset
                  y: y || 0
              };
  this.speed = {    x: 0,
                  y: 0
  }
  this.width = atlas.bird[0].sw || 0;
  this.height = atlas.bird[0].sh || 0;

  this.pixelMap = null; // pixel map for 'pixel collistion detection'
  this.type = 1; // image type, 0: falling down, 1: sliding, 2: raising up
  this.rdeg = 0; // rotate angle, changed along with speed.y
  
  this.draw = function drawPoint() {
    var ctx = this.ctx;
    ctx.drawImage( // draw the image
      image, atlas.bird[this.type].sx, atlas.bird[this.type].sy, this.width, this.height,
      this.pos.x, this.pos.y, this.width, this.height
    );
  };
  return this;
}

// derive fromt the Item class
extend(Bird, Item);

// fly action
Bird.prototype.fly = function(){        
  this.setSpeed(0, -5);
};

// reset the position and speed 
Bird.prototype.reset = function(){
  this.setPos(this.depos);
  this.setSpeed(0, 0);
};

// update the bird state and image
Bird.prototype.update = function() {    
  this.setSpeed(null, this.speed.y + this.gravity);
  this.setPos(this.pos.x + this.speed.x, this.pos.y + this.speed.y);    // update position
  this.draw();
}
```

`Bird` 的构造器基本上 `Item` 一样，特别的在于它的宽度和长度由图像的大小决定，而它在内部定制了 `draw` 方法，用于将小鸟的图像绘制到画布上。`draw` 方法中调用了跟绘制背景时一样的 `drawImage` 方法，只不过图源信息从 `atlas.bird` 中获取，暂时默认小鸟以滑翔状态显示。

`Bird` 多了两个方法，分别是 `reset` 和 `fly`。`reset` 用于重置小鸟的位置和速度；而 `fly` 则是给小鸟设置一个向上的速度 `speed.y`，让其向上飞一下。

此外 `Bird` 还“重载”了 `update` 方法。现在看来，这个方法跟 `Item` 中的没有什么区别，但由于它是世界的主角，后来会为它添置更多的动画等，所以预先在这里“重载”了。

要注意的是，`extend` 函数需要在定义 `Bird` 的 `prototype` 方法之前，否则新定义的方法会被 `Item` 类的 `prototype` 覆盖掉。

#### 在世界中添加小鸟

现在，就可以往世界里添加小鸟了，在 `World.init` 中添加如下代码：

```javascript
World.init = function(){
  ...
  this.bird = new Bird(this.ctx, this.width/10, this.height/2, 0.15);
  ...
}
```

此时，类封装的好处就显示出来了，由于 `Bird` 类已经将小鸟的构造过程封装好，创建小鸟实例的时候只需要传入 `context` 并设置位置及重力参数，创建过程变得极为简便。

除此以外，还需要在 `World.elementsUpdate` 中添加代码，让动画循环把小鸟图像绘制在画布上：

```javascript
World.elementsUpdate = function(){
  // update the pipes
  var i;
  for(i in this.items) {
      this.items[i].update();
  }

  // update the bird
  this.bird.update();
};
```

刷新页面，就可以在游戏世界中看到一只只会自由落体的小鸟了。

#### 控制小鸟

一只只会自由落体的小鸟显然是不好玩的，为此要在世界中添加控制小鸟的方法。简单地，我们让键盘按下任何键都会使小鸟往上飞，需要在 `World.init` 中添加代码：

```javascript
World.init = function(){
  // ...
  (function(that){
    document.onkeydown = function(e) {
            that.bird.fly();
    };
  })(this);
  // ...
}
```

通过 `document.onkeydown` 设置按键按下时的回调函数，进而调用 `bird.fly` 使其往上飞。在这里使用了闭包来传递 `World` 的 `this` 对象，因为执行回调的时候上下文会改变，需要使用闭包来获取定义回调函数时的上下文中的对象。重新刷新页面，按下键盘上任意一个按键，就可以让小鸟往上飞了。a

```javascript
<div style="padding:atuo"><img class="img-rounded" alt="image" src="/images/posts/2015-07-29-how_to_make_FlappyBird/3.png"></div>
```

### .2.4 反派：Pipe类

有了主角，就要有反派，世界才会充满乐趣。而在这里，我们的反派就是那些长长短短的烟囱们。我们用同样的方法来构造一个 `Pipe` 类。

首先还是得有图源的参数，采用与 `Bird` 类似的方式来保存，在这里只选用图集中绿色的两根烟囱。

```javascript
atlas.pipes = [
  { sx: 112, sy: 646, sw: 52, sh: 320 },    // face down
  { sx: 168, sy: 646, sw: 52, sh: 320 }    // face up
]
```
Pipe类代码：

```javascript
/*
*                        Pipe Class
*
*    a sub-class of Item, which can generate a 'bird' in the world
*@param ctx, context of the canvas
*@param x, posisiton x
*@param y, posisiton y
*@param w, width
*@param h, height
*@param spx, moving speed from left to right
*@param type, choose to face down(0) or face up(1)
*/
var Pipe = function(ctx, x, y, w, h, spx, type) {
  this.ctx = ctx;
  this.type = type || 0;
  this.gravity = 0; // the pipe is not moving down
  this.width = w;
  this.height = h;
  this.pos = {
    x: x || 0,
    y: y || 0
  };
  this.speed = {
    x: spx || 0,
    y: 0
  }

  this.pixelMap = null; // pixel map for 'pixel collistion detection'
  
  this.draw = function drawPoint(ctx) {
    var pipes = atlas.pipes;
    if(this.type == 0) { // a pipe which faces down, that means it should be on the top
      ctx.drawImage(
        image, pipes[0].sx, pipes[0].sy + pipes[0].sh - this.height, 52, this.height,
        this.pos.x, 0, 52, this.height
      );
    } else { // a pipe which faces up, that means it should be on the bottom
      ctx.drawImage(
        image, pipes[1].sx, pipes[1].sy, 52, this.height,
        this.pos.x, this.pos.y, 52, this.height)
      ;
    }

  return this;
}

// derived from the Item class
extend(Pipe, Item);
```

`Pipe` 类的定义同样不复杂，由于 `Pipe` 的长度会随机变化，而且有面朝上和面朝下两种形态，因此构造器保留长宽参数并设置有类型参数。在这里假定 `Pipe` 不能上下移动，因此 `speed.y` 设置为 `0`，同时只能初始化 `Pipe` 在 x 轴上的移动速度。

`Pipe` 的 `draw` 方法也使用与 `Bird` 类似的方式，区别在于要根据烟囱类型来选择绘制方式和参数。

#### 在世界中随机地添加烟囱

为了给世界增加趣味性，需要随机地在世界中创建烟囱，为此在World.pipesCreate写下代码：

```javascript
pipesCreate: function(){
  var type = Math.floor(Math.random() * 3);
  var that = this;
  // type = 0;
  switch(type) {
    // one pipe on the top
    case 0: {
      var height = 125 + Math.floor(Math.random() * 100);
      that.items.push(new Pipe(that.ctx, 300, 0, 52, height, -1, 0));                        // face down
      break;
    }
    // one pipe on the bottom
    case 1: {
      var height = 125 + Math.floor(Math.random() * 100);
      that.items.push(new Pipe(that.ctx, 300, that.height - height, 30, height, -1, 1));        // face up
      break;
    }
    // one on the top and one on the bottom
    case 2: {
      var height = 125 + Math.floor(Math.random() * 100);
      that.items.push(new Pipe(that.ctx, 300, that.height - height, 30, height, -1, 1) );    // face up
      that.items.push(new Pipe(that.ctx, 300, 0, 30, that.height - height - 100, -1, 0) );    // face down
      break;
    }
  }
}
```

每创建一个 `Pipe` 实例，都需要将其存入 `World.items` 中，由 `World.elementsUpdate` 来对所有的烟囱进行统一更新和绘制。

完成随机创建方法以后，在 `World.init` 中添加定时器来调用创造烟囱的方法：

```javascript
World.init = function(){
  (function(that){
    setInterval(function(){
      that.pipesCreate();
    }, 2000)
  })(this);
}
```

同样的，需要使用闭包了传递 `World` 本身，否则定时函数无法获取 `this.pipesCreate` 方法。由于在 `pipesCreate` 中创建的 `Pipe` 都设置的固定的初始位置，`Pipe` 以固定的速度向左移动，因此 `Pipe` 实例之间的距离就通过定时器的时间间隔来控制。当然，时间间隔越短，烟囱间距离就越窄，那么游戏的难度就加大了。

#### 处理出界的元素

现在 `World.pipesCreate` 会不断创建烟囱对象并保存到 `World.items` 中，即使出界了也没有做任何处理，那么不再出现的烟囱对象会一直累积下来，一点点地消耗内存。因此，需要对出界的烟囱来进行处理。

而对于 `Bird` 实例而言，`Bird` 掉落到世界下部时，如果没有任何操作，那么小鸟就会永远地掉落下去，很难再飞上来了。因此必须对小鸟的出界进行检测和处理。

要记住，`canvas` 画布中，原点在左上角，X 轴方向从左向右，而 Y 轴方向从上向下。

```html
<div style="padding:atuo"><img class="img-rounded" alt="image" src="/images/posts/2015-07-29-how_to_make_FlappyBird/4.png"></div>
```

检测及处理出界元素的代码：

```javascript
// boundary dectect
World.boundDectect = function(){
  // the bird is out of bounds
  if(this.isBirdOutOfBound()){
      this.bird.reset();
      this.items = [];
  } else {
      this.pipesClear();
  }
},

// pipe clearance
// clear the pipes which are out of bound
World.pipesClear = function(){
  var it = this.items;
  var i = it.length - 1;
  for(; i >= 0; --i) {
    if(it[i].pos.x + it[i].width < 0) {
      it = it.splice(i, 1);
    }
  }
};

// bird dectection
World.isBirdOutOfBound = function(callback){
  if(this.bird.pos.y - this.bird.height - 5 > this.height) {    // the bird reach the bottom of the world
      return true;
  }
  return false;
};
```

当检测到烟囱的位置越过了画面的左界的时候，就将该烟囱实例清除。这里使用了 `Array.splice` 方法，要注意的是，移除 `Array` 的时候会改变 `Array` 的长度和被移除元素后面元素的位置，因此在这里使用从后往前的遍历方式。

当小鸟位置超过画面下界时，利用 `World.items = []` 清除所有烟囱，并重置小鸟的位置。

再刷新一下页面，试着任由小鸟自由落体至画面底部，就会看到小鸟会被重置。

##.3 碰撞检测

到目前为止，游戏的基本元素都已经添加完毕了，但你会发现一个问题：无敌的小鸟像超级英雄一样穿越所有烟囱，反派仅仅起到装饰的作用。这是因为，我们还没有添加碰撞检测功能。

### .3.1 边框碰撞检测

尽管每个元素的形状都不一定是方方正正的，但是我们在创建元素的时候都为这个元素设置了长度和宽度，利用这个隐藏的边框，就可以实现边框检测。检测方法非常简单，只要检测两个框是否有重复部分即可，实现手段就是坚持两个框的边界距离是否相互交错。类似的算法在leetcode上面有算法题，都可以用来借鉴。

注意的是 `pos` 表示的是边框左上角的坐标。

```javascript
World.hitBox = function ( source, target ) {
  return !(
    ( ( source.pos.y + source.height ) < ( target.pos.y ) ) ||
    ( source.pos.y > ( target.pos.y + target.height ) ) ||
    ( ( source.pos.x + source.width ) < target.pos.x ) ||
    ( source.pos.x > ( target.pos.x + target.width ) )
  );
}
```

边框检测极其简单且快速，但是其效果是，小鸟还没有碰到烟囱就就会判定为碰撞已发生。那是因为小鸟的图像不仅没有填满这个边框，还拥有不规则的形状。因此边框检测只能用做初步的碰撞检测。

### .3.2 像素碰撞检测

根据精细的检测方式是对两个元素的像素进行检测，判断是否有重叠的部分。

```html
<div style="padding:atuo"><img class="img-rounded" alt="image" src="/images/posts/2015-07-29-how_to_make_FlappyBird/5.jpg"></div>
```

但是，像素碰撞检测需要遍历元素的像素，运算速度比较慢，如果元素较多，那么帧间隔时间内来不及完成检测任务。为了减少碰撞检测的耗时，可以先利用边框检测判断那些元素之间有可能发生碰撞，对可能发生碰撞的元素使用像素碰撞检测。

```javascript
// dectect the collision
Wordl.collisionDectect = function(){ 3     for(var i in this.items) {
    var pipe = this.items[i];
    if(this.hitBox(this.bird, pipe) && this.pixelHitTest(this.bird, pipe)) {
      this.reset();
      break;
    }
  } 
};
```



游戏里，只需要检测小鸟和烟囱之间的碰撞，因此只需要拿小鸟和烟囱逐个做检测，先进行边框碰撞检测，然后进行像素碰撞检测，以提高运算效率。检测到碰撞以后，调用World.reset来重置游戏或进行其他操作。

	World.reset = function(){
	    this.bird.reset();
	    this.items = [];
	}

下面来看看像素碰撞检测。尽管减少了像素碰撞检测的调用次数，但每次像素碰撞检测的运算量仍然非常大。将如两个图像各包含 200 个像素，那么逐个像素进行比较就需要 40000 次运算，显然效率低下。

仔细想想，图像发生碰撞时只有边缘发生碰撞就可以。那么只要记录图像的边缘数据，然后检查两幅图像边缘是否重合判别碰撞，降低需要运算的像素点数量从而降低运算量。然而边缘检测及边缘重合的算法并不简单，当中会出现许多问题。

在这里，我们打算将边框碰撞检测应用到像素碰撞检测当中。首先，需要将原图像进行稀疏编码，即将原图像的分辨率降低，这样就相当于将一个 1px 的像素点编程一个由更多像素点组成的方形小框。

```html
<div style="padding:atuo"><img class="img-rounded" alt="image" src="/images/posts/2015-07-29-how_to_make_FlappyBird/6.jpg"></div>
```

然后，把这些小框的数据保存到每个元素的 `pixelMap` 中，这样一来，在进行碰撞检测的时候，就可以看元素图像看作是多个边框组合而成的图像，我们要做的只需要检测组成两个元素的小框之间有没有发生碰撞。

####像素检测算法的实现

```javascript
World.pixelHitTest = function( source, target ) {    
  // Loop through all the pixels in the source image
  for( var s = 0; s < source.pixelMap.data.length; s++ ) {
    var sourcePixel = source.pixelMap.data[s];

    // Add positioning offset
    var sourceArea = {
      pos : {
          x: sourcePixel.x + source.pos.x,
          y: sourcePixel.y + source.pos.y,
      },
      width: target.pixelMap.resolution,
      height: target.pixelMap.resolution
    };

    // Loop through all the pixels in the target image
    for( var t = 0; t < target.pixelMap.data.length; t++ ) {
      var targetPixel = target.pixelMap.data[t];
      // Add positioning offset
      var targetArea = {
        pos:{
            x: targetPixel.x + target.pos.x,
            y: targetPixel.y + target.pos.y,
        },
        width: target.pixelMap.resolution,
        height: target.pixelMap.resolution
      };
      /* Use the earlier aforementioned hitbox function */
      if( this.hitBox( sourceArea, targetArea ) ) {
        return true;
      }
    }
  }
};
```

resolution 是指像素点放大的比例，如果为 4，则是将 1px 放大为 4X4 px 大小的边框。该算法是从原始的 pixelMap 中读取每个小框，并构造一对 `Area` 对象（方形边框）传递给 `World.hitBox` 方法进行边框碰撞检测。

####pixelMap的构造

而 `pixelMap` 的构造则需要用到 `context.getImageData` 方法。

本地环境下，`getImageData` 在IE 10或firefox浏览器下能够顺利运行，如果是在Chrome下则会产生跨域问题。除非使用 HTTP 服务器来提供 web 服务，否则需要更改 `chrome` 的启动参数--allow-file-access-from-files 才能够使用 `getImageData` 来获取本地图片文件的数据。

`getImageData` 是从 canvas 画布的指定位置获取指定大小的图像数据，因此如果存在背景的话，背景的图像数据也会被截取。因此需要创建一个临时的 `canvas DOM` 对象，在上面绘制目标图像，然后再从临时画布上截取图像信息。

`Bird` 类的 pixelMap：（在 `Bird` 类的 `draw` 方法中添加代码）

```javascript
var Bird = function(){
  // ...
  this.draw = function(){
    // ...
    // the access the image data using a temporaty canvas
    if(this.pixelMap == null) {
      var tempCanvas = document.createElement('canvas');        // create a temporary canvas
      var tempContext = tempCanvas.getContext('2d');
      tempContext.drawImage(image, atlas.bird[this.type].sx, atlas.bird[this.type].sy, this.width,  this.height,
                                    0, 0,  this.width,  this.height);    // put the image on the temporary canvas
      var imgdata = tempContext.getImageData(0, 0, this.width, this.height); // fetch the image from the temporary canvas
      this.pixelMap = this.generateRenderMap(imgdata, 4);        // using the resolution the reduce the calculation
    }
  // ...
  }
  // ...
}
```

`Pipe` 类的 `pixelMap`：（类似地在 `draw` 方法中添加代码）

```javascript
var Pipe = function() {
  // ...
  this.draw = function() {
    // ...
    if(this.pixelMap == null) {        // just create the pixel map from a temporary canvas
      var tempCanvas = document.createElement('canvas');
      var tempContext = tempCanvas.getContext('2d');
      if(this.type == 0) {
          tempContext.drawImage(image, 112, 966 - this.height, 52, this.height, 0, 0, 52, this.height);
      } else {                    // face up
          tempContext.drawImage(image, 168, 646, 52, this.height, 0, 0, 52, this.height);
      }
      var imgdata = tempContext.getImageData(0, 0, 52, this.height);
      this.pixelMap = this.generateRenderMap(imgdata, 4);
    }
    // ...
  }
  // ...
}
```

无论是 `Bird` 类还是 `Pipe` 类，都使用从 `Item` 类中继承而来的 `generateRenderMap` 方法

```javascript
// generate the pixel map for 'pixel collision dectection'
//@param image, contains the image size and data
//@param reolution, how many pixels to skip to gernerate the 'pixelMap'
Item.generateRenderMap = function( image, resolution ) {
  var pixelMap = [];

  // scan the image data
  for( var y = 0; y < image.height; y=y+resolution ) {
    for( var x = 0; x < image.width; x=x+resolution ) {
      // Fetch cluster of pixels at current position
      // Check the alpha value is above zero on the cluster
      if( image.data[4 * (48 * y + x) + 3] != 0 ) {
          pixelMap.push( { x:x, y:y } );
      }
    }
  }
  return {
    data: pixelMap,
    resolution: resolution
  };
}
```

resolution 决定小框的大小，也决定了每行每列跳过的像素点数量。当检测到一个像素点的 alpha 通道值不为0，就将其保存到 pixelMap 中即可。

此时刷新一下页面，你会发现小鸟再也不是无敌的了。

## .4 增添动画效果

基本的 Flappy Bird 基本完成了，然而小鸟只能以滑翔的姿态运动，没有有扇动翅膀的动作，显得没有生气。为此，我们可以给 `Bird` 类添加动画效果，让小鸟向上飞的时候会扇动翅膀，同时头部朝上；向下坠落的时候则头部朝下，以俯冲的姿态运动。

这时候，之前提到了“重载” `Bird.update` 方法的意义就来了。

```javascript
	// update the bird state and image
	Bird.prototype.update = function() {    
	    this.setSpeed(null, this.speed.y + this.gravity);
	    
	    if(this.speed.y < -2) {            // raising up
	        if(this.rdeg > -10) {
	            this.rdeg--;            // bird's face pointing up
	        }
	        this.type = 2;
	    } else if(this.speed.y > 2) {    // fall down
	        if(this.rdeg < 10) {
	            this.rdeg++;            // bird's face pointing down
	        }
	        this.type = 0;
	    } else {
	        this.type = 1;
	    }
	    this.setPos(this.pos.x + this.speed.x, this.pos.y + this.speed.y);    // update position
	    this.draw();
  }
```

当小鸟速度 `speed.y` 小于 -2（飞起来初速度是 -4 ）时，就减少其旋转角度 rdeg 让其脸逐渐朝上并更改图片显示状态为 2（向下拍翅膀）；

当小鸟速度 `speed.y` 大于 2（下落时速度 >0 ）时，就增加其旋转角度 rdeg 让其脸逐渐朝下并更改图片显示状态为 0（翅膀上拉，成俯冲姿态）；

速度在 -2 和 2 之间时，就维持滑翔状态。

###旋转小鸟

增加更新小鸟属性的方法后，还需要更新 `Bird.draw`，否则旋转的效果是不会显示出来的。

```javascript
var Bird = function() {
  // ...
  this.draw = function() {
    // ...
    ctx.save(); // save the current ctx
    ctx.translate(this.pos.x, this.pos.y); // move the context origin 
    ctx.rotate(this.rdeg*Math.PI / 180); // rotate the image according to the rdeg
    ctx.drawImage( // draw the image
      image, atlas.bird[this.type].sx, atlas.bird[this.type].sy, this.width, this.height,
      0, 0, this.width, this.height
    );
    ctx.restore(); // restore the ctx after rotation
    //...
  };
  //...
};
```

使用 `context.rotate` 旋转图像前，需要先保存原来的 `context` 状态，将画布的原点移动到当前图像的坐标，接着根据 `Bird.rdeg` 旋转图像，然后绘制图像。使用 `drawImage` 绘制图形时，需要将目标坐标改为（0,0），因为此时画布原点坐标以及移动到了 `Bird.pos` 上的。绘制完成后恢复 `context` 的状态。这样，就能够实现小鸟的身体倾斜了。

刷新一下页面，小鸟的动画特效就完成了。

至于碰撞特效之类的动画特效，就由大家自己自由发挥了，在这里，仅仅将最简单的 Flappy Bird 游戏功能实现。

## .5 总结

游戏耗时一天完成，期间在碰撞检测部分花费了大量的时间查阅资料和测试，找到合适的方法后又在getImageData折腾了好久解决本地调试的跨域问题和截取不含背景的图像数据的问题。但总算是完成了这个简化版的游戏。

目前，简化版本没有任何菜单、按键、显示文本等，日后会考虑继续把这部分功能完善。此外，有部分代码写的并不够精简，结构也不够清晰，编程技术有待磨练。

跟之前使用 Cocos2d-html 的开发经历对比，不使用任何框架的开发难度提高了不少，尤其是在动画循环、元素绘制、碰撞检测这些部分花了不是功夫。

[GitHub源码地址](https://github.com/elcarim5efil/HtmlGames/tree/dev_a_star/flap)

## 参考

[pixel-accurate-collision-detection-with-javascript-and-canvas](http://benjaminhorn.io/code/pixel-accurate-collision-detection-with-javascript-and-canvas)