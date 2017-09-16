---
title: KLTable组件开发总结
date: 2017-07-13
---

## 简介

根据新出的交互设计规范，列表（table）除了有固定的显示样式以外，最主要的是需要在指定条件下对表格的头部和底部进行悬浮显示，在列表超出一屏时依然能够将头部和底部操作栏显示给用户，并对与横向宽度过长的 table 对两头的列进行固定，将重要的信息保持显示。

<!-- more -->

因此，对于用户而言，作为基础组件的 [KLTable](http://nek-ui.kaolafed.com/components/layout_KLTable_.html) 应当承担的最重要的功能就是对表格、列、底部的展现方式进行控制，将相关的逻辑集成到组件内部。

KLTable 实际就是一个列表框架，列表中内容的类型、展现方式、交互等在不同的业务情景下都不尽相同，为了满足不同业务需求，则需要尽可能增加组件的的自定义功能。



## 主要功能

### 头部

1. sticky（悬浮在页面）
2. 表头固定在顶部
3. 宽度可拖动
4. 多级表头
5. 表头模版自定义
6. 基础的功能：提示，升降排序

### 内容

1. 左右固定列
2. 内容模版自定义
3. 多选
4. 下钻内容（打开行）

### 其他

1. 翻页器
2. footer 包括翻页器，水平滚动条，需要 sticky
3. footer 自定义模版

## 组件结构

```
├─KLTableCol       列
├─TableHeader      表头
├─TableBody      内容区
├─KLTableTemplate  模版容器
├─TDElements       预设内容样式
├─THElements       预设表头样式
├
├─index.html
├─index.js
├─index.mcss
├─utils.js
└─index.md
```

## 接口

表格组件主要是以列表的形式展现内容，每一列的展现形式可能都不相同，因此可以通过对不同列进行配置来确定每一列的表现形式。

### 使用方式

#### 通过数据配置

从数据结构的角度看，表格列配置可以存放在数组 `columns` 当中，而内容数据则存放在 `source` 数组当中。而排序、分页这两组可选数据，则分别存放到 `sorting`，`paging` 两个对象当中。

```javascript

var table = {
    columns: [
        {
            name: '名称',
            key: 'name'
        },
        {
            name: '价格',
            key: 'price'
        }
    ],
    source: [
        {
            name: '商品1',
            price: 100
        },
        {
            name: '商品2',
            price: 200
        }
    ],
    sorting: {
        isAsc: false,
        key: 'price'
    },
    paging: {
        current: 1,
        totalPage: 10,
        pageSize: 10,
        total: 100
    },
    loading: false
}
```

```html
<kl-table
    columns={table.columns}
    source={table.source}
    paging={table.paging}
    sorting={table.sorting}
    loading={table.loading}
/>
```

#### 通过模版配置

上面是单纯用数据来配置表格，但在某种程度上，纯数据的配置不是那么直观，因为列配置是与展示紧密相关的，如果列配置能够模版当中定义，那么表格组件在使用时，就能更好地将数据与表现进行分离。在此时，在`js`，表现层配置就能够与业务逻辑分离。

```html
<kl-table
    source={table.source}
    paging={table.paging}
    sorting={table.sorting}
    loading={table.loading}
>
    <kl-table-col name="名称" key="name" />
    <kl-table-col name="价格" key="price" />
</kl-table>
```

当然，在某些需求下，列配置可能会有变动的可能，此时如果使用数据而非模版来进行列配置，列配置修改就会变得更为方便。因此保留两种配置形式是有价值的。虽让两种接口在表面上有所区别，但时间上，通过模版配置表格的场景在初始化的时候同样会构建一个 `columns`数组，在实现上是一致的。

#### 通过模版进行配置的实现方式

在`regular`中，需要通过`{`#include this.$body}`才能将嵌套在里面的组件实例化，在实例化的过程中，`config`和`init`的执行顺序是:

    名称 config
    名称 init
    信息 config
    地址 config
    地址 init
    电话 config
    电话 init
    信息 init

在实例化的时候，要读取嵌套组件的数据，则需要在视觉父组件的`init`中进行。

```html
<kl-table>
    <kl-table-col name="名称" key="name" />
    <kl-table-col name="信息">
        <kl-table-col name="地址" key="addr" />
        <kl-table-col name="电话" key="tel" />
    </kl-table-col>
</kl-table>
```

为了方便起见，实现的时候是通过子组件向视觉父组件注入数据的形式来传递子组件包含的数据，利用了`this.$outer`来获取父组件。简化的代码如下。

```javascript
var TableCol = BaseComponent.extend({
    name: 'kl-table-col',
    template: '<div ref="bodyContainer" style="display:none">{`#include this.$body}</div>',
    init: function() {
      this._register();  
    },
    _register: function() {
        this.$outer.data._innerColumns.push({this.data});
    }

});

