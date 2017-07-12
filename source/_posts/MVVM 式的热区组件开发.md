---
title: MVVM 式的热区组件开发
date: 2017-07-04
---

## 目录

1. 什么是热区
2. 热区功能介绍
3. 实现手段与结构划分
4. 相关指令介绍
5. 体验增强
6. 其它总结

<!-- more -->

## 1. 什么是热区

热区，是指在一张图片上选取一些区域，每个区域链接到指定的地址。

因此热区组件的功能，就是在图片上设置多个热区区域并配置相应的数据。

## 2. 热区功能介绍

热区需要实现的功能为：

1. 创建新热区区域；
2. 热区大小可调整；
3. 热区位置可更改；
4. 热区数据可设置；
5. 设置数据可显示。

 - 试试实际效果： [**戳我**](http://aeodu.com/regular-hotzone)
 - 具体项目代码： [**戳我**](https://github.com/Deol/regular-hotzone)

## 3. 实现手段与结构划分

> 如果你在公司里听到有人在讨论 MVVM，那么话题一定离不开 RegularJS。

所以这里以 RegularJS 为例对主题进行介绍（但若是用 Vue 在实现上区别其实也不大）：

MVVM 讲究以数据驱动视图，然而热区这类场景需要需要大量的 DOM 操作。

还好，AngularJS、VueJS 和 RegularJS 等框架都提供了**自定义指令**（directive，写法及用法上大同小异），方便开发者对纯 DOM 元素进行底层操作。

也避免了在节点上大量的挂载 `on-` 开头的属性：

```html
<!-- not so good -->
<div on-keydown={this.handleKeyDown($event)}></div>

<!-- better -->
<div r-keydown></div>
```

由上可知，热区组件重度依赖指令，则代码结构最终设定为：

```
src
 ├── assets
 │      ├── directive
 │      │     ├── addItem.js        添加热区指令
 │      │     ├── changeSize.js     改变尺寸指令
 │      │     ├── dragItem.js       移动位置指令
 │      │     ├── resizeImg.js      图片resize处理指令
 │      │     └── ...
 │      ├── constant.js             通用常量
 │      ├── filter.js               过滤器
 │      ├── operations.js           处理逻辑封装
 │      └── util.js                 工具函数
 ├── components
 │      ├── modal                   数据设置模态窗组件
 │      └── zone                    热区区域组件
 ├── mcss
 │      ├── _reset.mcss             限定作用范围的样式 reset
 │      └── index.mcss              基本样式
 ├── index.js                       组件入口
 └── view.html                      组件模板
```

## 4. 相关指令介绍

> 指令一般需要返回一个函数用于指令销毁工作。
> 
> Question: 为什么要返回销毁函数，而不是通过监听 $destroy 事件来完成？
> 
> Answer: 因为指令的销毁并不一定伴随着组件销毁，指令的生命周期更短，一些语法元素（`if/list/include`）会导致它在组件销毁之前被重复创建和销毁。

由于热区组件使用了较多的事件监听，基于上述考虑，热区操作指令中都返回了用于 **事件解绑** 的函数。

---

新增热区区域、拖拽热区位置和调整热区大小指令，都存在三个操作阶段：

 1. mousedown: 鼠标选中；
 2. mousemove: 移动鼠标，此时通过 js 动态改变元素的样式，但并不对真实数据做修改；
 3. mouseup: 释放鼠标，将改动保存到真实数据中。

```js
import { dom } from 'regularjs';

export default function (content) {

    dom.on(content, 'mousedown', handleMouseDown);

    function handleMouseDown(e) {
        // ...

        dom.on(window, 'mousemove', handleChange);
        dom.on(window, 'mouseup', handleMouseUp);

        function handleChange(e) {
            // ...
        };

        function handleMouseUp() {
            // ...
            
            dom.off(window, 'mousemove', handleChange);
            dom.off(window, 'mouseup', handleMouseUp);
        };
    }

    return () => {
        // 解绑 mousedown 事件
        dom.off(content, 'mousedown', handleMouseDown);
    };
}
```

这里可以做些优化：

 1. 在 changeSize 时对可拖拽点的事件监听上，利用「事件委托」+「自定义属性」减少事件绑定，并统一处理不同方位的拖拽点：

![dragpoint](https://user-images.githubusercontent.com/4961878/27834666-699cea4c-610a-11e7-95a8-77488634d3a2.png)如图所示，热区区域周围的八个小方块就是「拖拽点」。
 
```html
<ul r-changeSize>
    <li class="hz-u-square hz-u-square-tl" data-pointer="dealTL"></li>
    <li class="hz-u-square hz-u-square-tc" data-pointer="dealTC"></li>
    <li class="hz-u-square hz-u-square-tr" data-pointer="dealTR"></li>
    <li class="hz-u-square hz-u-square-cl" data-pointer="dealCL"></li>
    <li class="hz-u-square hz-u-square-cr" data-pointer="dealCR"></li>
    <li class="hz-u-square hz-u-square-bl" data-pointer="dealBL"></li>
    <li class="hz-u-square hz-u-square-bc" data-pointer="dealBC"></li>
    <li class="hz-u-square hz-u-square-br" data-pointer="dealBR"></li>
</ul>
```

```js
function handleMouseDown(e) {
    // 获取选中节点的自定义属性值
    let pointer = e.target.dataset.pointer;
    if(!pointer) {
        return;
    }
    e && e.stopPropagation();

    dom.on(window, 'mousemove', handleChange);
    dom.on(window, 'mouseup', handleMouseUp);

    function handleChange(e) {
        e && e.preventDefault();

        // 处理选中不同拖拽点时的情况
        let styleInfo = operations[pointer](itemInfo, moveX, moveY);

        // 边界值处理
        itemInfo = operations.dealEdgeValue(itemInfo, styleInfo, container);
        
        // ...
    }
    function handleMouseUp() {
        // ...

        dom.off(window, 'mousemove', handleChange);
        dom.off(window, 'mouseup', handleMouseUp);
    }
};
```

统一封装 operations 处理逻辑：
```js
export default {
    /**
     * 改变热区大小时的边界情况处理
     * @param {Object} itemInfo   实际使用的热区模块数据 
     * @param {Object} styleInfo  操作中的热区模块数据
     * @param {Object} container  图片区域的宽高数据
     */
    dealEdgeValue(itemInfo, styleInfo, container) {},
    /**
     * 处理不同的拖拽点，大写字母表示含义：T-top，L-left，C-center，R-right，B-bottom
     * @param  {Object} itemInfo 
     * @param  {Number} moveX 
     * @param  {Number} moveY
     * @return {Object} 对过程数据进行处理
     */
    dealTL(itemInfo, moveX, moveY, minLimit = MIN_LIMIT) {},
    dealTC(itemInfo, moveX, moveY, minLimit = MIN_LIMIT) {},
    dealTR(itemInfo, moveX, moveY, minLimit = MIN_LIMIT) {},
    dealCL(itemInfo, moveX, moveY, minLimit = MIN_LIMIT) {},
    dealCR(itemInfo, moveX, moveY, minLimit = MIN_LIMIT) {},
    dealBL(itemInfo, moveX, moveY, minLimit = MIN_LIMIT) {},
    dealBC(itemInfo, moveX, moveY, minLimit = MIN_LIMIT) {},
    dealBR(itemInfo, moveX, moveY, minLimit = MIN_LIMIT) {}
};
```

 2. 在拖拽移动热区位置时，使用 translate 替代直接改变 top && left 值避免重绘，实现更流畅的拖拽效果。

```js
// bad
dom.css(elem, {
    top: `${moveY}px`,
    left: `${moveX}px` 
});

// better
dom.css(elem, {
    transform: `translate(${moveX}px, ${moveY}px)` 
});
```

 3. 热区尺寸单位用 % 取代 px，不同屏幕尺寸的用户都获得更好的热区操作体验。
 
    但热区区域存在最小尺寸限制，需要利用 [element-resize-detector](https://www.npmjs.com/package/element-resize-detector) 对图片进行监听，在图片尺寸变化时对边界区域做兼容。

```js
import elementResizeDetectorMaker from 'element-resize-detector';
const erd = elementResizeDetectorMaker();

export default function resizeImg(elem) {
    // ...

    const resize = _.debounce(() => {
        // ...
    }, 500);

    erd.listenTo(elem, resize);

    return () => {
        erd.removeListener(elem, resize);
    };
};
```

 4. 通过计算属性动态修改设置数据的 hover 位置，确保不超过图片范围，以确保信息的正确显示。

```html
<ul r-style={{top: infoTop, bottom: infoBottom, left: infoLeft, right: infoRight, transform: infoTransform}}></ul>
```

## 5. 体验增强

1. 使用透明小方块增强鼠标从热区区域移动到设置信息区域的体验：

![hideblock](https://user-images.githubusercontent.com/4961878/27834703-9181ed0a-610a-11e7-8315-a2c2f4ce9cec.png)（这里用灰色标识一下小方块）

2. 双击热区区域弹出信息设置模态框时，利用 r-autofocus 指令自动聚焦：

![modal](https://user-images.githubusercontent.com/4961878/27834712-98e566ee-610a-11e7-91a0-ef7bba3a0e85.png)

同时绑定键盘监听事件，监听 Enter 键和 Esc 键，增强「确认」和「取消」体验。

## 6. 其他总结

1. 无论 **reset**（容易遗漏）或基础样式，都需要限制 scope，避免命名空间污染；
2. 统一控制 Constant 全局常量，如最小热区尺寸限制。