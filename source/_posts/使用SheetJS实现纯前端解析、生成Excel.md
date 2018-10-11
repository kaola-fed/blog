---
title: 使用SheetJS实现纯前端解析、生成Excel
date: 2018-03-28
---

# 使用SheetJS实现纯前端解析、生成Excel


### 前言

目前大多数后台系统的导入、导出功能都是在后端解析、生成Excel，前端只是负责搬运。其实纯前端也是可以实现解析、创建Excel的，通过笔者的调研，自认为[SheetJS](https://github.com/SheetJS/js-xlsx)是当前最成熟的处理Excel的javascript插件。

下面简单介绍下这个插件的用法。

### 如何生成Excel

开始介绍之前，我们需要先理清两个概念：

    - workbook：简写wb， 一份Excel文件对应一个workbook
    
    - worksheet：简写ws， 对应Excel文件下的sheet，一个workbook可以包含多个worksheet

#### 1. 创建workbook

SheetJS提供了多个创建workbook的方法，这里只列举浏览器端常用的方法，高级用法以及node端的用法暂不列举，有兴趣可以自行查阅，[更多用法](https://sheetjs.gitbooks.io/docs/#parsing-workbooks)：

- XLSX.utils.book_new()：创建一个干净的workbook

```javascript
    var wb = XLSX.utils.book_new();
```

- XLSX.utils.table_to_book(table)：参数table为dom节点，根据传入的table元素，生成包含对应内容的workbook

```html
    <table id="tableau">
    	<thead>
    		<tr>
    			<th>This</th>
    			<th>is</th>
    			<th>a</th>
    			<th>Test</th>
    		</tr>
    	</thead>
    	<tbody>
    		<tr>
    			<td>hello</td>
    			<td>world</td>
    			<td>你</td>
    			<td>好</td>
    		</tr>
    		<tr>
    			<td>1</td>
    			<td>2</td>
    			<td>3</td>
    			<td>4</td>
    		</tr>
    	</tbody>
    </table>
```

```javascript
    var ws = XLSX.utils.table_to_sheet(document.getElementById('tableau'));
```

- XLSX.read(htmlstr, {type:'string'})：传入table的html字符串，生成包含对应内容的workbook

```javascript
    var html_str = document.getElementById('tableau').outerHTML;
	var workbook3 = XLSX.read(html_str, {type:'string'});
```

#### 2. 创建worksheet

同上，SheetJS提供了多个创建worksheet的方法，这里只列举浏览器端常用的方法，高级用法以及node端的用法暂不列举，有兴趣可以自行查阅，[更多用法](https://sheetjs.gitbooks.io/docs/#writing-functions)：

- XLSX.utils.table_to_sheet(table)：table为dom节点，根据传入的table元素，生成包含对应内容的worksheet
```html
    <table id="tableau">
    	<thead>
    		<tr>
    			<th>This</th>
    			<th>is</th>
    			<th>a</th>
    			<th>Test</th>
    		</tr>
    	</thead>
    	<tbody>
    		<tr>
    			<td>hello</td>
    			<td>world</td>
    			<td>你</td>
    			<td>好</td>
    		</tr>
    		<tr>
    			<td>1</td>
    			<td>2</td>
    			<td>3</td>
    			<td>4</td>
    		</tr>
    	</tbody>
    </table>
```

```javascript
    var ws = XLSX.utils.table_to_sheet(document.getElementById('tableau'));
```

- XLSX.utils.json_to_sheet(json): 根据传入的json数据，生成包含对应内容的worksheet

```javascript
    var ws = XLSX.utils.json_to_sheet([
      { A:"S", B:"h", C:"e", D:"e", E:"t", F:"J", G:"S" },
      { A: 1,  B: 2,  C: 3,  D: 4,  E: 5,  F: 6,  G: 7  },
      { A: 2,  B: 3,  C: 4,  D: 5,  E: 6,  F: 7,  G: 8  }
    ], {header:["A","B","C","D","E","F","G"], skipHeader:true});
```

- XLSX.utils.aoa_to_sheet(ws_data): 根据传入的二维数组，生成包含对应内容的worksheet

```javascript
    var ws_data = [
      [ "S", "h", "e", "e", "t", "J", "S" ],
      [  1 ,  2 ,  3 ,  4 ,  5 ]
    ];
    var ws = XLSX.utils.aoa_to_sheet(ws_data);
```
- XLSX.utils.sheet_add_json(ws, ws_data): 向指定的sheet追加数据，结合XLSX.utils..json_to_sheet(json)这个方法比较容易理解

```javascript
    XLSX.utils.sheet_add_json(ws, [
      { A: 4, B: 5, C: 6, D: 7, E: 8, F: 9, G: 0 }
    ], {header: ["A", "B", "C", "D", "E", "F", "G"], skipHeader: true, origin: -1});
```

- XLSX.utils.sheet_add_aoa(ws, ws_data): 向指定的sheet追加数据，结合XLSX.utils.aoa_to_sheet(ws_data)这个方法比较容易理解

```javascript
    var ws_data = [
      [  4 ,  5 ,  6 ,  7 ,  8 ]
    ];
    var ws = XLSX.utils.sheet_add_aoa(ws, ws_data, {origin:-1});
```

#### 3. 将worksheet添加进workbook

```javascript
    XLSX.utils.book_append_sheet(workbook, worksheet, "sheetName");
```

#### 4. 生成Excel文件

```javascript
    var s2ab = function (s) { // 字符串转字符流
		var buf = new ArrayBuffer(s.length)
		var view = new Uint8Array(buf)
		for (var i = 0; i !== s.length; ++i) {
			view[i] = s.charCodeAt(i) & 0xFF
		}
		return buf
	}
	// 创建二进制对象写入转换好的字节流
	let tmpDown =  new Blob([s2ab(XLSX.write(wb1,{bookType: 'xlsx', bookSST: false, type: 'binary'} ))], {type: ''}) 
	let a = document.createElement('a');
	// 利用URL.createObjectURL()方法为a元素生成blob URL
	a.href = URL.createObjectURL(tmpDown)  // 创建对象超链接

	a.download = '商品列表.xls';
	a.click();
```

### 解析Excel示例：

```html
    <div id="demo">
        <input type="file" onChange="app.importFile(event)" id="imFile"
            accept="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet, application/vnd.ms-excel"/>
    </div>
```

```javascript
    const app = {	
        importFile: function(e){
            let imFile = document.getElementById('imFile');
            let f = imFile.files[0];
            // 调用FileReader读取文件
            let reader = new FileReader();
            reader.onload = function (e) {
                let data = e.target.result;
                let workbook = XLSX.read(data, {
                    type: 'binary'
                })
                console.log('workbook:', workbook);
                /*
                    workbook数据结构
                    {
                       SheetNames['sheet1', 'sheet2'],
                       Sheets:{
                           'sheet1':{...},
                           'sheet2':{...},
                       }
                    }
                */
                let json = XLSX.utils.sheet_to_json(workbook.Sheets[workbook.SheetNames[0]]);
                console.log('json:', json);
                /*
                    json数据结构
                    [
                        { A:"S", B:"h", C:"e", D:"e", E:"t", F:"J", G:"S" },
                        { A: 1,  B: 2,  C: 3,  D: 4,  E: 5,  F: 6,  G: 7  },
                        { A: 2,  B: 3,  C: 4,  D: 5,  E: 6,  F: 7,  G: 8  }
                    ]
                */
            }
            reader.readAsBinaryString(f);

        }
    }
```

### 总结
纯前端解析、导出Excel的关键在于调用了Blob、FileReader、URL.createObjectURL这几个API，SheetJS在此基础上进行了更深层次的封装，针对IE等老旧浏览器进行了兼容，并且支持node等环境使用，提供了丰富的API，本文只列举了浏览器端常规使用的方法，更多细节需自行查阅官方文档。

## 参考资源:
[SheetJS](https://github.com/SheetJS/js-xlsx)

[SheetJS docs](https://sheetjs.gitbooks.io/docs/#sheetjs-js-xlsx)

[纯前端实现excel表格导入导出](https://segmentfault.com/a/1190000011057149)


