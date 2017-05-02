---
title: JavaScript设计模式
date: 2017-03-22
---

> 设计模式的定义：在面向对象软件设计过程中针对特定问题的简洁而优雅的解决方案

<!-- more -->

当然我们可以用一个通俗的说法：设计模式是解决某个特定场景下对某种问题的解决方案。因此，当我们遇到合适的场景时，我们可能会条件反射一样自然而然想到符合这种场景的设计模式。

比如，当系统中某个接口的结构已经无法满足我们现在的业务需求，但又不能改动这个接口，因为可能原来的系统很多功能都依赖于这个接口，改动接口会牵扯到太多文件。因此应对这种场景，我们可以很快地想到可以用适配器模式来解决这个问题。

_下面介绍几种在JavaScript中常见的几种设计模式：_

## 1.单例模式

> 单例模式的定义：保证一个类仅有一个实例，并提供一个访问它的全局访问点。实现的方法为先判断实例存在与否，如果存在则直接返回，如果不存在就创建了再返回，这就确保了一个类只有一个实例对象。

适用场景：一个单一对象。比如：弹窗，无论点击多少次，弹窗只应该被创建一次。

```javascript
class CreateUser {
    constructor(name) {
        this.name = name;
        this.getName();
    }
    getName() {
         return this.name;
    }
}
// 代理实现单例模式
var ProxyMode = (function() {
    var instance = null;
    return function(name) {
        if(!instance) {
            instance = new CreateUser(name);
        }
        return instance;
    }
})();
// 测试单体模式的实例
var a = new ProxyMode("aaa");
var b = new ProxyMode("bbb");
// 因为单体模式是只实例化一次，所以下面的实例是相等的
console.log(a === b);    //true
```

## 2.策略模式

> 策略模式的定义：定义一系列的算法，把他们一个个封装起来，并且使他们可以相互替换。

策略模式的目的就是将算法的使用算法的实现分离开来。

一个基于策略模式的程序至少由两部分组成。第一个部分是一组策略类（可变），策略类封装了具体的算法，并负责具体的计算过程。第二个部分是环境类Context（不变），Context接受客户的请求，随后将请求委托给某一个策略类。要做到这一点，说明Context中要维持对某个策略对象的引用。

```javascript
/*策略类*/
var levelOBJ = {
    "A": function(money) {
        return money * 4;
    },
    "B" : function(money) {
        return money * 3;
    },
    "C" : function(money) {
        return money * 2;
    } 
};
/*环境类*/
var calculateBouns =function(level,money) {
    return levelOBJ[level](money);
};
console.log(calculateBouns('A',10000)); // 40000
```

## 3.代理模式

> 代理模式的定义：为一个对象提供一个代用品或占位符，以便控制对它的访问。

常用的虚拟代理形式：某一个花销很大的操作，可以通过虚拟代理的方式延迟到这种需要它的时候才去创建（例：使用虚拟代理实现图片懒加载）

图片懒加载的方式：先通过一张loading图占位，然后通过异步的方式加载图片，等图片加载好了再把完成的图片加载到img标签里面。

```javascript
var imgFunc = (function() {
    var imgNode = document.createElement('img');
    document.body.appendChild(imgNode);
    return {
        setSrc: function(src) {
            imgNode.src = src;
        }
    }
})();
var proxyImage = (function() {
    var img = new Image();
    img.onload = function() {
        imgFunc.setSrc(this.src);
    }
    return {
        setSrc: function(src) {
            imgFunc.setSrc('./loading,gif');
            img.src = src;
        }
    }
})();
proxyImage.setSrc('./pic.png');
```

使用代理模式实现图片懒加载的优点还有符合单一职责原则。减少一个类或方法的粒度和耦合度。

## 4.中介者模式

> 中介者模式的定义：通过一个中介者对象，其他所有的相关对象都通过该中介者对象来通信，而不是相互引用，当其中的一个对象发生改变时，只需要通知中介者对象即可。通过中介者模式可以解除对象与对象之间的紧耦合关系。

例如：现实生活中，航线上的飞机只需要和机场的塔台通信就能确定航线和飞行状态，而不需要和所有飞机通信。同时塔台作为中介者，知道每架飞机的飞行状态，所以可以安排所有飞机的起降和航线安排。

中介者模式适用的场景：例如购物车需求，存在商品选择表单、颜色选择表单、购买数量表单等等，都会触发change事件，那么可以通过中介者来转发处理这些事件，实现各个事件间的解耦，仅仅维护中介者对象即可。

```javascript
var goods = {   //手机库存
    'red|32G': 3,
    'red|64G': 1,
    'blue|32G': 7,
    'blue|32G': 6,
};
//中介者
var mediator = (function() {
    var colorSelect = document.getElementById('colorSelect');
    var memorySelect = document.getElementById('memorySelect');
    var numSelect = document.getElementById('numSelect');
    return {
        changed: function(obj) {
            switch(obj){
                case colorSelect:
                    //TODO
                    break;
                case memorySelect:
                    //TODO
                    break;
                case numSelect:
                    //TODO
                    break;
            }
        }
    }
})();
colorSelect.onchange = function() {
    mediator.changed(this);
};
memorySelect.onchange = function() {
    mediator.changed(this);
};
numSelect.onchange = function() {
    mediator.changed(this);
};
```

## 5.装饰者模式

> 装饰者模式的定义：在不改变对象自身的基础上，在程序运行期间给对象动态地添加方法。

例如：现有4种型号的自行车分别被定义成一个单独的类，如果给每辆自行车都加上前灯、尾灯、铃铛这3个配件，如果用类继承的方式，需要创建4*3=12个子类。但如果通过装饰者模式，只需要创建3个类。

装饰者模式适用的场景：原有方法维持不变，在原有方法上再挂载其他方法来满足现有需求；函数的解耦，将函数拆分成多个可复用的函数，再将拆分出来的函数挂载到某个函数上，实现相同的效果但增强了复用性。

例：用AOP装饰函数实现装饰者模式

```javascript
Function.prototype.before = function(beforefn) {
    var self = this;    //保存原函数引用
    return function(){  //返回包含了原函数和新函数的 '代理函数'
        beforefn.apply(this, arguments);    //执行新函数，修正this
        return self.apply(this,arguments);  //执行原函数
    }
}
Function.prototype.after = function(afterfn) {
    var self = this;
    return function(){
        var ret = self.apply(this,arguments);
        afterfn.apply(this, arguments);
        return ret;
    }
}
var func = function() {
    console.log('2');
}
//func1和func3为挂载函数
var func1 = function() {
    console.log('1');
}
var func3 = function() {
    console.log('3');
}
func = func.before(func1).after(func3);
func();
```
