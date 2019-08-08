业务中我们经常会使用到双向数据绑定，其中很重要的一个功能就是实现对数据的监听，虽然不同的MVVM框架可能会采取不同的方式去实现数据监听的操作。这篇文章主要是探讨Vue 2.0+版本采用的Object.defineProperty（注：Vue3.0版本将采用ES6的proxy）

首先看下Object.defineProperty是什么？
> Object.defineProperty() 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性， 并返回这个对象。
语法：[Object.defineProperty(obj, prop, descriptor)](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)

Object.defineProperty核心：
```javascript
let val;
Object.defineProperty(data, key, {
    get: () => { // 一个给属性提供 getter 的方法, 返回值被用作属性值
        return val;
    },
    set: (newVal) => { // 一个给属性提供 setter 的方法
        if (val !== newVal) {
            val = newVal;
        }
    }
});
```
当然，正常情况下，监听的data为一个对象，因此，在监听对象之前需要先判断传入的数据类型是否为Object
```javascript
const type = typeOf(data);
switch(type){
    case 'object':
        this.observe(data);
        break;
    default:
        console.error(`${data} must be an Object`);
        break;
}
```

下面是一个较为完整的代码：
```javascript
class Core {
    constructor (data, cb) {
        const type = typeOf(data);
        switch(type){
            case 'object':
                this.observe(data);
                break;
            default:
                console.error(`${data} must be an Object`);
                break;
        }
    }
    // 将data的属性转换为访问器属性
    observe (data) {
        // 考虑性能的话可以用for循环
        Object.keys(data).forEach((key, index) => {
            let val = data[key];
            Object.defineProperty(data, key, {
                enumerable: true,
                configurable: true,
                get: () => {
                    return val;
                },
                set: (newVal) => {
                    if (val !== newVal) {
                        if (typeOf(newVal) === 'object') {
                            this.observe(newVal);
                        }
                        val = newVal;
                    }
                }
            });

            // 递归调用
            if (typeOf(data[key]) === 'object') {
                this.observe(data[key]);
            }
        });
    }
};


export default Core;
```

vue对应的源码：[点击前往查看源码](https://github.com/vuejs/vue/blob/dev/src/core/observer/index.js)
相比较Vue可以发现，vue的源码比上面的代码多了observeArray的数组监听逻辑，原因是以上展示的代码并不支持监听Array.push()等数组数据操作，[从代码中可以看出](https://github.com/vuejs/vue/blob/dev/src/core/observer/array.js)Vue监听数组是通过保存数组的原始原型、重写数组方法、监听重写的数组方法等一系列操作来实现的。