---
title: 浅谈浏览器本地存储-indexedDB
date: 2017-03-22
---

indexedDB 是 HTML5 提供的一种本地存储，一般用户保存大量用户数据并要求数据之间有搜索需要的场景,当网络断开的时候，可以做一些离线应用，它比 SQL 方便，不用去写一些特定的语句对数据进行操作，数据格式为 json。

<!-- more -->

### 步骤：
#### 1. 创建数据库，并指定数据库的版本号

```
var indexedDB = window.indexedDB || window.webkitIndexedDB || window.mozIndexedDB || window.msIndexedDB;
var request = indexedDB.open(databasename, version);
request.onerror = function(e){
    // 创建失败回调函数
};
request.onsuccess = function(e){
    // 创建成功回调函数
};
request.onupgradeneededd = function(e){
    // 当数据库改变是回调函数
};
```

> 注意这里的版本号只可以是整数

#### 2. 建立对象存储空间

```
request.onupgradeneeded = function(event) { 
  var db = event.target.result;
  var objectStore = db.createObjectStore("name", { keyPath: "id" });
};
```

#### 3. 数据的增、删、改、查

```
// 增
addData:function(db,storename,data){
    var store = store = db.transaction(storename,'readwrite').objectStore(storename),request;
    for(var i = 0 ; i < data.length;i++){
        request = store.add(data[i]);
        request.onerror = function(){
            console.error('add添加数据库中已有该数据')
        };
        request.onsuccess = function(){
            console.log('add添加数据已存入数据库')
        };
    }

}
```

> 添加数据，重复添加会报错

```
// 删
deleteData:function(db,storename,key){
    var store = store = db.transaction(storename,'readwrite').objectStore(storename);
    store.delete(key)
    console.log('已删除存储空间'+storename+'中'+key+'记录');
}
```

```
// 改
putData:function(db,storename,data){
    var store = store = db.transaction(storename,'readwrite').objectStore(storename),request;
    for(var i = 0 ; i < data.length;i++){
        request = store.put(data[i]);
        request.onerror = function(){
            console.error('put添加数据库中已有该数据')
        };
        request.onsuccess = function(){
            console.log('put添加数据已存入数据库')
        };
    }
}
```

> 重复添加已经存在的数据会更新原有数据

```
// 查
getDataByKey:function(db,storename,key){
    var store = db.transaction(storename,'readwrite').objectStore(storename);
    var request = store.get(key);
    request.onerror = function(){
        console.error('getDataByKey error');
    };
    request.onsuccess = function(e){
        var result = e.target.result;
        console.log('查找数据成功')
        console.log(result);
    };
}
```

> 根据存储空间的键找到对应数据，本例为 id

#### 4. 关闭数据库

```
db.close();
```

#### 2.使用场景

掌握了使用步骤之后，我们来结合它的优、缺点谈谈其使用场景。
- 不适合过大的数据存储，浏览器对 indexDB 有 50M 大小的限制
- 不适合对兼容性要求高的项目
- 不适合存储敏感数据
- 当用户清除浏览器缓存的时候可能出现问题
- indexedDB 受到同源策略的限制 

> 参考: [indexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)