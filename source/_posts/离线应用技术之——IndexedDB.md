想要构建离线应用，除了使用service worker，另一个绕不开的话题便是IndexedDB。

IndexedDB是浏览器端的一个基于键值对存储的**事务型**数据库。为什么要用它，因为想要做到在离线情况下展示数据，数据的持久化是离线应用绕不过去的一个坎。以前使用的是Web SQL，不过它已经被废弃掉了，所以never mind。什么，你说localstorage？确实有很多人会使用localstorage来存储数据，但是相比IndexedDB，它存在很大的不足，原因有三：

1. localstorage 存储有大小限制，限制5MB，不能存储大量数据，尤其是带有结构的数据。

2. 没有查询语句，没有schema，基本上没有任何有关数据库的操作。每次的写入和写出都要字符串化和对象化，何其麻烦。所以在处理带结构的大型数据上基本毫无扩展性。

3. 最关键的一点是，localstorage的API是同步的，这就意味着它会阻塞DOM操作。并且很多时候，离线应用的数据操作需要在service worker中进行，service worker只接受异步的API，所以相较而言，IndexedDB是更好的选择。

下面这张表摘自[张鑫旭的博文](http://www.zhangxinxu.com/wordpress/2017/07/html5-indexeddb-js-example/)，改装了一下，可以快速了解IndexeDB的一些基础特性。


|               | IndexedDB |
| ------------- |:-------------|
| **优点**      | 1. 允许对象的快速索引和搜索，因此在Web应用程序场景中，您可以非常快速地管理数据以及读取/写入数据；2. 由于是NoSQL数据库，因此我们可以根据实际需求设定我们的JavaScript对象和索引；3. 在异步模式下工作，每个事务具有适度的粒状锁。这允许您在JavaScript的事件驱动模块内工作。|
| **不足**      | 如果你的世界观里面只有关系型数据库，恐怕不太容易理解。      |
| **位置** | 包含JavaScript对象和键的存储对象。      |
| **查询机制** | Cursor APIs，Key Range APIs，应用程序代码      |
| **事务** | 锁可以发生在数据库版本变更事务，或是存储对象“只读”和“读写”事务时候。      |
| **事务提交** | 事务创建是显式的。默认是提交，除非我们调用中止或有一个错误没有被捕获。|

也许，上表中说到的一些概念你还不是很懂。嘿，您先别急，先坐下，且听我慢慢给您说。

**Q1**：什么是NoSQL数据库？
**A1**：非关系型数据库，其中的一大类便是通过键值（key-value）存储数据。IndexedDB便是属于这一类。

**Q2**：什么是事务？
**A2**：指访问并可能更新数据库中各种数据项的一个程序执行单元(unit)。它有以下四点特性：

- 原子性(Atomicity)：事务中的所有操作作为一个整体提交或回滚。
- 一致性(Consistemcy)：事物完成时，数据必须是一致的，以保证数据的无损。
- 隔离性(Isolation)：对数据进行修改的多个事务是彼此隔离的。
- 持久性(Durability)：事务完成之后，它对于系统的影响是永久的，该修改即使出现系统故障也将一直保留。

**Q3**：Cursor是干嘛用的？
**A3**：cursor即游标，类似于现实中的游标，一个刻度表示一行数据，游标就是尺子上的一片区域，想要获得数据库一行一行的数据，我们可以遍历这个游标就好了。

前菜上的差不多了，现在进入我们的正餐部分。

### 如何使用IndexedDB？

使用IndexedDB其实还蛮简单的，你只需要做两件事：

1. 创建或打开一个数据库

2. 创建一个Object Store对象仓库（它是IndexedDB存储数据的机制，习惯了关系型数据库的同学可以把它想象成一张表）

![IndexedDB](https://i.stack.imgur.com/zGJPJ.png)

遵循上面两步，我们便可以开始愉快的使用IndexedDB了。代码如下：

```javascript
// 创建或使用一个数据库
const request = window.indexedDB.open("todos", 1)

// 创建schema, 如果浏览器没有找到todos数据库，则它会创建一个，同时触发upgradeneeded事件。
request.onupgradeneeded = event => {
  //db代表的是与数据库的连接
  const db = event.target.result;
  
  //首先检测是否存在我们要创建的Object Store
  if(!db.objectStoreNames.contains("todo-meta")) {
    //通过createObjectStore方法创建ObjectStore对象来存储对象
    const todoStore = db.createObjectStore('todo-meta', {keyPath: 'id'});
    
    //使用index来检索数据，createIndex方法接受两个参数，一个代表index的name，一个代表要被indexed的属性
    todoStore.createIndex('nameIdx', 'name');
  }
  
  if(!db.objectStoreNames.contains("todo-items")) {
    // keyPath也可以传递一个数组，来共同组成可以索引
    const itemStore = db.createObjectStore(
        "todo-items",
        { keyPath: [ "todoId", "row" ] }
    );
    itemStore.createIndex("todoIndex", "todoId");
  }
    
  if(!db.objectStoreNames.contains("attachments")) {
    // 不传keypath也行，交由数据库自己处理key，适合于一些文件内容的存储。
    const fileStore = db.createObjectStore(
        "attachments",
        { autoIncrement: true }
    );
  }
    
}
```
好吧，我撒谎了，从代码上看，使用IndexedDB并不是那么简单啊，这还是在我没有考虑兼容性的情况下，而忽略了以下这么一大段代码。

```Javascript
  // In the following line, you should include the prefixes of implementations you want to test.
window.indexedDB = window.indexedDB || window.mozIndexedDB || window.webkitIndexedDB || window.msIndexedDB;
  // DON'T use "var indexedDB = ..." if you're not in a function.
  // Moreover, you may need references to some window.IDB* objects:
window.IDBTransaction = window.IDBTransaction || window.webkitIDBTransaction || window.msIDBTransaction;
window.IDBKeyRange = window.IDBKeyRange || window.webkitIDBKeyRange || window.msIDBKeyRange;
  // (Mozilla has never prefixed these objects, so we don't need window.mozIDB*)
```

来自中外各大名宿的吐槽，IndexedDB的API是出了名的膈应人。就比如说新建一个数据库，为啥我要给它取名叫request而不是idb之类的。事实上，对IndexedDB来说，新建数据库是一个request请求，用MDN的原话是，`IndexedDB uses a lot of requests`。通过这些请求，它能够获取到相应的DOM事件，从而判断该操作是成功了还是失败了。同时，这些请求也有readyState，result和errorCode等一系列属性。这怎么看都像是一个XMLHTTPRequest对象，简直不能再坑了。

吐槽归吐槽，我们还是认真看一下上一段代码做了什么。在连接数据库后，我们通过createObjectStore方法新建了三个对象仓储，第一个参数即为仓储名，第二个参数即为配置项，包含两个属性：1. keyPath，2. autoIncrement。keyPath用于指定对象的键，如果未指定，则对象的创建使用的是out-of-line keys；指定了，则使用in-line keys。至于什么是out-of-line keys和in-line keys，我们通过一图流来进行详细的说明。

![key](https://haitao.nos.netease.com/22c25240-834c-4676-ae60-215507fce160.png)

可以这么理解，out-of-line keys即为单独生成的一个key，可能需要我们自己指定。而in-line keys则是指定对象的一个属性作为key值，由数据库自动绑定。

至于autoIncrement属性，可以看成一个key generator，自动生成key值。一般来说，keyPath和autoIncrement属性只要使用一个就够了，如果两个同时使用，表示键名为递增的整数，且对象不得缺少指定属性。

当然，除了使用key存储对象，也可以为对象指定index。就像上面代码中的createIndex做的那样，我们依然通过一图流来说明key和index的区别。

![entry](https://haitao.nos.netease.com/150121c1-0d43-4758-987a-01505187998a.png)

可以看到，两者其实是同一份数据，只是由不同的属性索引，当需要检索某一特定属性的数据时，index格外有用。

下面讲讲如何进行数据库的CRUD操作，毕竟这才是我们真正关心的。

执行IndexedDB的CRUD操作只需要如下五步：

1. 与数据库建立连接
2. 创建一个事务
3. 指明Object Store
4. 在该store上执行操作
5. 清除数据库连接

#### 1. 添加数据

```javascript
const request = window.indexedDB.open("todos", 1);

request.onsuccess = () => {
    const db = request.result;
    
    // db.transaction的第一个参数是一个数组，表明我们希望事务执行操作的ObjectStore列表，通常是只有一个。第二个参数表明事务执行的操作，一般为readonly（只读）和readwrite（读写）
    const transaction = db.transaction(
        [ "todo-meta", "todo-items" ],
        "readwrite"
    );
    
   // 事务使用的两个store
    const metaStore = transaction.objectStore("todo-meta");
    const itemStore = transaction.objectStore("todo-items");
  
    // 新增数据
    metaStore.add(
        { todoDate: 112342392131, todoAuthor: 'luffy' }
    );
    
    itemStore.add({
        todoId: 1,
        row: 1,
        name: 'todo 1',
        completed: false
    });
  
    // 清除数据库连接
    transaction.oncomplete = () => {
        db.close();
    };
```

#### 2. 更新数据

```javascript
// ...如上连接数据库，创建事务

// 这个操作会通过keypath来更新数据，itemStore的keyPath为属性todoId和row的结合，可以看到上个操作新建的todo item的completeed属性被更新为true
const itemStore = objectStore.put({
      todoId: 1,
      row: 1,
      name: 'todo 1',
      completed: true
  });
  
// 当然，objectStore.put也可以接受第二个参数，表明对象的键，一般用于out-of-line key的情况。
attachmentStore.put(VALUE, KEY);

// ...清除数据库连接

```

#### 3. 删除数据

```Javascript
// ...如上连接数据库，创建事务

// 大概是最简单的API了，指明我们要删除的key就行了
itemStore.delete([ "123", "2" ]);

// ...清除数据库连接
```

#### 4. 读取数据

读取操作有那么点不同，因为它会新建一个request，读取数据在request的回调中。

```javascript
// ...如上连接数据库，创建事务

// 通过监听onsuccess来获取数据，通过键名获取指定数据
const getRequest = itemStore.get("123");
// 也可以通过getAll方法获取全部数据
const getAllRequest = itemStore.getAll();

getRequest.onsuccess = () => {
    // 获取的数据在request的result属性中
    console.log(getRequest.result);
};
```

#### 5. 遍历数据

get()和getAll()可以获取单个数据或全部数据，但如果想进行更精细的读取操作，比如读取3-20范围的数据，则需要用到cursor及IDBKeyRange两个对象了。

索引的有用之处，在于可以指定读取数据的范围

IDBKeyRange对象的作用则是生成一个表示范围的Range对象。它的生成方法有四种

> lowerBound方法：指定范围的下限。
> upperBound方法：指定范围的上限。
> bound方法：指定范围的上下限。
> only方法：指定范围中只有一个值。

下面的代码直接摘自[阮一峰老师的文章](http://javascript.ruanyifeng.com/bom/indexeddb.html)，仅供参考

```javascript
// All keys ≤ x	
var r1 = IDBKeyRange.upperBound(x);

// All keys < x	
var r2 = IDBKeyRange.upperBound(x, true);

// All keys ≥ y	
var r3 = IDBKeyRange.lowerBound(y);

// All keys > y	
var r4 = IDBKeyRange.lowerBound(y, true);

// All keys ≥ x && ≤ y	
var r5 = IDBKeyRange.bound(x, y);

// All keys > x &&< y	
var r6 = IDBKeyRange.bound(x, y, true, true);

// All keys > x && ≤ y	
var r7 = IDBKeyRange.bound(x, y, true, false);

// All keys ≥ x &&< y	
var r8 = IDBKeyRange.bound(x, y, false, true);

// The key = z	
var r9 = IDBKeyRange.only(z);
```

通过IDBKeyRange，结合cursor，便可以实现在一定范围内读取数据的操作了。

```javascript
// 想要遍历数据，就要openCursor方法，它在当前对象仓库里面建立一个读取光标（cursor）

// 绑定range
var range = IDBKeyRange.bound('3', '20');

const cursor = db.trasaction(['todo-item'], 'readOnly').objectStore('todo-item').openCursor(range);

//回调函数接受一个事件对象作为参数，该对象的target.result属性指向当前数据对象。当前数据对象的key和value分别返回键名和键值（即实际存入的数据）。continue方法将光标移到下一个数据对象，如果当前数据对象已经是最后一个数据了，则光标指向null。

cursor.onsuccess = function(e) {
    var res = e.target.result;
    if(res) {
        console.log("Key", res.key);
        console.dir("Data", res.value);
        res.continue();
    }
}

```

###结语
终于把IndexedDB的API给走了一遍，基本上涵盖了我们日常开发的大部分操作，当然还有一部分API可以直接从[MDN](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)上查阅。可以看到，这些API不可谓不繁琐，如果直接使用这些API，估计你们不是累死就是被气死。

正所谓哪里有压迫哪里就有反抗，我们的[Jake Archibald](https://twitter.com/jaffathecake)大神在官方的基础上封装了一个Promise风格的库--[idb](https://github.com/jakearchibald/idb)，可以方便开发者们按照现代JavaScript的方式使用IndexedDB。这里有[一篇文章](https://medium.freecodecamp.org/a-quick-but-complete-guide-to-indexeddb-25f030425501)便是基于这个库来介绍IndexedDB的，写的相当不错。

下面这行代码大概展示了使用idb来写出promise及async风格的代码，来源于[medium](https://medium.com/@filipvitas/indexeddb-with-promises-and-async-await-3d047dddd313)。

``` JavaScript
async function getAllData() {
    let db = await idb.open('db-name', 1)

    let tx = db.transaction('objectStoreName', 'readonly')
    let store = tx.objectStore('objectStoreName')

    // add, clear, count, delete, get, getAll, getAllKeys, getKey, put
    let allSavedItems = await store.getAll()

    console.log(allSavedItems)

    db.close()
}
```

感兴趣的同学也可以看看这个简单的使用idb实现的[todoList](https://github.com/AIluffy/todo-list-indexeddb)。

以上，XD。

by zhangxueai@corp.netease.com
