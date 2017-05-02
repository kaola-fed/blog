---
title: Regular实现动画的痛点
date: 2017-03-03
---

## Regular实现动画的痛点
### 一个需求
PC首页增加一个模块如下图：左侧区域3个强推品牌轮播，右侧10个个性化推荐品牌展示，右上角换一批，点击发请求获取10个新品牌展示。

<!-- more -->

左侧我们有轮播组件可以直接拿来用，右侧的换一批获取到新数据后直接替换老数据，触发脏检查regular自动刷新右侧列表，实现起来还是比较简单。
![需求图](http://haitao.nos.netease.com/269e605c-de3b-4e7a-ae7a-eb5e254f6be4.png)
### 优化需求
交互说搞点动画吧，我们网站太缺了。我很同意，所以乐意花点时间做。他说效果参照某猫，如下：
![某猫翻牌效果](http://haitao.nos.netease.com/39974273-d4f4-4579-a51a-c67b019c5681.gif)
确实比较酷炫，值得借鉴。
### 思路
实现这个效果主要有3个需要思考的点：
- 单个品牌翻转动画的实现
- 全部品牌翻转的时序控制
- 如何在翻转的过程中替换数据
#### 单个翻转
看到这类dom动画是不是第一反应animation、transition、transform等？翻转后品牌变了，是否要两个节点互相backface-visibility:hidden;？又考虑到PC首页要保证较好的低版本浏览器兼容，然后考虑降级方案。。。

其实此处有更好的方案。这个是比较简单的翻转效果，其实用width也能实现。通过将品牌节点的width从100%->0->100%，该节点会先压缩到0宽不可见，然后再展开到原宽度，视觉上看和rotateY并无差异。即保证了效果，也兼容了低版本浏览器，一举两得。

*简单的动画可以多考虑用兼容性更好的css属性实现，尤其是在PC端。*

width的变化可以用nej的缓动函数封装`util/animation/bezier`轻松实现。为了与regular结合，扩展了regular的[animation指令](http://regularjs.github.io/guide/zh/basic/animation.html)：

``` javascript
 Regular.animation("brand-rotate", function(step){
  var param = step.param, // 传入brand-rotate的参数
      element = step.element, //触发节点
      ...;
  // 它只接受一个done函数作为参数，标志着本步骤的结束
  return function(done){ 
    doAnimation(element,done);
  }
})
```
``` html
<!--品牌节点-->
<div class='brand' r-animation=
       "on:brand-rotate;
        brand-rotate:;
         ">
    ...
</div>
```
在需要翻转的时候$emit("brand-rotate")即可。

#### 时序控制
当数据更新时，所有品牌并不是同时翻转，而是根据品牌所在位置按照一定的延时间隔，从左上角到右下角依次翻转。各个品牌翻转的次序可以根据其位置计算得出，这个不难。接下来的任务就是控制品牌在该翻转的时候翻转。
![翻转次序](http://haitao.nos.netease.com/326ffc51-d7ac-44bb-8bc2-a33c67b2b98f.png)

regular文档的[animation指令](http://regularjs.github.io/guide/zh/basic/animation.html)一章确有delay相关参数，
``` html
<!--7. wait: duration-->
<!--等待数秒再进入下一个步骤-->
<div class='box animated' r-animation=
     "on:click; 
        class: swing ;
        wait: 2000 ;
        class: shake">
  wait: click me
</div>
```
这正是我想要的，我想我应该这么写就能按序翻转了：
``` html
{#list brandList as brand}
    <!--品牌节点-->
    <div class='brand' r-animation=
           "on:brand-rotate;
            wait:{brand_index|getDelayMs};
            brand-rotate:;
             ">
        ...
    </div>
{/list}
```
然鹅，报错了。。。如下图：
![报错](http://haitao.nos.netease.com/fefa5f57-c6bb-48b9-aba6-cebb3d6ca2d7.png)
value中包含表达式，变当做表达式解析成一个对象，然后这里直接在对象上调用trim()，就报错了，看来此处value只支持String类型，也就是说wait值只能写死，不能用表达式。。。因此delay方法只能自己通过setTimeout、requestAnimationFrame等实现了。

看了一下regular在github上的[最新代码](https://github.com/regularjs/regular/blob/master/src/directive/animation.js#L147)，此处已被机智的波神修复了，也早有人提过[issue](https://github.com/regularjs/regular/issues/61)。*看来工程里的regular需要升级了！*

#### 数据更新
这听起来是最容易的，MVVM做的就是从数据到视图自动更新，我们只需关心数据，而无需操心如何更新。然鹅和动画结合后就变得不那么容易了。

数据的更新是在动画进行到一半时触发的，width渐变成0-->更新品牌数据-->width渐变成100%。
当更新数据时，数据对应的节点将会被刷新成新节点并恢复成模板中初始的样子，也就是说节点动画被终止、样式被重置，width直接从0变成100%而没有过渡。

*当我们的用js操作regular模板节点时，会与regular自身的视图更新相冲突，这是问题所在。*

想过把有动画的部分纯用自己的脚本操作节点实现翻转、数据更新，那就是一夜回到解放前（jQuery时代），奈何节点上还有事件绑定、节点数据修改等操作，想想都麻烦。

最终有两个可行方案：
1. 把动画逻辑和更新数据完全分开
- 动画：width 100% -> 0
- width==0时，直接js修改图片url和说明文案
- 动画：width 0 -> 100%（以上全是为了视觉上的翻转效果）
- 等全部动画执行完，整体更新brandList数据
- 自动刷新视图、绑上各种事件（虽说最后dom节点是都刷新了，但是刷新前后显示的图片、文案都是一样的，所以视觉上并看不出）

2. 取巧的方案。
- 设置品牌节点的css样式为width=0
- 模块首次出现时，需要执行动画width 0 -> 100%，使之出现
- 当换一批时，执行动画：width 100% -> 0
- 当有节点width==0时，就刷新这个单个品牌的数据
- 这个品牌数据被更新，样式被重置成初始的样子width=0（正好现在width也是0，所以视觉上并无变化）
- 然后执行动画width 0 -> 100%

*既然运用了regular，如果非必须，还是不要直接操作dom节点的好。* 所以最终选择了第二种。效果如下图：

![最终效果](http://haitao.nos.netease.com/6fd2446a-1feb-437c-b644-1d3b638923ee.gif)

### 痛点

这个需求的实现可以看到regular实现动画的一些局限性：
- r-animation比较适合控制增删class、修改css，从而触发简单的css3动画，如果要实现比较复杂的动画，还是需要直接操作dom节点，并依赖专业的js动画库。
- 动画操作和regular刷新视图会互相影响，应尽量把两块操作分开，一次只做一件事，按序进行。密集的串插进行可能会导致意想不到的问题。
- 波神也提到“动画使用不当是可能与组件本身的数据-UI状态产生冲突，应该尽量使用on/emit的组合，而少使用when/call的组合来实现动画序列”，为的就是避免状态更新而对动画状态产生影响。