```

利用`this.$outer`属性，能够在子组件实例化的过程中逐级向上传递数据，从而构建树结构`columns`。

```javascript
// columns
[
    {
        name: '名称',
        key: 'name'
    },
    {
        name: '信息',
        children: [
            {
                name: '地址',
                key: 'addr'
            },
            {
                name: '电话',
                key: 'tel'
            }
        ]
    }
] 
```

但于内容列而言，只是一个一维数组，因此需要将`columns`中的叶都提取出来构造出一个一维数组`_dataColumns`。两个数组分别用于构建表格的表头和内容。提取时并不会进行深拷贝，所以对`columns`上各项的改动会直接映射到`_dataColumns`上。

```javascript
// _dataColumns
[
    {
        name: '名称',
        key: 'name'
    },
    {
        name: '地址',
        key: 'addr'
    },
    {
        name: '电话',
        key: 'tel'
    }
] 
```

`TableBody` 中的模版片段
```html
{`#list source as item by item_index}
  <tr>
  {`#list _dataColumns as column by column_index}
    <td>{item[column.key]}</td>
  {/list}
  </tr>
{/list}
```

### 表头和内容区的自定义模版

#### 通过模版`kl-table-template`定义

其实现方式与`kl-table-col`类似，也是向父组件注入模版字符串，这里的区别在于，在定义模版时，对于插值需要特殊处理。如果在两端不加上`{'`、`'}`，那么进行父组件渲染的时候，`{item.name}`会被当作是父组件的插值进行解析和替换，因此可以通过插值字符串的形式绕开这一步，保留自定义模版中的插值语法。这样写虽然比较丑陋，但是有效。

考虑的另一种方式，是使用其他符号替换regular默认的插值符合，例如`$:item.name:$`，在`kl-table-template`实例化的时候将两端的`$:`、`:$`再替换成`{`、`}`，但在使用`r-class`、`r-style`时的`{`、`}`也会被解析，所以还是不可行。

定义时声明 `type` 为 `header` 则为表头模版，默认值是 `content`，即内容模版。

```html
<kl-table>
    <kl-table-col key="name">
        <kl-table-template type="header">
            <span>名称</span>
        </kl-table-template>
        <kl-table-template>
            {'<span>{item.name}</span>'}
        </kl-table-template>
    </kl-table-col>
</kl-table>
```

实际上，在完成实例化后，该模版字符串会被注入到列的`template`属性当中。

#### 通过数据进行配置

利用数组配置，可以直接将模版字符串赋予`template`，此时就不需要考虑插值的解析问题了。

由于可以在js定义模版，那么其形式不应该仅限于字符串，对于不同的情景可以采用不同的形式。

1. template，模版字符串；
2. format，纯粹的字符串格式化，不对html进行渲染，保留插值语法；
3. formatter，通过函数返回模版字符串，适用于当模版需要动态运算生成的情景。

### 多级表头

`table`元素是通过`colSpan`和`rolSpan`控制每个单元格左占列和行的多少。

```html
<table>
  <thead>
    <tr>
      <th colspan="1" rowspan="2"></th>
      <th colspan="2" rowspan="1"></th>
    </tr>
    <tr>
      <th colspan="1" rowspan="1"></th>
      <th colspan="1" rowspan="1"></th>
    </tr>
  </thead>
</table>
```

![](http://og40ypzfa.bkt.clouddn.com/table.png)

要实现多级表头，则同样需要利用这种特性，它将依赖一个二维数组。因此需要将`columns`这个树结构根据树的层级建立一个二维数组。

![](http://og40ypzfa.bkt.clouddn.com/table.2.png)

模版通过嵌套`kl-table-col`进行配置

```html
<!-- 一个多级表头的例子-->
<kl-table source={table.source} count={count}>
  <kl-table-col name="name" />
  <kl-table-col name="info">
      <kl-table-col name="addr" />
      <kl-table-col name="phone" />
  </kl-table-col>
</kl-table>
```

生成的`columns`如下

```javascript
// column的结构
columns = [
  {
    name: 'name'
  },
  {
    name: 'info'
    children: [
      { name: 'addr' },
      { name: 'phone' }
    ]
  }
]
```

要将这个树的每一个节点的宽度和深度都计算出来，并映射出一个二维数组。

```javascript
const updateHeaderSpan = function (headers) {
  const len = headers.length;
  headers.forEach((row, rowIndex) => {
    row.forEach((header) => {
      header._headerColSpan = header._nodeWidth;
      header._headerRowSpan = len - rowIndex - (header._nodeDepth - 1);
    });
  });
};
// 转化headers的方法
_.getHeaders = function (_columns) {
  const headers = [];
  const extractHeaders = function (columns, depth) {
    columns.forEach((column) => {
      if (hasChildren(column)) {
        extractHeaders(column.children, depth + 1);
      }
      if (!headers[depth]) {
        headers[depth] = [];
      }
      // 计算深度和宽度
      if (hasChildren(column)) {
        column._nodeDepth =
          1 +
          column.children.reduce(
            (previous, current) => (
              current._nodeDepth > previous
                ? current._nodeDepth
                : previous
            ),
            0,
          );
        column._nodeWidth = column.children.reduce(
          (previous, current) => previous + (current._nodeWidth || 0),
          0,
        );
      } else {
        column._nodeDepth = 1;
        column._nodeWidth = 1;
      }
      headers[depth].push(column);
    });
  };
  extractHeaders(_columns, 0);
  return updateHeaderSpan(headers);
};
```

```javascript
// 生成的二维数组
headers = [
  [
    {
      name: 'name',
      _nodeDepth: 1,
      _nodeWidth: 1,
      _headerColSpan: 1,
      _headerRowSpan: 2
    },
    {
      name: 'info',
      _nodeDepth: 2,
      _nodeWidth: 2,
      _headerColSpan: 2,
      _headerRowSpan: 1
    }
  ],ee
  [
    {
      name: 'addr',
      _nodeDepth: 1,
      _nodeWidth: 1,
      _headerColSpan: 1,
      _headerRowSpan: 1
    },
    {
      name: 'phone',
      _nodeDepth: 1,
      _nodeWidth: 1,
      _headerColSpan: 1,
      _headerRowSpan: 1
    }
  ]
]
```


```html
<!-- 简化的模版片段-->
{`#list headers as headerRow by headerRow_index}
  <tr class="tb_hd_tr">
    {`#list headerRow as header by header_index}
      <th
        colspan={header._headerColSpan}
        rowspan={header._headerRowSpan}
        >
          {header.name}
      </th>
    {/list}
  </tr>
{/list}

```


### 表头 fixed 与 sticky

在页面可视范围有限的情况下，表头、列的固定能够提升用户的使用体验。对于表头的固定，通常有两种实现方式，固定在表格中，或者悬浮在页面顶部。

参考饿了么的组件库`element`，它是采用了前一种方式，主要原理是将表格拆分成两部分，分别是表头和内容，当需要固定表头时，实际上就是设置内容的高度，并使其可以滚动。

但在我们的交互规范中，是使用 sticky 方式，将表头固定在页面顶部。实现方式是，比较滚动的高度与表格的位置，满足条件时通过`position:fixed`定位使能悬浮。在表头脱离表格时，设置一个与表头同高的占位 `div`占位，防止内容上移。

尽管两种固定模式不太一样，但都可以基于同一种结构实现，因此两种模式可以共存，可以选择保留两种固定模式。

而对于水平滚动条和表格底部的分页器，我们可以将二者放到`table_footer`中，同样利用`sticky`的方式来进行固定，使得表格头尾固定，方便操作。

#### fixed

表头固定在表格上，通过样式进行控制。

```html
<div class="kl_table_header">
  <table-header />
</div>
<div class="kl_table_body" style="overflow: auto">
  <table-body />
</div>
```

#### sticky

1. 滚动监听

由于滚动的容器不一定是页面，因此需要指定滚动监听的对象，默认监听`window`的滚动，否则通过`document.querySelector(scrollParent)`获取监听对象。

```javascript
_getScrollParentNode() {
  const data = this.data;
  if (data.scrollParentNode) {
    return data.scrollParentNode;
  }
  if (data.scrollParent) {
    return (data.scrollParentNode =
      document.querySelector(data.scrollParent) || window);
  }
  return (data.scrollParentNode = window);
}
```

2. 位置计算

页面滚动高度

`scrollY = window.pageYOffset || document.documentElement.scrollTop`

非页面容器滚动高度

`scrollY = scrollParentNode.scrollTop`

表格位于页面中的高度，需要利用`getBoundingClientRect`方法计算

`tableTop = tableRect.top - parentRect.top + scrollParentNode.scrollTop`

`tableBottom = tableRect.bottom - parentRect.top + scrollParentNode.scrollTop`


通过比较表格位置与容器滚动高度的大小，判断是否`sticky`悬浮显示的状态。

```javascript
if(scrollY + headerHeight > tableBottom
  || scrollY < tableTop) {
  this.data.stickyActive = false;
} else {
  this.data.stickyActive = true;
}
```

3. 表头占位

当表头利用`position:fixed`属性固定到页面中后，表头原来的位置就会有空缺，需要设置一个占位用的block防止表格下部往上跳。

```html
<div class="header_placeholder"
  r-style={`{
    height: stickyHeader && stickyHeaderActive ? headerHeight + 'px' : 0
}`}/>
```

### 列固定

列固定的实现用到了则是对表格本地进行拷贝，利用`position:absolute`固定到左边和右边，固定表格的可视宽度由固定列的总宽度决定，从而实现显示所需固定列的效果。

```html
<!-- 列固定的基本原理 -->
<style>
   .table-fixed {
      position: absolute;
      overflow: hidden;
   }
</style>
<div class="table-fixed" style="width:{fixedTableWidth};left:0;">
  <table-header/>
  <table-body/>
<div>
<div class="table-fixed table-fixed-right" style="width:{fixedTableRightWidth};right:0;">
  <table-header/>
  <table-body/>
<div>
```

### 可变列宽

参考[element-table](http://element.eleme.io/#/zh-CN/component/table)的实现方式，检查鼠标在表头的位置，进入可拖动范围后更改鼠标图案，在第一次点击时记录鼠标与表格左边的距离，鼠标释放后再次记录鼠标与左边的距离，然后计算当前拖动列表的新宽度，并通过数据绑定更新到表格的各个部件。

#### 实现

1. 宽度计算

```javascript
_startResizing(e, header, headerIndex, headerTrIndex) {
  const self = this;
  // 表格左边的位置
  const tableLeft = self.$parent.$refs.table.getBoundingClientRect().left;
  // 根据id获取当前要拖动宽度的表头
  const headerEle = self.$refs[`table_th_${headerTrIndex}_${headerIndex}   `];
  // 获取拖动表头的左边位置
  const headerLeft = headerEle.getBoundingClientRect().left;

  header._resizeParam = {
    tableLeft,
    headerLeft,
  };

  // 获取标尺
  const resizeProxy = self.$parent.$refs.resizeProxy;
  resizeProxy.style.visibility = 'visible';

  // 鼠标移动时更改标尺的位置
  const onMouseMove = function (_e) {
    _e.preventDefault();

    const proxyLeft = _e.pageX - tableLeft;
    const headerWidth = _e.pageX - headerLeft;

    if (headerWidth > HEADER_MIN_WIDTH) {
      resizeProxy.style.left = `${proxyLeft}px`;
    }
  };

  // 鼠标放开时开始设置新的宽度
  // 当更改一个多级表头的宽度时，所拖动的是该节点下所有最又边节点的宽度
  const onMouseUp = function (_e) {
    _e.preventDefault();
    resizeProxy.style.visibility = 'hidden';
  
    // 当前鼠标到表格最左边的距离
    const headerWidth = _e.pageX - headerLeft;
    // 获取当前header节点的宽度信息
    // 当前节点总的宽度
    // 当前节点的最右边一个叶节点的宽度
    const leftLeavesWidth = getLeftLeavesWidth(header);
    setColumnWidth(
      header,
    
      // headerWidth - leftLeavesWidth 则是新的列宽度
      headerWidth - leftLeavesWidth,
    );

    self.$emit('columnresize', {
      sender: self,
    });

    document.removeEventListener('mousemove', onMouseMove);
    document.removeEventListener('mouseup', onMouseUp);

    header._isDragging = false;
    self._disableResize();
  };

  document.addEventListener('mousemove', onMouseMove);
  document.addEventListener('mouseup', onMouseUp);
}

// 计算当前表头节点下非最右边叶节点列的总宽度
const getLeftLeavesWidth = function (column) {
  const info = getColumnWidth(column);
  // leftLeavesWidth = widthInfo.width - widthInfo.lastLeafWidth
  return info.width - info.lastLeafWidth;
};

// 计算出当前表头节点的总宽度和最后一个叶节点的宽度。
const getColumnWidth = function (column) {
  const ret = {
    width: 0,
    lastLeafWidth: 0,
  };
  if (hasChildren(column)) {
    column.children.forEach((item, index) => {
      const tmp = getColumnWidth(item);
      if (index === column.children.length - 1) {
        ret.lastLeafWidth = tmp.width;
      }
      ret.width += tmp.width;
    });
  } else {
    return {
      width: column._width,
      lastLeafWidth: column._width,
    };
  }
  return ret;
};

// 找到该节点下最右边一个节点并设置列宽度
const setColumnWidth = function (column, width) {
  const children = column.children;
  if (hasChildren(column)) {
    setColumnWidth(children[children.length - 1], width);
    return;
  }
  column._width = Math.max(width, HEADER_MIN_WIDTH);
};

```

### 下钻

目前功能还没有完善，初步的设计方案与其他自定义模版类似，`table.template`的`type`设置为`expand` 。也可以直接赋值`expandTemplate`字段定义下钻内容的模版。

通过`column.expandable && !item.disexpandable`控制是否可以下钻，支持多个下钻的列对应不同的模版

```javascript
_onExpand(item, itemIndex, column) {
  if (!this.data.fixedCol) {
    this._expandTr(item, itemIndex, column);
  }

  this.$emit('expand', {
    sender: this,
    expand: item.expand,
    column,
    item,
    index: itemIndex,
  });
},
_expandTr(item, itemIndex, column) {
  item._expanddingColumn = column;
  item.expand = !item.expand;
  if (column.expandable) {
    this._updateSubTrHeight(item, itemIndex);
  }
}
```

```html
{`#if item.expand}
<tr class="tb_bd_tr td_bd_tr_nohover">
  <td ref="td{item_index}"
    r-style={`{
        height: item._expandHeight && fixedCol ? item._expandHeight + 'px' : 'auto'
    }`}
    class="m-sub-protable-td {column.tdClass}"
    colspan={_dataColumns.length}>
      {`#include item._expanddingColumn.expandTemplate}
  </td>
</tr>
{/if}
```

使用例子：

```html
<kl-table>
  <kl-table-col name="名称" key="name">
    <kl-table-template type="expand">
      {'<span>{item.name}</span>'}
    </kl-table-template>
  </kl-table-col>
</kl-table>
```

### 自定义事件传递

内容和表头都可以自定义模版，那么相关的时间需要传递出来可以利用 `emit('eventtype', event)` 将事件抛出，并可在表格中可以绑定该事件。

由于是表头和内容都是独立的组件，因此不能直接调用`this.$emit`触发事件，可以调用封装过的`this.emit`来进行触发。

当然也可以直接通过`this.$table.$parent`调用外部方法。

```javasript
  emit(...args) {
    this.$parent.$emit.call(this.$parent, ...args);
  }
```

```html
<kl-table on-linkclick={this.onLinkClick($event)}>
  <kl-table-col name="title" key="title">
    <kl-table-template>
      {'<a on-click={this.emit('linkclick', item)}>{item.title}</a>'}
    </kl-table-template>
  </kl-table-col>
</kl-table>
```

### filter 注册实现

```javascript
const oldFilterFunc = KLTable.filter;

KLTable.filter = function (...args) {
  TableHeader.filter(...args);
  TableBody.filter(...args);
  oldFilterFunc.apply(KLTable, args);
};
```

### 未实现的功能

1. 多级内容的展示

正如多级表头的实现，需要一个二维数组才能利用`rowspan`、`colspan`实现多级展示，但在实际的业务当中，所取得的没一行的`item`的数据格式、字段都难以确定，如何转换为一个二维数组是个令人头疼的问题。

2. 下钻

功能尚未完善，因为还没有具体的交互规范，但基本的功能设计已经完成，如上文所述。

## 问题

1. 对不同业务场景的适应性。
 
由于不同的项目中有不同的应用场景和需求，在开发初期，考虑得不够周全，随着组件在业务中投入使用，收集到了许多问题和功能需求，因此在不断地往组件上添加属性、接口。但还是要尽可能得保持组件的简洁，因为简洁才容易使用，各项功能之间的影响和干扰会容易得到控制。

2. 性能优化。

在组件有部分功能需要使用到定时器，用于更新某些属性。因为是在非Regular组件范围内，因此在更新时一般会用到`this.$update`方法。`this.$update`执行后，会向上"冒泡"到它的父组件上，直到根组件或设置了`isolate`属性的组件为止，因此频繁的在定时器中调用`this.$update`是可能会导致大批的Regular组件强制进入到`$digest`过程中，降低页面性能。

```javascript
// 容易导致性能低的写法
setInterval(() => {
  this.data.prop = this.getProp();
  this.$update();
  // this.$update('prop', this.getProp());
}, 200);
```

因此需要尽可能地减少`this.$udpate`的调用。如果确实有要调用`this.$update`的地方，则可以先行对该值进行判断，然后再调用。

```javascript
updateData(key, val) {
    if(this.data[key] !== val) {
        this.$update(key, val);
    }
},
```

## links

[code](https://github.com/kaola-fed/nek-ui/tree/master/src/js/components/layout/KLTable)

[demo](http://nek-ui.kaolafed.com/components/layout_KLTable_.html)

## 参考

[element-table](http://element.eleme.io/#/zh-CN/component/table)

by [Elcarim](https://github.com/elcarim5efil)