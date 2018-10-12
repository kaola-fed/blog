<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
在[Vue官方文档](https://cn.vuejs.org/v2/guide/list.html#%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)里，有这样一段内容：

由于JavaScript的限制，Vue不能检测以下变动的数组：
1. 当你利用索引直接设置一个项时，例如：vm.items[indexOfItem] = newValue
2. 当你修改数组的长度时，例如：vm.items.length = newLength

为了解决第一类问题，以下两种方式都可以实现和 vm.items[indexOfItem] = newValue 相同的效果，同时也能触发状态更新：
```
// Vue.set
Vue.set(example1.items, indexOfItem, newValue)
```
```
// Array.prototype.splice
example1.items.splice(indexOfItem, 1, newValue)
```

为了解决第二类问题，可以使用 splice：
```
example1.items.splice(newLength, 0, undefined)
```

其中Array.splice( )方法用于从一个数组中移除元素，如有必要，在所移除元素的位置上插入新元素，并返回所移除的元素，而原数组被改变。
其语法为：
```
arrayObject.splice(index, howmany, [item1[,item2[,…[,itemN]]]])
```

所以上面的第二个方法（Array.prototype.splice）在example1里添加了新值newValue，并改变了example1本身。【Vue重新封装了包括Array.prototype.splice在内的许多数组方法，这些方法会把数据属性转换为访问器属性】

对于对象，还可以用Object.assign( )浅拷贝一个对象来解决（替换掉vm.data的原对象）：
```
vm.user = Object.assign({}, vm.user, {age: 18, name: ’zoro'})
```

那么为什么直接对数组（或对象）利用索引进行设值、直接更改length时不会触发状态更新呢？Vue官方文档只说“由于JavaScript的限制”，那么这个限制到底是什么呢？

Vue监听数据变动，是利用了JavaScript的Object.definedProperty方法，依靠descriptor里的getter和setter这两个[访问器属性](https://github.com/FridaS/blog/issues/7)。（Vue不支持IE8以下浏览器就是因为IE8以下浏览器不支持Object.definedProperty）

new Vue(obj) 时（初始化实例时），其内部发生了大体如下的代码转换（将数据属性，转换为了访问器属性）：
```
function Vue (obj) {
    // 遍历对象所有的属性，并使用Object.defineProperty
    // 把这些属性全部转为getter/setter
    obj.data.keys().forEach((item, index) => {
        Object.defineProperty(obj.data, item, {
            set () {
                // 可以在此处进行事件监听
            },
            get () {
                //
            }
        })
    })
    return obj;
}
```

所以当使用索引给数组或对象添加属性（如 vm.items[0] = {}）时，是直接 = 赋值，是数据属性的（而不是访问器属性），是默认的[[Get]]操作和[[Put]]操作，所以无法检测变动。
至于push等操作可以检测到变动，是因为Vue把这些Api重新封装了。

另外，Vue不允许在已经创建的实例上动态添加新的根级响应式属性，所以必须在初始化实例前声明根级响应式属性，哪怕只是一个空值：
```
var vm = new Vue({
    data: {
        // 声明 message 为一个空值字符串
        message: ''
    },
    template: ‘<div>{{ message }}</div'
})
// 之后设置 `message`
vm.message = ‘Hello!'
```

这样的限制，除了消除了在依赖项跟踪系统中的一类边界情况，也使Vue实例在类型检查系统的帮助下运行得更高效。而且在代码可维护方面也有一点重要的考虑：data 对象就想组件状态的概要，提前声明所有的响应式属性，可以让代码在以后重新阅读或其他开发人员阅读时更易于被理解。


#### 总结：
    Vue不允许在已经创建的实例上动态添加新的根级响应式属性，即所有根级属性必须初始化前就已经在data里声明了，但是对data里已经声明的属性（该属性为数组或对象）的响应式子属性可以使用特定的方法动态添加：
* 对于数组和对象，可以使用Vue.set(object, key, value)
* 对于对象，还可以使用Object.assign()来构造一个新属性、替换掉原属性
* 对于数组，还可以使用Array.prototype.splice方法，如：example1.items.splice(indexOfItem, 1, newValue)


#### 参考：
1. https://cn.vuejs.org/v2/guide/reactivity.html
2. 《你不知道的JavaScript（上）》
3. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/splice
4. https://msdn.microsoft.com/zh-cn/library/wctc5k7s(v=vs.94).aspx
5. http://www.w3school.com.cn/js/jsref_splice.asp
6. https://segmentfault.com/q/1010000006938552
7. https://segmentfault.com/q/1010000008332647

by Frida