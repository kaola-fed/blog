---
title: 实现 Excel 纯前端复制粘贴
date: 2017-10-27
---

### 背景
后台系统中经常存在通过上传 Excel 导入数据的功能需求。

但运营方在使用该功能时经常会遇到这些问题：
1. Excel 文件需要修改至符合系统要求；
2. 上传时可能需要进入较深的文件路径定位文件；
3. 数据过多时需长时间等待系统响应，甚至很可能上传失败；
4. 发现 Excel 中存在错误，修改重传时又可能循环上述问题。

换思路，若直接通过 `Ctrl + C` 和 `Ctrl + V` 复制粘贴 Excel 中的数据到页面中，上述问题似乎都能解决。

试试实际效果： [戳我](http://aeodu.com/regular-excel-clipboard)

具体项目代码： [戳我](https://github.com/Deol/regular-excel-clipboard)

### 准备工作

1. 剪贴板操作

总共有 6 个剪贴板事件：

 - beforecopy：在发生复制操作前触发。
 - copy：在发生复制操作时触发。
 - beforecut：在发生剪切操作前触发。
 - cut：在发生剪切操作时触发。
 - beforepaste：在发生粘贴操作前触发。
 - **paste：在发生粘贴操作时触发（划重点）。**

要访问剪贴板中的数据，可以使用 clipboardData 对象。

基本上在非 IE 类浏览器中，为了防止对剪贴板的未授权访问，只有在处理剪贴板事件期间 clipboardData 对象才能被访问。因此为了确保跨浏览器兼容性，建议在发生剪贴板事件期间使用该对象。

该对象有三个方法：getData()、setData() 和 clearData()。**其中，getData()
用于从剪贴板中取得数据，它接受一个参数，一般为 `text/plain`，即要取得的数据的格式。**

2. contenteditable

富文本编辑很多时候可以由 div 模拟 textarea 文本框实现，只使用需要 contenteditable 属性。但此时若文本中包含样式，HTML 及样式信息也会被贴入。此时想要往在表格单项中粘贴纯文字，就需要做过滤。

从 [W3C 规范草案](https://w3c.github.io/editing/contentEditable.html#contenteditable) 可知，contenteditable 分为以下几类：

 - inherit（默认值）
 - true / ""
 - false
 - events
 - caret
 - typing
 - **plaintext-only（直接将粘贴数据过滤为纯文本）**

那么，直接利用 `contenteditable="plaintext-only"` 就能解决问题。

### 核心实现

1. 获取剪贴板数据

```js
/**
 * 获取剪贴板中的表格数据，并将其处理成可用的数据
 * @param {Object} e 
 */
getClipboardData(e = {}) {
    let clipboard = e.event.clipboardData;
    let data = clipboard.getData('text/plain').trim();
    if(isSimpleString(data)) {
        return {
            type: 'string',
            data
        };
    }
    return {
        type: 'table',
        data: data.split((/\r\n?/g)).map((row = {}) => {
            return row.split('\t').map(item => item.trim());
        }).filter((item = {}) => {
            return item.some(subItem => !!subItem);
        })
    };
}
```

2. 执行粘贴操作

 - 流程图

![image](https://user-images.githubusercontent.com/4961878/32105865-9c876d04-bb5c-11e7-8def-c4ccd0ef0509.png)

 - 实现原理

   实际上，除非当前粘贴数据为纯文本且粘贴位置为表格项中，此时直接利用 `contenteditable="plaintext-only"` 实现粘贴操作。
   
   其它所有情况，都利用 `e.preventDefault()` 阻止默认事件，统一通过更新数据进而更新视图，利用数据驱动方式实现粘贴操作。

```js
/**
 * 粘贴操作，对用户进行粘贴的数据进行处理
 * @param {Object} e 
 */
paste(e) {
    let data = this.data;
    let { clipboard, table } = this.$refs;
    let excelInfo = this.getClipboardData(e);

    // 往表格中粘贴纯字符串时不做不做处理直接贴入，样式由单项的 contenteditable="plaintext-only" 去除
    if(excelInfo.type === 'string') {
        // 在输入框中粘贴字符串时直接阻止
        if([e.target, e.target.parentNode].indexOf(clipboard) > -1) {
            e && e.preventDefault();
            window.alert('只能粘贴 Excel 表格数据哦~');
        }
        return;
    }

    e && e.preventDefault();

    // 未设置过数据时直接贴入表格
    if(!ut.existTable(clipboard, table)) {
        this.updateClipboard(excelInfo.data);
        return;
    }
    
    // ...存在数据时的拼接覆盖处理逻辑
}
```
   
   而在数据发生变更后，直接利用 `{#include template}` 方式使模板重新渲染。


```js
/**
 * 更新操作区域的表格数据
 * @param {Array}   list 
 * @param {Boolean} concat
 * @param {Number}  reload
 */
updateClipboard(list = [], concat = false, reload = Math.random()) {
    let { clipboard, table } = this.$refs;
    // 拼接时，之前数据可能经过用户编辑，与当前 Model 不一致
    let prevList = ut.getTableData(clipboard, table);
    Object.assign(this.data, {
        list: concat ? prevList.concat(list) : list,
        content: `${boardTpl}<input type="hidden" data-reload=${reload} />`
    });
    this.$update();
}
```

### 后续处理

在完成 Excel 数据的复制粘贴以及处理后，下一步可以将该表格与系统表格，进行表头项关联操作。

![image](https://user-images.githubusercontent.com/4961878/32106143-6a2a41b4-bb5d-11e7-91c2-46048e00a8e1.png)

而若是贴入数据仅有一项或有对应规则，我们甚至可以执行自动绑定，简化操作流程。

### 其他总结

 - 模拟 placeholder

利用 div 的 `contenteditable="true"` 可以实现 textarea 模拟功能，但是在 div placeholder 属性是不生效的。

此时可以这么处理：

```html
<!-- HTML -->
<div data-placeholder="在这里粘贴从Excel表中复制的数据，注意确保粘贴表头哦~"></div>

<!-- CSS -->
<style>
    div:empty:before {
        content: attr(data-placeholder);
    }
</style>
```
