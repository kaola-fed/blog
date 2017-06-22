---
title: 初识mongoDB（二）
date: 2017-06-05
---

## 常见的基本操作
- **列出所有/当前的数据库**
    
    ```
    show dbs
    ```
    ![image](https://haitao.nos.netease.com/49cc04e3-b972-40e2-a900-679e171712d8.png)
    
    **Q:为什么连接了test数据库，在显示所有数据库的时候竟没有test呢？**
    
    > A:这是因为连接test时，并没有向其中写数据，所以并没有真正的建立、只是在使用当前的数据库。

<!-- more -->

- **删除当前的数据库**
    ```
    db.dropDatabase()
    ```
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftfzknzqtj20qi0fy405.jpg)

## 集合的操作
    
- **创建集合**

    方法一：
    
    ```
    db.createCollection(COLLECTION_NAME, options)
    ```
     ![image](https://haitao.nos.netease.com/6c4db96f-28db-4aee-9215-548b2d7dec8e.png)
    
    方法二：
        
    ```
    db.COLLECTION_NAME.insert({})
    ```
     ![image](https://haitao.nos.netease.com/1cf05c05-a129-4343-9f88-590e9c1d0937.png)

    > COLLECTION_NAME：集合名称
    
- **查看当前数据库中创建的集合**

    ```
    show collections
    ```  
   
- **删除当前数据库中的某些集合**

    ```
    db.COLLECTION_NAME.drop()
    ```
    ![image](https://haitao.nos.netease.com/1f12f416-0893-4fa2-9d04-842e33c11402.png)
    
- **访问集合**
    方法一：
    ```
    db.COLLECTION_NAME
    ```
   ![image](https://haitao.nos.netease.com/67238d74-9545-490a-b6ba-f52f5f61d4ee.png)
    方法二：
        
    ```
    db.getCollection(COLLECTION_NAME)
    ```
    ![image](https://haitao.nos.netease.com/9203aa3f-5fbf-44e6-81bc-a6b13ffceae0.png)    
    
    注意：COLLECTION_NAME 要加引号哦
    
- **列出当前文档中所有的集合**
    
    ```
    show tables
    ```
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftcdqodpoj20qi06gwf7.jpg)


## 操作集合的几个基本命令

- **在集合中插入一条数据**
    方法一：
    
    ```
    db.COLLECTION_NAME.insert({})
    db.COLLECTION_NAME.insert([{}, {}]) //插入多条数据
    ```
    如果我们不指定_id参数，mongodb会自动分配一个独特的ObjectId。（个人理解：_id就是一个uuid通用唯一识别码）
    
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftbpbnho0j20qm01qq36.jpg)
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftbo3pv2wj20qk0dy0ud.jpg)
    
    方法二：
    ```
    db.COLLECTION_NAME.save({})
    ```
    区别：
    > 一、使用save函数里，如果原来的对象不存在，那他们都可以向collection里插入数据，如果已经存在，save会调用update更新里面的记录，而insert则会忽略操作 
    >
    > 二、insert可以一次性插入一个列表，而不用遍历，效率高， save则需要遍历列表，一个个插入。 
    >
    > https://my.oschina.net/u/1590519/blog/342765
    
    举个简单的例子:
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftbxq69hoj20qg0hu76j.jpg)
    
- **删除集合中的数据**
    方法一：
    
    ```
    db.COLLECTION_NAME.remove()
    ```
    方法二：
    
    ```
    db.COLLECTION_NAME.drop()
    ```
    
    区别：
    缺点：drop()不能指定任何限定条件。整个集合都被删除了，所有元数据也都不见了。
    优点：drop()的速度快很多
    
- **查找集合中的数据**

    ```
    db.COLLECTION_NAME.find():
    ```
    
    默认返回所有数据
    
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftio9o1pyj20qi0nqgs6.jpg)
    
    > 返回包含x: 1的数据
    
    Q: 返回的数据中不希望包含_id字段怎么办?
    
    A: 将不想的设置为0
    
    ```
    db.COLLECTION_NAME.find({}, {_id: 0}):
    ```
    
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftiqxslspj20qg0kumyz.jpg)
    
- **对查找后的结果进行操作**

    - **计数**
    
        ```
        db.COLLECTION_NAME.find().count()
        ```
        
        ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftigzv529j20qg0mmjxi.jpg)
        
    - **跳过n条数据**

        ```
        db.COLLECTION_NAME.find().skip(n)
        ```
        
        ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftihmtq11j20qm022mxg.jpg)
        
    - **限制返回的数据条数**

        ```
        db.COLLECTION_NAME.find().limit(n)
        ```
        
        ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftii70kx9j20qi05mta5.jpg)

    - **使用x排序：1升序，-1降序**

        ```
        db.COLLECTION_NAME.find().sort({x: 1})
        ```
        
        ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftii70kx9j20qi05mta5.jpg)
        
- **返回特定条件的内容**
   "$lt"、"$lte"、"$gt"、"$gte"、"$ne"就是全部的比较操作符
        

