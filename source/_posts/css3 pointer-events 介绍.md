---
title: css3 pointer-events
date: 2017-05-31
---

## css3 pointer-events
### pointer-events 是什么？
顾名思义，pointer-events 是一个用于 HTML 指针事件的属性。

pointer-events 可以禁用 HTML 元素的 hover/focus/active 等动态效果。

<!-- more -->

默认值为 auto，语法：

```
pointer-events: auto | none | visiblepainted | visiblefill | visiblestroke | visible | painted | fill | stroke | all;
```

我们常用的 auto | none 属性，需要注意的是，其他的属性只有 SVG 元素适用。

auto：可以使用指针事件。
none：禁用指针事件，需要注意的是，**当禁用指针的的元素有子/父元素时，在时间冒泡/捕获阶段，事件将在其子/父元素触发。**

### 常用场景
1. 禁用 a 标签事件效果

在做 tab 切换的时候，当选中当前项，禁用当前标签的事件，只有切换其他 tab 的时候，才重新请求新的数据。

```
<!--CSS-->
<style>
    .active{
        pointer-events: none;
    }
</style>
<!--HTML-->
<ul>
    <li><a class="tab"></a></li>
    <li><a class="tab active"></a></li>
    <li><a class="tab"></a></li>
</ul>
```

2. 切换开/关按钮状态

点击提交按钮的时候，为了防止用户一直点击按钮，发送请求，当请求未返回结果之前，给按钮增加 pointer-events: none，可以防止这种情况，这种情况在业务中也十分常见。

```
<!--CSS-->
.j-pro{
    pointer-events: none;
}
<!--HTML-->
<button r-model={this.submit()} r-class={{"j-pro": flag}}>提交</button>
<!--JS-->
submit: function(){
    this.data.flag = true;
    this.$request(url, {
        // ...
        onload: function(json){
            if(json.retCode == 200){
                this.data.flag = false;
            }
        }.bind(this)
        // ...
    });
}
```

3. 防止透明元素和可点击元素重叠不能点击

一些内容的展示区域，为了实现一些好看的 css 效果，当元素上方有其他元素遮盖，为了不影响下方元素的事件，给被遮盖的元素增加 pointer-events: none; 可以解决。

```
<!--CSS-->
.layer{
    backround: linear-gradient(180deg, #fff, transparent);

}
.j-pro{
    poninter-events: none;
}
<!--HTML-->
<ul>
    <li class="layer j-pro"></li>
    <li class="item"></li>
    <li class="item"></li>
    <li class="item"></li>
</ul>
```

### poninter-events 兼容性
![image](https://haitao.nos.netease.com/40dd599a-82d0-4662-bfcd-060d34c212cd.png)
