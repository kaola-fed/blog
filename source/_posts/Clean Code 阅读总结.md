---
title: Clean Code 阅读总结
date: 2017-03-23
---

## 1 开始

本文是在阅读 `clean code` 时的一些总结，原书是基于 Java 的，这里将其中的一些个人认为实用性较强且容易与日常业务开发结合的一些原则重新进行整理，并参考了 [clean-code-javascript](https://github.com/ryanmcdermott/clean-code-javascript) 一文给出了一些代码实例，希望本文能够给日常开发编码和重构作出一些参考。

<!-- more -->

## 2 有意义的命名

### 2.1 名副其实

变量取名要花心思想想，不要贪图方便，过于简略的名称，时间长了以后就难以读懂。

```javascript
// bad
var d = 10;
var oVal = 20;
var nVal = 100;


// good
var days = 10;
var oldValue = 20;
var newValue = 100;
```

### 2.2 避免误导

命名不要让人对变量的信息 (类型，作用) 产生误解。

accounts 和 accountList，除非 accountList 真的是一个 List 类型，否则 accounts 会比 accountList 更好。因此像 List，Map 这样的后缀，不要随意使用。

```javascript
// bad
var platformList = {
    web: {},
    wap: {},
    app: {},
};


// good
var platforms = {
    web: {},
    wap: {},
    app: {},
};
```

### 2.3 做有意义的区分

用明确的意义去表述变量直接的区别。

很多情况下，会有存在 product，productData，productInfo 之类的命名，Data 和 Info 很多情况下并没有明显的区别，不如直接就使用 product。

```javascript
// bad
var goodsInfo = {
    skuDataList: [],
};

function getGoods(){};          // 获取商品列表
function getGoodsDetail(id){};  // 通过商品ID获取单个商品


// good
var goods = {
    skus: [],
};

function getGoodsList(){};      // 获取商品列表
function getGoodsById(id){};    // 通过商品ID获取单个商品

```

### 2.4 使用读得出来的名称

缩写要有个度，比如像 DAT 这样的写法，到底是 DATA 还是 DATE...

```javascript
// bad
var yyyyMMddStr = eu.format(new Date(), 'yyyy-MM-dd');
var dat = null;
var dev = 'Android';


// good
var todaysDate = eu.format(new Date(), 'yyyy-MM-dd');
var data = null;
var device = 'Android';
```

### 2.5 使用可搜索的名称

可搜索的名称能够帮助快速定位代码，尤其对于一些数字状态码，不建议直接使用数值，而是使用枚举。

```javascript
// bad
var param = {
    periodType: 0,
};


// good
const HOUR = 0, DAY = 1;
var param = {
    periodType: HOUR,
};
```

### 2.6 避免使用成员前缀

把类和函数做得足够小，消除对成员前缀的需要。因为长期以后，前缀在人们眼里会变得越来越不重要。

### 2.7 添加有意义的语境

对于某些名称，在不同语境下可能代表不同的含义，最好为它添加有意义的语境。

firstName，lastName，street，houseNumber，city，state，zipcode 一连串变量放在一起可以判断是一个地址，但是如果将这些变量单独拎出来，有些变量名意义就不明确了。这时可以添加语境明确其意义，如 addrFirstName，addrLastName，addrState。

当然也不要随意添加语境，这样只会让变量名变得冗长。

```javascript
// bad
var firsName, lastName, city, zipcode, state;
var sku = {
    skuName: 'sku0',
    skuStorage: 'storage0',
    skuCost: '10',
};


// good
var addrFirsName, addrLastName, city, zipcode, addrState;
var sku = {
    name: 'sku0',
    storage: 'storage0',
    cost: '10',
};
```

### 2.8 变量名从一而终

变量名取名多花一点时间，如果这一对象会在多个函数，模块中使用，就应该使用一致的变量名，否则每次看到这个对象，都需要重新去理清变量名，造成阅读障碍。

```javascript
// bad
function searchGoods(searchText) {
    getList({
        keyword: searchText,
    });
}
function getList(option) {

}

// good
function searchGoods(keyword) {
    getList({
        keyword: keyword,
    });
}

function getList(keyword) {}
```

## 3 函数

### 3.1 短小

短小是函数的第一规则，过长的函数不仅会造成阅读困难，在维护的时候难度也会增加。短小，要求每个函数做尽可能少的事情，同时减少代码的嵌套和缩进，要知道，代码的嵌套和缩减同样会带来阅读的困难。

```javascript
// bad
function initPage(initParams) {
    var data = this.data;
    if ('dimension' in initParams) {
        data.dimension = initParams.dimension;
        data.tab.source.some(function(item, index){
            if (item.value === data.dimension) {
                data.tab.defaultIndex = index;
            }
        });
    }
    if ('standardMedium' in initParams) {
        data.hasStandardMedium = true;
        data.filterParams[data.dimension].standardMedium = initParams.standardMedium;
    }
    if ('plan' in initParams || 'name' in initParams) {
        data.filterParams[data.dimension].planQueryString = initParams.plan || initParams.name;
    } else if ('traceId' in initParams) {
        data.filterParams[data.dimension].planQueryString = 'id:' + initParams.traceId;
    }
}

// good
function initPage(initParams) {
    initDimension(initParams);
    initStandardMedium(initParams);
    initPlanQueryString(initParams);
}
function initDimension(initParams) {
    var data = this.data;
    if ('dimension' in initParams) {
        data.dimension = initParams.dimension;
        data.tab.source.some(function(item, index){
            if (item.value === data.dimension) {
                data.tab.defaultIndex = index;
            }
        });
    }
}
function initStandardMedium(initParams) {
    var data = this.data;
    if ('standardMedium' in initParams) {
        data.hasStandardMedium = true;
        data.filterParams[data.dimension].standardMedium = initParams.standardMedium;
    }
}
function initPlanQueryString() {
    var data = this.data;
    if ('plan' in initParams || 'name' in initParams) {
        data.filterParams[data.dimension].planQueryString = initParams.plan || initParams.name;
    } else if ('traceId' in initParams) {
        data.filterParams[data.dimension].planQueryString = 'id:' + initParams.traceId;
    }
}

```

### 3.2 只做一件事情

函数应该做一件事情，做好这件事，只做这一件事。

如果函数只是做了该函数名下同一个抽象层上的步骤，则函数还是只做了一件事。当函数中出现另一抽象层级所做的事情时，则可以将这部分拆成另一层级的函数，因此缩小函数。

当一个函数可以被划分成多个区段时(代码块)时，这就说明了这个函数做了太多事情。

```javascript
// bad
function onTimepickerChange(type, e) {
    if(type === 'base') {
        // do base type logic...
    } else if (type === 'compare') {
        // do compare type logic...
    }
    // do other stuff...
}

// good
function onBaseTimepickerChange(e) {
    // do base type logic
    this.doOtherStuff();
}

function onCompareTimepickerChange(e) {
    // do compare type logic
    this.doOtherStuff();
}

function doOtherStuff(){}
```

### 3.3 每个函数一个抽象层级

一个函数中不应该混杂了多个抽象层级，即同一级别的步骤才放到一个函数中，因为通过这些步骤就能完整地完成一件事情。

回到之前提到变量命名的问题，一个变量或函数，其作用域余越广，就越需要一个有意义的名字来对其进行描述，提高可读性，减少在阅读代码时还需要去查询定义代码的频率，有些时候有意义的名字就可能需要更多的字符，但这是值得的。但对于小范围使用的变量和函数，可以适当缩短名称。因为过长的名称，某些时候反而会增加阅读的困难。

可以通过向下原则划分抽象层级

    程序就像是一系列 TO 起头的段落，每一段都描述当前层级，并引用位于下一抽象层级的后续 TO 起头段落
    - 如果要完成 A，需要完成 B，完成 C;
    - 要完成 B，需要完成 D;
    - 要完成 C，需要完成 E;

函数名明确了其作用，获取一个图表和列表，函数中各个模块的逻辑进行了划分，明确各个函数的分工, 拆分的函数名直接表明了每个步骤的作用, 不需要额外的注释和划分。在维护的时候, 可以快速的定位各个步骤, 而不需要在一个长篇幅的函数中需找对应的代码逻辑.

实际业务例子, 数据门户-流量看板-流量总览的一个获取趋势图和右边列表的例子。选择一个通过 tab 选择不同的指标，不同的指标影响的趋势图和右边列表的内容，两个模块的数据合并到一个请求中得到。流水账的写法可以将函数写成下面的样子，这种写法有几个明显的缺点:

- 长。通常情况下趋势图配置可能就需要20多行，整个函数加起来，轻易就超过50行了;
- 函数名不准确。函数名仅表明是获取一个图表的，但实际上还获取了右边列表数据并进行了配置;
- 函数层级混乱，还可以进行更细的划分;

根据向下原则

```javascript
// bad
getChart: function(){
    var data = this.data;
    var option = {
        url: '/chartUrl',
        param: {
            dimension: data.dimension,
            period: data.period,
            comparePeriod: data.comparePeriod,
            periodType: data.periodType,
        },
        fn: function(json){
            var data = this.data;
            // 设置图表
            data.chart = json.data.chart;
            data.chart.config = {
                //... 大量的图表配置，可能有20多行
            }
            // 设置右边列表
            data.sideList = json.data.list;
        }
    };
    // 获取请求参数
    this.fetchData(option);
},

// good
getChartAndSideList: function(){
    var option = {
        url: '/chartUrl',
        param: this.getChartAndSideListParam();
        fn: function(json){
            this.setChart(json);
            this.setSideList(json);
        }
    };
    this.fetchData(option);
},
```

### 3.4 switch语句

switch语句会让代码变得很长，因为switch语句天生就是要做多件事情，当状态不断增加的时候，switch语句也会不断增加。因此可能把取代switch语句，或者将其放在较低的层级.

放在底层的意思，可以理解为将其埋藏到抽象工厂地下，利用抽象工厂返回内涵不同的方法或对象来进行处理.

### 3.5 减少函数的参数

函数的参数越多，不仅注释写得长，使用的时候容易使得函数参数发生错位。当函数参数过多时，可以考虑以参数列表或者对象的形式传入.

数据门户里面的一个例子:

```javascript
// bad
function getSum(a [, b, c, d, e ...]){}


// good
function getSum(arr){}
```

```javascript
// bad
function exportExcel(url, param, onsuccess, onerror){}


// good
/**
 * @param option
 *    @property url
 *    @property param
 *    @property onsucces
 *    @property onerror
 */
function exportExcel(option){}
```

参数尽量少，最好不要超过 3 个

### 3.6 取个好名字

函数应该取个好一点的名字，适当使用动词和关键字可以提高函数的可读性。例如:

一个判断是否在某个区间范围的函数，取名为 `within`，从名称上可以容易判断出函数的作用，但是这仍然不是最好的，因为这个函数带有三个参数，无法一眼看出这个函数三个参数之间的关系，是 `b <= a && a<= c`，还是 `a <= b && b <= c` ?

或许可以通过更改参数名来表达三个参数的关系，这个必须看到函数的定义后才可能得知函数的用法.

如果再把名字改一下，从名字就可以容易得知三个参数依次的关系，当然这个名字可能会很长，但如果这个函数需要大范围地使用，较长的名字换来更好的可读性，这一代价是值得的.

```javascript
// bad
function within(a, b, c){}

// good
function assertWithin(val, min, max){}

// good
function assertValWithinMinAndMax(val, min, max){}
```

### 3.7 无副作用

一个有副作用的函数，通常都是是非纯函数，这意味着函数做的事情其实不止一件，函数所产生的副作用被隐藏了，函数调用者无法直接通过函数名来明确函数所做的事请.

## 4 注释

### 4.1 好注释

法律信息，提供信息的注释，对意图的解释，阐释，警示，TODO，放大(放大某种看似不合理代码的重要性)，公共 API 注释

尽量让函数，变量变得刻度，不要依赖注释来描述，对于复杂难懂的部分才适当用注释说明.

### 4.2 坏注释

喃喃自语，多余的注释(例如本来函数名就能够说明意图，还要加注释)，误导性注释，循规式注释(为了规范去加注释，其实函数名和参数名已经可以明确信息了)，日志式注释(记录无用修改日志的注释)，废话注释

### 4.3 原则

1. 能用函数或变量说明时，就别用注释，这就意味着要花点时间取个好名字

```javascript
// bad
var d = 10;     // 天数

// good
var days = 10;
```

2. 注释掉的代码不要留，重要的代码是不会被注释掉的

数据门户-实时概况里面的一段代码，`/src/javascript/realTimeOverview/components/index.js`

```javascript
// bad
function dimensionChanged(dimension){
    var data = this.data.keyDealComposition;
    data.selectedDimension = dimension;
    // 2016.10.31 modify：产品改动，选择品牌分布的时候不显示二级类目
    // if (dimension.dimensionId == '6') {
    //     data.columns[0][0].name = dimension.dimensionName;
    //     data.columns[0].splice(1, 0, {name:'二级类目', value:'secCategoryName', noSort: true});
    // } else {
        this.handle('util.setTableHeader');
    // }
    this.handle('refreshComposition');
};

// good
function dimensionChanged(dimension){
    var data = this.data.keyDealComposition;
    data.selectedDimension = dimension;
    this.handle('util.setTableHeader');
    this.handle('refreshComposition');
};
```

3. 不要在注释里面加入太多信息，没人会看

4. 非公用函数，没有必要加过多的注释说明，冗余的注释会使代码变得不够紧凑，增加阅读障碍

```javascript
// bad
/**
 * 设置表格表头
 */
function setTableHeader(){},

// good
function setTableHeader(){},
```

5. 括号后的注释

```javascript
// bad
function doSomthing(){
    while(!buffer.isEmpty()) {  // while 1
        // ...
        while(arr.length > 0) {  // while 2
            // ...
            if() {

            }
        } // while 2
    } // while 1
}
```

6. 不需要日志式，归属式注释，相信版本控制系统

```javascript
// bad
/**
 * 2016.12.03 bugfix, by xxxx
 * 2016.11.01 new feature, by xxxx
 * 2016.09.12 new feature, by xxxx
 * ...
 */


// bad
/**
 * created by xxxx
 * modified by xxxx
 */
function addSum() {}

/**
 * created by xxxx
 */
function getAverage() {
    // modified by xxx
}
```

7. 尽量别用用位置标记

```javascript
// bad

/*************** Filters ****************/

///////////// Initiation /////////////////
```

## 5 格式

### 5.1 垂直方向

1. 相关代码紧凑显示，不同部分的用空格隔开

```javascript
// bad
function init(){
    this.data.chartView = this.$refs.chartView;
    this.$parent.$on('inject', function () {
        this.dataConvert(this.data.source);
        this.draw();
    });
    this.$watch('source', function (newValue, oldValue) {
        if (newValue && newValue != this.data.initValue) {
            this.dataConvert(newValue);
            this.draw();
        } else if (!newValue) {
            if (self.data.chartView) {
                this.data.chartView.innerHTML = '';
            }
        }
    }, true);
}

// good
function init(){
    this.data.chartView = this.$refs.chartView;

    this.$parent.$on('inject', function () {
        this.dataConvert(this.data.source);
        this.draw();
    });

    this.$watch('source', function (newValue, oldValue) {
        if (newValue && newValue != this.data.initValue) {
            this.dataConvert(newValue);
            this.draw();
        } else if (!newValue) {
            if (this.data.chartView) {
                this.data.chartView.innerHTML = '';
            }
        }
    }, true);
}
```

2. 不要在代码中加入太多过长的注释，阻碍代码阅读

```javascript
// bad
BaseComponent.extend({
    checkAll: function(status){
        status = !!status;
        var data = this.data;
        this.checkAllList(status);
        this.checkSigList(status);
        data.checked.list = [];
        if(status){
            // 当全选的时候先清空列表, 然后在利用Array.push添加选中项
            // 如果在全选的时候不能直接checked.list = dataList
            // 因为这样的话后面对checked.list的操作就相当于对dataList直接进行操作
            // 利用push可以解决这一个问题
            data.sigList.forEach(function(item,i){
                data.checked.list.push(item.data.item);
            })
        }
        this.$emit('check', {
            sender: this,
            index: CHECK_ALL,
            checked: status,
        });
    },
});

// good
BaseComponent.extend({
    checkAll: function(status){
        status = !!status;
        this.checkAllList(status);
        this.checkSigList(status);
        this.clearCheckedList();
        if(status){
            this.updateCheckedList();
        }

        this.emitCheckEvent(CHECK_ALL, status);
    },
});
```

3. 函数按照依赖顺序布局，被调用函数应该紧跟调用函数

```javascript
// bad
function updateModule() {}
function updateFilter() {}
function reset() {}
function refresh() {
    updateFilter();
    updateModule();
}

// good
function refresh() {
    updateFilter();
    updateModule();
}
function updateFilter() {}
function updateModule() {}
function reset() {}
```

4. 相关的，相似的函数放在一起

```javascript
// bad
function onSubmit() {}
function refresh() {}
function onFilterChange() {}
function reset() {}

// good
function onSubmit() {}
function onFilterChange() {}

function refresh() {}
function reset() {}
```

5. 变量声明靠近其使用位置

```javascript
// bad
function (x){
    var a = 10, b = 100;
    var c, d;

    a = (a-b) * x;
    b = (a-b) / x;
    c = a + b;
    d = c - x;
}

// good
function (x){
    var a = 10, b = 100;

    a = (a-b) * x;
    b = (a-b) / x;

    var c = a + b;
    var d = c - x;
}
```

#### 5.2 水平方向

1. 运算符号之间空格，但是要注意运算优先级

```javascript
// bad
var v = a + (b + c) / d + e * f;

// good
var v = a + (b+c)/d + e*f;
```

2. 变量水平对齐意义不大，应该让其靠近

```javascript
// bad
var a       = 1;
var sku     = goodsInfo.sku;
var goodsId = goodsInfo.goodsId;

// good
var a = 1;
var sku = goodsInfo.sku;
var goodsId = goodsInfo.goodsId;
```

### 5.4 对于短小的if，while语句，也要尽量保持缩进

突然间改变缩进的规律，很容易就会被阅读习惯欺骗

```javascript
// bad
if(empty){return;}


// good
if(empty){
    return;
}

// bad
while(cli.readCommand() != -1);
app.run();


// good
while(cli.readCommand() != -1)
;

app.run();
```

## 6 实际业务代码中的应用

### 庞大的config函数

对于一些较为复杂的组件或页面组件，需要定义很多属性，同时又要对这部分属性进行初始化和监听，像下面这段代码。在好几个大型的页面里面都看到了类似的代码，config 方法少的有 100行，多的有 400行。

 config 方法基本就是一个组件的入口，在进行维护的时候一般都会先读 config 方法，但是对于这么长的函数，很容易第一眼就懵了。

```javascript
Component.extend({
    template: tpl,
    config: function(data){
        eu.extend(data, {
            tabChartTab: 0,
            periodType: 0,
            dimensionType: 1,
            dealConstituteCompare:false,
            dealConstituteSort: {
                dimensionValue: 'sales',
                sortType: 0,
            },
            dealConstituteDecorate: {
                noCompare:[],
                progress: ['salesPercent'],
                sort:[
                ]
            },
            defaultMetrics: [
            ],
            // ...下面还有几百行关于其他模块的属性, flow, hotSellRank等
        });

        this.$watch('periodType', function(){
            // ...
        });

        this.$watch('topCategoryId', function(){
            // ...
        });

        // 这里还有一部分异步请求代码...
        this.refresh();
    },
})
```

针对上述这段代码代码，明显的缺点是:

- 太长
- 变量命名有冗余信息，且搜索性差
- 变量(属性)太多
- 做的事情太多，初始化组件属性，添加监听方法，还有一些业务逻辑代码

这对这些可以作出一些改进:

- 使用枚举代替数值
- config内只保留一切作为范围加大属性的直接初始化代码，其余针对于模块的属性将通过调用 `initData` 方法来初始化
- `initData` 进一步根据模块划分初始化方法
- 对于属于摸个模块的属性，则将其划分到同一个对象上，减少组件上挂载的属性数量，同时也简化了属性的命名
- 监听方法同样是通过 `addWatchers` 初始化
- 初始化过程中需要执行的部分逻辑，尽可能放在 `init` 等组件实例化后执行

```javascript
const TAB_A = 0, TAB_B = 1;
const HOUR = 0, DAY = 1;
const DIMENSION_A = 0, DIMENSION_B = 1;
const DISABLE = false, ENABLE = true;

Component.extend({
    template: tpl,
    config: function(data){
        eu.extend(data, {
            tabChartTab: TAB_A,
            periodType: HOUR,
            dimensionType: DIMENSION_B,
        });

        this.initData();
        this.addWatchers();
    },

    initData: function(){
        this.initDealConsitiuteData();
        this.initFlowData();
        this.initHotSellRank();
    },

    initDealConsitiuteData: function(){
        this.data.dealConstitute = {
            compare: DISABLE,
            sort: {
                dimensionValue: 'sales',
                sortType: 0,
            },
            decorate: {
                noCompare:[],
                progress: ['salesPercent'],
                sort:[
                ]
            },
            defaultMetrics: [
            ],
        }
    },

    addWatchers: function(){
        this.$watch('periodType', function(){
            // ...
        });

        this.$watch('topCategoryId', function(){
            // ...
        });
    },

    init: function(){
        // 部分初始化要执行的逻辑
        this.refresh();
    },

})
```

其实按照上面进行优化以后，代码的可读性是有所提高，但由于这是一个页面组件，代码行数极多，修改后方法变得更多了，仍然不便于阅读。所以，针对于这种大型的页面，更适当的做法是，将页面拆分为几个模块，将业务逻辑拆分，减少每个模块的代码量，提高可读性。而对于不可再拆分的组件或模块，如果仍然包含大量需要初始化的属性，上述例子就可以作为参考了。


## 7 总结

本文整理的几个要点:

- 写代码就像写故事，里面各个角色 (变量，函数) 的名字要取得好，才读得流畅;
- 函数要短小，不要混杂太多不相关，不同层级的逻辑;
- 注释要精简准确，能不写就不要写;
- 代码布局要向报纸学习，排版注意垂直与水平方向的间隔，联系紧密的布局要紧凑;

就算是经验老道的大神，也很难一遍就能写出简洁的代码，所以要勤于对代码进行重构，边写代码边修改。代码只有在经过一遍一遍修改和锤炼以后，才会逐渐地变得简洁和精致。

## 8 参考

1. [Clean Code](https://book.douban.com/subject/3032825/)
2. [clean-code-javascript](https://github.com/ryanmcdermott/clean-code-javascript)
