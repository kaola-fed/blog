---
title: W3C CSS Transforms摘译
date: 2017-05-31
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
-->

---------
**CSS Transforms**可以对一个元素进行二维平面或三维空间的变换，如translate, rotate, scale和skew等变换。

下面是对W3C官网CSS Transforms模块的部分摘译，为了理解的连贯性，调整了W3C规范中相关章节的顺序。

<!-- more -->

---------
### 5. 二维子集（Two Dimensional Subset）
用户浏览器（UAs）可能不总能渲染出三维变换，那么它们就只能支持该规范的一个二维子集。在这种情况下，三维变换和**transform-style、perspective、perspective-origin**以及**backface-visibility**属性将不被支持。三维相关的变换渲染也不会起作用。
> 对于二维变换情况，矩阵分解采用**Graphics Gems II（Jim Arvo著）**书中**[unmatrix](https://github.com/erich666/GraphicsGems/blob/master/gemsii/unmatrix.c)**算法的二维简化版本。下面是一个二维的3x3变换矩阵，其中6个参数a~f，分别对应二维变换函数**matrix(a, b, c, d, e, f)**的6个参数。
> 
>![二维变换的3x3矩阵](https://www.w3.org/TR/css-transforms-1/3x3matrix.png)
>
>图1 二维变换3x3矩阵
>
>开发者可以很简单的为不支持三维变换的浏览器提供备选变换方案。下面的例子中，**transform**属性有两次赋值定义。第一次定义包括了两个二维变换函数，第二次定义包括一个二维变换和一个三维变换。
> ```css
> div {
>   transform: scale(2) rotate(45deg);
>   transform: scale(2) rotate3d(0, 0, 1, 45deg);
> }
> ```
> 当浏览器支持三维变换时，第二次属性定义将覆盖第一次的定义，当不支持三维变换时，第二次定义将会失效，浏览器会使用第一种定义。
> 
> [常见的变换矩阵以及计算示例](http://www.useragentman.com/blog/2011/01/07/css3-matrix-transform-for-the-mathematically-challenged/)

### 6. 变换渲染模型（The Transform Rendering Model）
当为元素的**transform**属性指定了一个非**none**属性值时，就会在该元素上建立了一个**本地坐标系（local coordinate system）**。从元素最初始的渲染（未指定**transform**属性值）到**本地坐标系**的映射由该元素的**变换矩阵（transformation matrix）**给定。变换是可累积的，也就是说，元素是在它们祖先的坐标系中建立自己的**本地坐标系**。从用户的角度来看，就是一个元素会累积应用所有它祖先**‘transform’**属性设置的变换，以及自身**"transform"**属性设置的变换，规范中将这些变换的累积称为元素**当前变换矩阵（current transformation matrix, CTM）**。

坐标空间是一个有两个轴的坐标系：X轴是水平向右增加，Y轴是垂直向下增加。三维变换函数将这个坐标空间扩展到三维，增加了垂直于屏幕平面一个Z轴，并且指向观察者。

>![初始坐标空间示例](https://www.w3.org/TR/css-transforms-1/coordinates.svg)
>
> 图2 初始坐标空间示例

变换矩阵是基于**'transform'**和**'transform-origin'**属性，按照以下步骤计算而来：
1. 从单位矩阵开始
2. 首先按照**'transform-origin'**属性的X，Y，Z 值进行位移
3. 从左至右依次乘以**'transform'**属性中指定的各个变换函数
4. 最后再按照**'transform-origin'**属性的X，Y，Z 值的负值进行位移

>注意，变换只是影响了元素的显示，不影响元素自身CSS的布局。意味着变化不会影响**getClientRects()**及**getBoundingClientRect()**的值。对于基于CSS盒模型布局定位的元素，如果该元素的**transform**属性为**非none**，则该元素将成为其所有fixed定位的后代元素的**包含块（Containing Block）**。
>
> **[示例1](http://codepen.io/llwanghong/pen/yOvrOZ?editors=1111)，[示例2](http://codepen.io/llwanghong/pen/wGyZxQ?editors=1111)，[示例3](http://codepen.io/llwanghong/pen/YqeMBY)**

### 18. 变换函数的元和派生（Transform function primitives and derivatives）
transform中一些变换函数的效果可以通过更具体的变换函数来实现，比如translate的一些操作可以用translateX来实现。此时称translate为**元变换**，translateX为**派生变换**。

下面列出了所有二维和三维的元变换以及相应的派生变换。
####二维元变换以及相应的派生变换

|元变换 |派生变换                  |
|:-----|:------------------------|
|translate() |translateX(), translateY(), translate()|
|rotate()带三个参数|rotate()带一个或三个参数|
|scale()|scaleX(), scaleY()，scale()|

####三维元变换以及相应的派生变换

|元变换 |派生变换                  |
|:-----|:------------------------|
|translate3d() |translateX(), translateY(), translateZ(), translate()|
|scale3d()|scaleX(), scaleY(), scaleZ(), scale()|
|rotate3d()|rotate(), roateX(), rotateY(), rotateZ()|

>对于同时具有二维和三维元变换的派生变换，具体是使用二维元变换或三维的元变换，是由上下文环境来决定。

### 19. 元变换函数和派生变换函数的插值（Interpolation of primitives and derived transform functions）
两个具有相同数量参数的变换函数，会直接进行数值的插值计算，而不需要转换为相同的元变换。插值计算的结果即是带有相同参数的相同变换。对于**rotate3d(), matrix(), matrix3d(), perspective()**有特殊的插值计算规则。
>例如，对于变换函数**translate(0)**和**translate(100px)**，就是两个具有相同数量参数的相同变换，所以它们会直接进行数值上的插值计算。但是对于变换函数**translateX(100px)**和**translate(100px, 0)**，两个变换既不是相同的变换，使用的参数数量也不同，所以它们就需要先转换为元变换函数，然后才能进行数值上的插值计算。

两个不同的变换，但都是从相同的元变换派生出来的（即相同的派生变换），或者相同的变换，但使用了不同数量的参数，此时两个变换可以进行数值插值计算。需要先将两种变换转换为相同的元变换，然后才能进行数值插值计算。插值计算的结果相当于使用了相同数量参数的相同元变换。
>下面的例子，当div发生鼠标hover事件时，会发生从**translateX(100px)**到**translateY(100px)**的3秒过渡变换。此时两个变换都是从相同的元变换translate()派生的，所以需要先将两个变换转换为**translate()**元变换，然后才能进行数值插值计算。
> ``` css
> div {
>   transform: translateX(100px);
> }
> 
> div:hover {
>   transform: translateY(100px);
>   transition: transform 3s;
> }
> ```
>当发生3秒的transition时，**translateX(100px)**将会转化为**translate(100px, 0)**，**translateY(100px)**会转化为**translate(0, 100px)**，然后两个变换才能进行数值的插值计算。

如果两个变换都可以从同一个二维元变换派生，则都会转换为二维元变换。如果其中一个是或者都是三维变换，则会都转换为三维元变换。
>下面的例子中，一个二维变换函数经过3秒过渡变换到三维变换函数。两个变换函数的公共元变换为**translate3d()**。
> ``` css
> div {
>   transform: translateX(100px);
> }
> 
> div:hover {
>   transform: translateZ(100px);
>   transition: transform 3s;
> }
> ```
>当发生3s的transition时，**translateX(100px)**会转化为**translate3d(100px, 0, 0)**，**translateZ(100px)**会转化为**translate3d(0, 0, 100px)**，然后两个变换才能进行数值的插值计算。

对于**matrix(), matrix3d(), perspective()**三种变换将会首先被转化为4x4的矩阵，然后进行矩阵的插值计算。
对于**rotate3d()**的插值计算，首先会得到两个变换的单位方向向量，如果相等，则可以直接进行数值的插值计算；否则，就需要先将两种变换转化为4x4的矩阵，然后对矩阵进行插值计算。

### 17. 变换的插值（Interpolation of Transforms）

当变换函数之间发生过渡时（比如对transforms施加transition属性），就需要对变换函数进行插值计算。从一个初始的变换（**from-transform**）到一个结束的变换（**to-transform**），如何进行插值计算需要遵循下面的规则。
#### I. 当from-transform和to-transform的值都为none
此时没有必要进行计算，保持原值。

#### ||. 当from-transform和to-transform中有一个的值为none
值为**none**的那个将被替换为**恒等变化（Identity Transform functions）**，然后继续按照下面的规则进行插值计算。
>  **恒等变换（Identity Transform functions）** 就是标准里给出的一系列特殊的变换函数，类似线性代数里面的单位矩阵（Identity Matrix），无论怎么施加多少次变换，都不会改变原有的变换，标准里面给出的恒等变换有**translate(0)、translate3d(0, 0, 0)、translateX(0)、translateY(0)、translateZ(0)、scale(1)、scaleX(1)、scaleY(1)、scaleZ(1)、rotate(0)、rotate3d(1, 1, 1, 0)、rotateX(0)、rotateY(0)、rotateZ(0)、skew(0, 0)、skewX(0)、skewY(0)、matrix(1, 0, 0, 1, 0, 0)**和**matrix3d(1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1)**。一种特殊的情况是**透视(perspective): perspective(infinity)**，此时**M34**的值变得无穷小，因此假定它等于单位矩阵。
>
> 例如，**from-transform**为**scale(2)**，**to-transform**为**none**， 则**none**将会被替换为**scale(1)**，然后继续按照下面的规则进行插值。类似的，如果**from-transform**为**none**，**to-transform**为**scale(2) rotate(50deg)**，则**from-transform**会被替换为**scale(1) rotate(0)**。

#### III. 如果from-transform和to-transform中都使用了相同数量的变换函数，并且各个对应的变化是相同的变换或是从相同的元变换派生的变换。
按照**元变换函数和派生变换函数的插值**里面的步骤，对**from-transform**和**to-transform**对应的变换进行插值。计算出的结果作为最终的变换。
> 例如，**from-transform**为**scale(1) translate(0)**，**to-transform**为**translate(100px) scale(2)**，虽然都是用了scale和translate变换，但是对应的变换既不相同也不是从相同的元变换派生出来的，此时就不能按照本条规则来进行插值计算。 

#### IV. 所有其他情况
**from-transform**和**to-transform**中使用的变换都会被转化为4x4的矩阵，并进行相应的矩阵插值计算。如果**from-transform**和**to-transform**对应的变换矩阵都可以表示成3x2矩阵或者matrix3d矩阵，则元素就按照相应插值计算的矩阵进行变换渲染。在某些特殊的情况下，变化矩阵可能是一个奇异或不可逆矩阵，则元素将不会被渲染。

### 20. 矩阵的插值（Interpolation of Matrices）
当对两个矩阵进行插值时，首先将矩阵分解为一系列变换操作的值，比如对应的translation、rotation、scale和skew的变换矩阵，然后对各个变换操作对应的矩阵进行数值插值计算，最后将各个变换矩阵重新组合为原始矩阵。

>下面的例子中，元素初始变换为rotate(45deg)，当发生hover时，将会在X轴和Y轴移动100像素，并旋转1170度。如果设计人员给出下面的写法，很可能是期望看到元素会顺时针旋转3.5圈（1170度）。
> ``` css
> <style>
> div {
>   transform: rotate(45deg);
> }
> div:hover {
>   transform: translate(100px, 100px) rotate(1215deg);
>   transition: transform 3s;
> }
> </style>
> 
> <div></div>
> ```
>初始变换**‘rotate(45deg)’**和目标变换***'translate(100px, 100px) rotate(1125deg)'*完全不同，按照***变换的插值*** 最后一条规则，两个变换都需要进行矩阵的插值计算。但需要注意，将变换转化为矩阵的过程中，旋转3圈的信息将会丢失掉，所以最终看到的效果只顺时针旋转了半圈（90度）。
>为了达到期望的效果，只需要更新上述变换的写法，使前后两个变换满足***变换的插值*** 的第三条规则。初始变换改为**‘translate(0, 0) rotate(45deg)’**，此时就会进行数值的插值计算，从而不会丢失旋转信息，达到期望的效果。
>
> **[示例4](http://codepen.io/llwanghong/pen/mPXgNj)**

具体的decomposing和recomposing矩阵的算法，见[Graphics Gems II](http://www.amazon.com/Graphics-Gems-II-IBM-No/dp/0120644819)。

译 by Hong