- **更新集合中的数据**
    
    ```
    db.COLLECTION_NAME.update(SELECTIOIN_CRITERIA, UPDATED_DATA)
    ```
    SELECTIOIN_CRITERIA：用来定位到需要选择的文档
    UPDATED_DATA：修改后的数据
    
    **注：** 会更新匹配到的第一个文档(如下图所示)
    
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftgruznp1j20qe0hyq62.jpg)
    
    > $set: 部分更新操作符
    
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftgu83fqrj20qk07i0uc.jpg)
    
    上图中{x: 1, y: 1}被整个更新掉了；解决办法是使用 **$set** 操作符如下：
    
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1fftgwvig8pj20qe07omys.jpg)
    
    **$unset** 删除某个不需要的键

    ***Q：更新的时候如果匹配不到相应的文档，怎么自动创建一条呢？***
    
    ***A：*** update() 可以传入第三个参数，true, 如果匹配不到相应的文档那么就创建一条文档出来。
    
    ![image](https://ww1.sinaimg.cn/large/86b9ad20gy1ffth2nhmnwj20qe0c840s.jpg)
    ## mongoDB索引

索引的好处：
    加快索引相关的查询
    
索引的不好处：
    增加磁盘空间消耗，降低写入性能

## 基本语法
ensureIndex() 方法

> db.COLLECTION_NAME.ensureIndex({KEY:1})  

Key 值为你要创建的索引字段，1为指定按升序创建索引，-1降序；

可选字段如下：

Parameter | Type | Description  
---|---|---
background | Boolean |  建索引过程会阻塞其它数据库操作，background可指定以后台方式创建索引，即增加 "background" 可选参数。 "background" 默认值为false。
unique | Boolean |  建立的索引是否唯一。指定为true创建唯一索引。默认值为false.
name  | string |    索引的名称。如果未指定，MongoDB的通过连接索引的字段名和排序顺序生成一个索引名称。
dropDups  |     Boolean  | 在建立唯一索引时是否删除重复记录,指定 true 创建唯一索引。默认值为 false.
sparse |    Boolean   | 对文档中不存在的字段数据不启用索引；这个参数需要特别注意，如果设置为true的话，在索引字段中不会查询出不包含对应字段的文档.。默认值为 false.
expireAfterSeconds |    integer  |指定一个以秒为单位的数值，完成 TTL设定，设定集合的生存时间。
v  |    index version  |    索引的版本号。默认的索引版本取决于mongod创建索引时运行的版本。
weights  |  document  |     索引权重值，数值在 1 到 99,999 之间，表示该索引相对于其他索引字段的得分权重。
default_language |  string  |   对于文本索引，该参数决定了停用词及词干和词器的规则的列表。 默认为英语
language_override |     string |    对于文本索引，该参数指定了包含在文档中的字段名，语言覆盖默认的language，默认值为 language.

---

### _id索引

![image](https://haitao.nos.netease.com/c9a721ad-f9a7-45a8-b68a-5aaa6ff57aac.png)

默认就创建了_id索引

---

### 单键索引

![image](https://haitao.nos.netease.com/4493eeb7-a523-4ea9-ab7d-77c60571b3f3.png)

### 多键索引
值具有多个记录

![image](https://ww1.sinaimg.cn/large/86b9ad20gy1ffxfysarjgj213g020wes.jpg)

### 复合索引

查询条件不只有一条时

![image](https://ww1.sinaimg.cn/large/86b9ad20gy1ffxg6chxskj20jy06l0tu.jpg)

### 过期索引

一段时间后索引会过期，比如日志信息等

> db.foo.ensureIndex({"time" : 1}, {"expireAfterSecs" : 30})

expireAfterSecs指定的值以秒为单位 

过期索引不能是复合索引

### 全文索引


```
db.stores.insert(
   [
     { _id: 1, name: "Java Hut", title: "Coffee and cakes" },
     { _id: 2, name: "Burger Buns", title: "Gourmet hamburgers" },
     { _id: 3, name: "Coffee Shop", title: "Just coffee" },
     { _id: 4, name: "Clothes Clothes Clothes", title: "Discount clothing" },
     { _id: 5, name: "Java Shopping", title: "Indonesian goods" }
   ]
)
```

建立文本索引的方法：

```
> db.COLLECTION_NAME.ensureIndex({key: "text"})  
> db.COLLECTION_NAME.ensureIndex({key_1: "text", key_2: "text"})  
> db.COLLECTION_NAME.ensureIndex({"$**": "text"})  
```

查询全文索引的方法：

```
> db.COLLECTION_NAME.find({$text:{$search: ""}})  
> db.COLLECTION_NAME.find({$text:{$search: "key1 key2"}})    //key1或者key2
> db.COLLECTION_NAME.find({$text:{$search: "key1 -key2"}})   //不好含key2
> db.COLLECTION_NAME.find({$text:{$search: "\"key1\" \"key2\""}})  
```

相似度：
 {score: {$meta: "textScore"}}
> db.COLLECTION_NAME.find({$text:{$search: ""}}, {score: {$meta: "textScore"}}).sort({score: {$meta: "textScore"}})

