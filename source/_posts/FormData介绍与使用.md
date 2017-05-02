---
title: FormData介绍与使用
date: 2017-04-01
---

## FormData
XMLHttpRequest Level 2添加了一个新的接口FormData.在开发中使用后，觉得是form表单的加强版。可以异步的，灵活的发送数据，并且包括上传二进制文件

<!-- more -->

## 创建FormData
创建FormData的方式
- 直接创建
```
var formData = new FormData();
formData.append('businessId', '3049');
formData.append('companyName', 'test');
```
- 利用form节点初始化
```
<form enctype="multipart/form-data" method="post" name="fileinfo">
  <label>Your email address:</label>
  <input type="email" autocomplete="on" autofocus name="userid" placeholder="email" required size="32" maxlength="64" /><br />
  <label>Custom file label:</label>
  <input type="text" name="filelabel" size="12" maxlength="32" /><br />
  <label>File to stash:</label>
  <input type="file" name="file" required />
</form>
```
```
var fData = new FormData(document.forms.namedItem('fileinfo'));
```
## method
### append
FormData.append()会添加一个新值到 FormData 对象内的一个已存在的键中，如果键不存在则会添加该键,如果存在则会加在原有的值后面.即类似于数组的2号位
```
formData.append(name,value);
formData.append(name,value,fileName);
```
name: key(键)
value: 值
fileName: 当value为Blob或者file的时候,fileName为文件名
### delete
FormData.delete()会删除一对键值
```
formData.delete(name);
```
### entries
FormData.entries()会返回一个iterator对象，我们可以通过它遍历FormData的键值.
```
var formData = new FormData();
formData.append('businessId', '3049');
formData.append('companyName', 'test');
for(var pair of formData.entries()) {
   console.log(pair[0]+ ', '+ pair[1]); 
}
```
结果为
```
businessId, 3049
companyName, test
```
### get
FormData.get()用于获取key所对应的value的首位值
```
var formData = new FormData();
formData.append('businessId', '3049');
formData.append('businessId', '3050');
console.log(formData.get('businessId'));
```
结果为
```
3049
```
### getAll
FormData.getAll()用于获取key所对应的value
的全部值。如果对应的值超过一个，则会返回一个数组.
```
var formData = new FormData();
formData.append('businessId', '3049');
formData.append('businessId', '3050');
console.log(formData.get('businessId'));`
```
结果为
```
['3049','3050'];
```
### has
FormData.has()用于检查是否含有指定的key。返回一个布尔值，true有，false即没有
```
var formData = new FormData();
formData.has('businessId'); //false
formData.append('businessId','3049');
formData.has('businessId'); //true
```
### keys
FormData.keys()会返回一个iterator对象，对于遍历所有的key
```
var formData = new FormData();
formData.append('businessId', '3049');
formData.append('companyName', 'test');
for (var key of formData.keys()) {
   console.log(key); 
}
```
结果为
```
businessId
companyName
```
### set
FormData.set()方法会对FormData对象里的某个key设置一个新的值，如果该key不存在，则添加。如果存在，不管之前key所对应的值有几个，都会被完全覆盖。
```
var formData = new FormData();
formData.append('businessId', '3049');
formData.append('businessId', '3050');
formData.set('businessId','3051');
console.log(formData.getAll('businessId'));
```
结果为
```
['3051']
```
### values
FormData.values()会返回一个iterator对象，values
```
var formData = new FormData();
formData.append('businessId', '3049');
formData.append('companyName', 'test');
for(var value of formData.values()) {
   console.log(value); 
}
```
结果为
```
3049
test
```
## 兼容性
| Feature | Chrome | Firfox(Gecko) | Intenet Explorer | Opera | Safari |
| ------------- |:-------------:|:-------------:|:-------------:|:-------------:|:-------------:|
| Basic support | 7+ | 4.0(2.0) | 10+ | 12+ | 5+ |
| append with filename | (Yes) | 22.0(22.0) | ? | ? | ? |
| delete, get, getAll, has, set | Behind Flag | Not supported | Not supported | (Yes) | Not supported |


