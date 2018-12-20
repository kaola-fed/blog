众所周知，Javascript是一门基于原型编程的动态脚本语言，虽然它也支持面向对象的编程范式，但想要做到像传统编程语言那样使用OOP，Javascript依然存在这样那样的不足。

从ES6开始，JS开始支持使用Class来编写类，但是该语法依然不够完善，一个显而易见的缺点就是我们无法使用private关键字定义变量。在其它面向对象的语言中，private修饰的成员或方法只能够在其定义的类中的方法中引用，无法通过该类及派生类的对象访问。

为了实现Javascript中的private，防止数据在运行时被改变，本文总结了五类方法来达成这一目的，接下来将一一介绍

![](https://haitao.nos.netease.com/238ee644-be1b-4932-b1cb-88a9d85e7e15.png)

### 一. 使用命名规则 (_xx)

在一个工程中，为了区分代码中的私有成员和公有成员，当我们表示一个成员是私有时，就以下划线开头的命名规则来命名。

```js
class Company {
  constructor(name) {
    //public
    this.name = name;
    //private
    this._asset = 'computer';
  }
  
  getAsset() {
    return this._asset;
  }
}
```
上面的Company类中，_asset表示私有成员，只能通过调用getAsset方法来获取。然而我们并不能阻止外部直接访问这个变量并修改它。
```js
const company = new Company('netease');

console.log(company.getAsset());   // computer
console.log(company._asset)    // computer
company._asset = 'desk';
console.log(company.getAsset());   //desk
```

可以看到，使用命名规则并不能真正的使变量私有化，更多的是作为一种编码规范，约定了代码中的哪些成员是私有的，使得在后期的代码维护中可以一目了然。在后面介绍的方法中，我们也将使用这种命名规则来表示私有成员。

### 二. 使用闭包 (closure)

> 闭包是函数和声明该函数的词法环境的组合

下面的例子展示了如何使用闭包封装私有成员变量。

```js
function Employee() {
  const this$ = {};
  
  class Employee {
    constructor(name) {
      this$._name = name;
    }
    
    get name() {
      return this$._name
    }
  }
  
  return new Employee(...arguments)
};

const employee = new Employee('anonymous');

console.log(employee.name)    // 'anonymous'
console.log(employee._name)   // undefined
```

可以看到，外部环境不能直接访问闭包内的变量‘this$’对象内的属性，从而实现Employee对象成员的私有化。然而闭包的缺点也很明显，便是其所封装的变量常驻内存，如果管理不善，有可能会造成内存泄漏。那么除了使用闭包，是否还有其它方案可以实现对象成员的私有化呢？答案是yes，接下来的章节我们将考察如何使用ES6新增的特性实现Private成员变量。

### 三. 使用WeakMap

WeakMap是ES6中新增的数据类型，它有一个很大的特点，就是无法遍历，由于WeakMap的key存储的对象是弱引用，所以它的key是无法枚举的。（如果key是可枚举的话，其列表将会受垃圾回收机制的影响，从而得到不确定的结构）利用WeakMap这一特性，我们可以实现对象的私有成员。

```js
const map = new WeakMap();

// 用来保存每个对象实例的私有成员
const internal = obj => {
  if (!map.has(obj)) {
    map.set(obj, {});
  }
  
  return map.get(obj);
}

class Employee {
  constructor(name) {
    internal(this)._name = name;
  }
  
  get name() {
    return internal(this)._name;
  }
}

const employee = new Employee('anonymous');

console.log(employee.name)    // 'anonymous'
console.log(employee._name)   // undefined
```

虽然我们无法直接通过Employee实例访问到私有属性_name，但是由于存储私有成员的WeakMap对象暴露在外部，我们依然可以通过修改WeakMap来达到访问并修改employee私有成员的目的。

```js
map.get(employee)._name = 'boss';

console.log(employee.name)   // 'boss'
```

所以为了更进一步的优化，WeakMap往往会结合闭包来实现私有成员。

```js
function Employee() {
  const private = new WeakMap();
  
  const internal = obj => {
    if (!private.has(obj)) {
      private.set(obj, {});
    }
    
    return private.get(obj);
  };
  
  class Employee {
    constructor(name) {
      internal(this)._name = name;
    }
    
    get name() {
      return internal(this)._name;
    }
  }
  
  return new Employee(...arguments)
}

const employee = new Employee('anonymous');

console.log(employee.name)    // 'anonymous'
console.log(employee._name)   // undefined
```
因为WeakMap的每个键对自己所引用对象的引用都是 "弱引用"，所以当没有其他引用和该键引用同一个对象时,这个对象将会被当作垃圾回收。这样，闭包所引起的潜在内存泄漏问题就得到来解决。

### 四. 使用Symbol

同WeakMap一样，Symbol也是ES6新增的基本数据类型。Symbol作为对象属性不可见，意味着无法通过遍历或者JSON.stringify访问到，基于这种特性，我们也可以创建对象的私有属性。

```js
const _name = Symbol('name');

class Employee{
  constructor(name) {
    this[_name] = name;
  }
  
  get name() {
    return this[_name];
  }
}

const employee = new Employee('anonymous');

console.log(employee.name)    // 'anonymous'
console.log(employee._name)   // undefined
```

使用Symbol来隐藏数据其实也是一种不错的方案，然而同WeakMap一样，它依然有被访问到并修改的风险。通过调用Object.getOwnPropertySymbols()方法，我们可以访问到这个对象的所有Symbol属性，并对其做出修改。

```js
for (var symbol of Object.getOwnPropertySymbols(employee)) {
  console.log(employee[symbol])  // 'anonymous'
}
```

### 五. 使用Proxy

Proxy也是ES6新增对象，它的作用就是在目标对象之前架设一层“拦截”。当外界访问该对象，都必须先通过这层拦截，因此提供了一种机制，可以对外界的访问进行过滤和改写。

利用这一特性，我们可以按照命名规则，对象的私有属性命名以下划线开头，通过代理属性的get和set方法，拦截外部对对象私有属性的访问，从而实现对象的封装。

首先我们定义一个包含陷阱的占位符对象handler，其中的陷阱便是get和set，通过这两个陷阱来控制属性的读写。

```js
 const handler = {
   get: function(target, key) {
    if (key[0] === '_') {
      throw new Error('Attempt to access private Property');
    }
    
    return target[key]
   },
   set: function(target, key, value) {
    if (key[0] === '_') {
      throw new Error('Attempt to accsess private Property');
    }
    
    target[key] = value;
   }
 }
```

上面的两个陷阱代理了get和set，每当要访问对象成员时，都会检测是否私有。

```js
  class Employee{
    constructor(name) {
      this._name = name;
    }
    
    get name() {
      return this._name;
    }
  }
  
  const employee = new Proxy(new Employee('anonymous'), handler);
  
  console.log(employee.name);    // 'anonymous'
  console.log(employee._name);   // Error: Attempt to access private Property
  
  employee._name = 'boss';  // Error: Attempt to access private Property
  console.log(square.area) // 100
  console.log(square instanceof Shape)
```

当尝试用JSON.stringify序列化employee对象时，因为会访问到私有属性，所以会报错。

```js
JSON.stringify(employee)   
//Error: Attempt to access private Property
//at Object.get (fawicagejo.js:4:13)
//at JSON.stringify (<anonymous>)
```

参考[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify)
> 如果一个被序列化的对象拥有 toJSON 方法，那么该 toJSON 方法就会覆盖该对象默认的序列化行为：不是那个对象被序列化，而是调用 toJSON 方法后的返回值会被序列化

所以需要改写get陷阱。

```js
get: function(target, key) {
  if (key[0] === '_') {
    throw new Error('Attempt to access private property');
  } else if (key === 'toJSON') {
    const obj = {};
    
    for (const key in target) {
      if (key[0] !== '_') {
        obj[key] = target[key];
      }
    }
    
    return () => obj;
  }
  
  return target[key];
}
```

改写完后，再通过JSON.stringify()便能够正常序列化对象了。

不过以上方法依然存在一个问题，就是使用迭代，我们依然可以访问到对象的私有属性。

```js

```
但是我们依然可以通过迭代的形式依然能遍历到私有属性名，比如：
```js
for (let key in employee) {
  console.log(key)
}
// "_name"
```

这个时候，我们需要使用getOwnPropertyDescriptor陷阱，将私有属性的属性描述符中的enumerable设置为false。

```js
getOwnPropertyDescriptor(target, key) {
  const desc = Object.getOwnPropertyDescriptor(target, key);
  
  if (key[0] === '_') {
    desc.enumerable = false;
  }
  
  return desc;
}
```

完整代码如下

```js
 const handler = {
   get: function(target, key) {
    if (key[0] === '_') {
      throw new Error('Attempt to access private property');
    } else if (key === 'toJSON') {
      const obj = {};
      
      for (const key in target) {
        if (key[0] !== '_') {
          obj[key] = target[key];
        }
      }
      
      return () => obj;
    }
    
    return target[key];
  },
   set: function(target, key, value) {
    if (key[0] === '_') {
      throw new Error('Attempt to accsess private Property');
    }
    
    target[key] = value;
   },
   getOwnPropertyDescriptor(target, key) {
    const desc = Object.getOwnPropertyDescriptor(target, key);
    
    if (key[0] === '_') {
      desc.enumerable = false;
    }
    
    return desc;
  }
 }
 
 const employee = new Proxy(new Employee('anonymous'), handler);
```

## 新的(#)符号
以上的努力都是在Javascript 新的private特性出来所做的尝试，难免繁琐。

ES2018最新的Privatet特性已经进入了[https://github.com/tc39/proposals](https://github.com/tc39/proposals) stage3, 我们可以一睹芳容

```js
class Employee {
  #name;
  
  constructor(name) {
    this.#name = name;
  }
  
  get name() {
    return this.#name;
  }
}

const employee = new Employee('anonymous');

console.log(employee.name)    // 'anonymous'
console.log(employee.#name)   // undefined
```

使用#来描述私有成员，使得Javascript也能像其他高级语言一样实现封装了。

## 结语

随着新的特性不断涌现，可以看到Javascript是越趋完善，开发者能够体验越来越丰富的功能以提升开发效率。但是我们依然要感谢前人为了实现这些尚未出现的特性所做出的努力，吃水不忘挖井人。

以上

by zhangxueai@corp.netese.com