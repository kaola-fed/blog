---
title: 关于transform引起的app crash问题
date: 2017-09-26
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
## 一.Chrome 如何将DOM绘制到屏幕
从概念上讲，会经历以下几个步骤

* 获取DOM 并将其分割为多个层（层的单位是瓦片tile:蓝色的瓦格）【图1】 
* 将每个层(GraphicsLayer)独立的绘制进位图中
* 将层作为纹理（texture）上传至 GPU
* 复合(Composite)多个层来生成最终的屏幕图像


> &emsp;&emsp;当 Chrome 首次为一个 web 页面创建一个帧(frame)时，以上步骤都需要执行。但对于以后出现的帧可以走些捷径：
    &emsp;&emsp;1. 如果某些特定 CSS 属性变化，并不需要发生重绘。Chrome 可以使用早已作为纹理而存在于 GPU 中的层来重新复合，但会使用不同的复合属性(例如，出现在不同的位置，拥有不同的透明度等等)。
    &emsp;&emsp;2. 如果层的部分失效，它会被重绘并且重新上传。如果它的内容保持不变但是复合属性发生变化(例如，层被转化或透明度发生变化)，Chrome 可以让层保留在 GPU 中，并通过重新复合来生成一个新的帧。
  
![](http://divio.qiniudn.com/Fq9iXaEPLykGOdOhTfZV0eP406uE)
  
  **【结论】以层为基础的复合模型对渲染性能有着深远的影响。当不需要绘制时，复合操作的开销可以忽略不计。当需要进行动画的元素包含在这个合成层之下时，动画的每一帧只需要去重新绘制这个 Graphics Layer 即可，从而达到提升动画性能的目的。**  
   
   
   
## 二.层的隐藏坑

   &emsp;&emsp;层会占用系统 RAM 与 GPU(在移动设备上尤其有限)的内存，并且拥有大量的层会因为记录哪些是可见的而引入额外的开销。许多层还会因为过大与许多内容重叠而导致“过度绘制(overdraw)”的情况发生，也就可能引发移动设备的崩溃.因此我们要注意管理好我们页面上的Graphics Layer
   
## 三.层的创建标准
   > &emsp;&emsp;什么情况下能使元素获得自己的层？虽然 Chrome 的启发式方法(heuristic)随着时间在不断发展进步，但是从目前来说，满足以下任意情况便会创建层：
  ```text 
  1.3D or perspective transform CSS properties
  2.<video> elements using accelerated video decoding
  3.<canvas> elements with a 3D (WebGL) context or accelerated 2D context
  4.Composited plugins (i.e. Flash)
  5.Elements with CSS animation for their opacity or using an animated transform
  6.Elements with accelerated CSS filters
  7.Element has a descendant that has a compositing layer (in other words if the element has a child element that’s in its own layer)
  8.Element has a sibling with a lower z-index which has a compositing layer (in other words the it’s rendered on top of a composited layer)  
  元素有一个 z-index 较低且包含一个复合层的兄弟元素(换句话说就是该元素在复合层上面渲染)
  
  ```
## 四.层触发了GPU硬件加速，硬件加速的注意事项

* 内存。如果GPU加载了大量的纹理，那么很容易就会发生内容问题，这一点在移动端浏览器上尤为明显，所以，一定要牢记不要让页面的每个元素都使用硬件加速

* 使用GPU渲染会影响字体的抗锯齿效果。这是因为GPU和CPU具有不同的渲染机制。即使最终硬件加速停止了，文本还是会在动画期间显示得很模糊。著作权归作者所有。

  
  
## 五.由上面第8条引出的问题
 &emsp;&emsp;搜索文章发现了[CSS3硬件加速也有坑！！!](https://div.io/topic/1348)，然后发现了一个很坑的场景。大家可以先看下这篇文章。
 &emsp;&emsp;头部有个轮播，用了translateX实现滚动，下面有一个列表。按照原本预期，应该只有轮播形成了一个层，但实际上下面的li也创建了自己的层。就这个场景做了以下几个实验。
 &emsp;&emsp;先说下结论：
 
 **使用3D硬件加速提升动画性能时，最好给元素增加一个z-index属性，人为干扰复合层的排序，可以有效减少chrome创建不必要的复合层，提升渲染性能，移动端优化效果尤为明显。**
 
 
 首先在控制台打开layers面板,我们通过看layers面板来看我们的层
 ![chrome.jpg](quiver-image-url/F4D141B7EDFA1632C0CF1CC77C0151A3.jpg =806x800)
 
 
 
 
 ```html
 <head>
    <style>
        @keyframes move {
              0% { transform:translateX(0px) }
             50% { transform:translateX(10px) }
            100% { transform:translateX(0px) }
        }
        #title_animation {
            animation: move 1s linear infinite;
        }

        #title_rotate {
            transform: rotateX(-30deg) rotateY(30deg);
        }

        #title_2d {
            transform: translate(100px, 100px);
        }

        #title_3d {
            transform: translate3d(100px, 100px, 0); 
        }      
    </style>
 </head>
 <body>
    <div id="parent">
  		<h1 id="title_rotate">
        Ha Ha Ha~  
      </h1>
      <ul>
        <li>1</li>
        <li>11</li>
        <li>111</li>
        <li>1111</li>
        <li>11111</li>
        <li>111111</li>
      </ul>
    </div>      
</body>
 ```
 
 然后我们给我们的h1 变化不同的样式，打开layers控制台可以发现下面几种不同的情况（形成层会出现黄色的框）：
 
 ### 1.添加transform: translate(100px, 100px);
 ![title_2d.jpg](https://haitao.nos.netease.com/ad20e64a-480c-438c-92aa-c64165da46bf.jpg) 没有形成层
 
 ### 2.添加transform: translate3d(100px, 100px, 0) 下面的ul区域也形成了层;
 ![title_3d.jpg](https://haitao.nos.netease.com/09d3a1a6-d5a5-4d23-bece-d002bf738e14.jpg)
 
 ### 3.添加transform: rotateX(-30deg) rotateY(30deg) 下面的ul区域也形成了层;
 ![title_rotate_1.jpg](https://haitao.nos.netease.com/3cd8b70e-8ca2-46ae-86c3-15ab44913b29.jpg)
 
 ### 4.添加animation, ul形成了层， li也形成了层
 ```javascript
  @keyframes move {
      0% { transform:translate(100px, 100px) }
     50% { transform:translateX(100px, 200px) }
    100% { transform:translateX(100px) }
  }
  #title_animation {
    animation: move 1s linear infinite;
  }
 ```
 ![title_animation.jpg](https://haitao.nos.netease.com/e80350b2-daf8-4484-8d48-f620ce253f19.jpg)
 
 ### 5.添加transform: translate3d(100px, 100px, 0);position:relative;z-index:2; 变为只有一个层
 ![title_3d_z.jpg](https://haitao.nos.netease.com/cc4aae20-d875-4645-afbd-3d5bbdf3affe.jpg)
 
 ### 6.添加transform: rotateX(-30deg) rotateY(30deg);position:relative;z-index:2; 变为只有一个层
![title_rotate_z.jpg](https://haitao.nos.netease.com/5268b6d6-757b-4507-ba96-083662c7f529.jpg)

### 7.添加animation和z-index, 变为只有一个层
![title_animation_z.jpg](https://haitao.nos.netease.com/44f77a99-f732-48ad-a759-f0a118f82e31.jpg)
 
 
 参考文章：
* [CSS3硬件加速也有坑！！！](https://div.io/topic/1348)
* [Accelerated Rendering in Chrome](https://www.html5rocks.com/zh/tutorials/speed/layers/)
* [CSS动画之硬件加速](https://www.w3cplus.com/css3/introduction-to-hardware-acceleration-css-animations.html)
* [你所不知道的 CSS 动画技巧与细节](https://github.com/chokcoco/iCSS/issues/27.html)
* [运用webkit绘制渲染页面原理解决iscroll4闪动的问题](http://www.tuicool.com/articles/rYby6v)
* [iOS Safari使用"-webkit-transform"内存不足](http://www.developerq.com/article/1505089976)