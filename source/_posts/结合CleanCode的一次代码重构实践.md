---
title: 结合CleanCode的一次代码重构实践
date: 2017-04-19
---

类目参谋近期进行版本迭代，添加新功能。原本代码开发已久，可优化空间较大也为了便于日后的维护和拓展，决定页面重构。我参与了其中交易构成页面的代码重构。（数据门户 - 类目参谋 - 交易分析 - 交易构成）

<!-- more -->

**目标：函数拆解，组件降耦，健壮逻辑，bugfix**

## 步骤
1. 线上请求分析，online-bug记录
2. 阅读原代码，功能、模块拆分
3. 按模块重构代码，连线上检验重构结果
4. clean code
5. 静态代码检测

## 重构
### 1. 线上请求分析
####  多次请求、频繁请求

1. 切换维度tab，连续2次获取数据列表 /dealConstituteDetail

    1. 改变一级维度重新获取列表
    2. 筛选器multiFilter监听dimension -> 改变 -> 重置筛选器 -> 触发请求。
    
    第一个触发是合理的。第二个触发应该有前提，如果原先没有使用筛选器（下图为筛选器），则不需要触发请求
    ![](http://i1.piimg.com/567571/c562bf59802ff7d7.png)
2. 每一次切换维度tab，都会获取二级维度 /secondDimensions

    请求返回二级维度列表 = '未选择' + 一级维度列表 - 当前tab维度
        
    所有的请求都是这样的返回，因此根据当前一级维度动态更新二级维度列表
3. 选择日期，发出2-3次 /dealConstituteDetail 和1次 /secondDimensions
        
    1. /dealConstituteDetail 一级维度重置，触发请求
    2. /secondDimensions 获取二级维度
    3. /dealConstituteDetail 对比时间重置，触发请求
    
    第三次请求不合理：修改基础时间不应该使对比时间选择器触发请求。两个组件耦合在一起。

#### 报错、页面异常
![](http://i1.piimg.com/567571/c562bf59802ff7d7.png)
4. [选择指标] 选择少于6个指标，更换时间类型，页面报错
5. [选择指标] 选择除'uv'、'平均转化率'之外的6个以上的指标，第4和第6个表头字段被换成'UV/日均UV'和'转化率/平均转化率'，跟底下的数据对不上，显然错误
6. 只有[选择指标]中的前6个有排序
7. 页面宽度不够时，对比时间样式异常

   ![](http://i2.buimg.com/567571/13221b404557978e.png)

### 2. CleanCode
1. 'period'设置默认值交给函数
2. 'periodType'采用枚举
3. 'periodMap'改为数组
4. 'comparePeriod'使用key-val原本应该是考虑到本页中有其他模块的情况，建议模块自理，消除耦合
5. 'tradeConstituteSort.sortType' 0指代不明，使用枚举，使赋值语义化```var DESC = 0; sortType = DESC;```
```
// befor
eu.extend(data, {
    "period": data.period||u._$format(new Date(+now - 24 * 3600 * 1000), 'yyyy-MM-dd') + '~' + u._$format(new Date(+now - 24 * 3600 * 1000), 'yyyy-MM-dd'), // 默认近1天
    "periodType": data.periodType||0, // 时间段类型，0 'day' 自然日，1 '7day' 近7天，2 '30day' 近30天，3 'month' 自然月
    "periodMap": {
        "0": 'day', // 自然日
        "1": 'recent7', // 近7天
        "2": 'recent30', // 近30天
        "3": 'month' // 自然月
    },
    "comparePeriod": {
        "tradeConstitute": "" //交易构成
    },
    "tradeConstituteSort": {    //默认排序字段
        "dimensionValue": "sales",
        "sortType": 0
    },
    ...
});
```
```
// after
eu.extend(data, {
    period: ut.getYesterday(),
    periodMap: ['day', 'recent7', 'recent30', 'month'],
    periodType: +PERIOD_TYPE.DAY,
    comparePeriod: '',
    sortInfo: {
        dimensionValue: 'sales',
        sortType: DESC,
    },
    ...
});
```
6. 函数应根据调用顺序排放（不在主流程中且不与其他函数有过多交集的函数可置后，如导出文件功能）
7. 注释掉的代码，原代码中有多处
8. 多余的注释
    ```
    //如果没有搜索信息 则清空搜索栏
    if (!searchText) {
        data.searchText = "";
    } else if(searchText !== true) {
        data.searchText = searchText;
    }
    ```
9. 代码过长

    getTradeConstitute获取交易构成明细，函数40几行。做了几件事情，应拆解：
    1. 处理传参
    2. 获取类目名称
    3. 请求参数声明
    4. 设置回调函数

10. 重复的功能相似的代码

    获取交易构成明细 与 导出文件 的参数基本一致，声明了两次。

11. 不应在公共组件堆叠当前模块特有的组件

dealComposition <- multifilter <- multiSelector。将multi-两个组件放到dealComposition模块

12. 参数意义尽量直接
```
getTradeDataBySort: function(sort, notRequest) {
    ...
    if (!notRequest) {  // !不请求 = 请求
        this.getTradeConstitute(undefined, undefined, sort, true);
    }
},
```
换成doRequest更直接

13. 函数参数过多

getTradeConstitute这个方法带四个参数，html和js中调用常有undefined实参传入，阅读的时候会很困惑。

```
getTradeConstitute: function (currentPage, filters, sort, searchText) {...}

// html中使用该函数
on-filterSearch={this.getTradeConstitute(undefined, $event, undefined, undefined)}
```
建议将形参能去则去，请求数据时从组件的状态中获取参数或者根据不同情形重置参数。


### 3. 静态代码检测
#### 书写规范
1. js中的字符串、文件路径采用```'单引号'```
2. 统一文件命名，采用.分隔 ```tradeComposition -> deal.composition```
3. object.key不用引号或者使用单引号
4. 统一缩进
5. 弱等== 改为 强等===

#### 多余变量
6. 大量未使用的变量

###  4. 组件自治要彻底

```
<checkBoxHub source={metrics} defaulted={defaultMetrics}
             selected={tradeMetrics} periodType={periodType}>
</checkBoxHub>

// checkBoxHub
getCheckedBox: function() {
    ...
    for (var i = 0; i < data.checkBox.length; i++) {
        if (data.checkBox[i].checked){
            if( i === 3 && data.periodType!=0){
                data.checkBox[i].name = "日均UV";
            }
            ...
        }
    }
    ...
},
```
periodType不同的值会影响checkBoxHub组件字段和表头展示'UV/转化率'或者'日均UV/平均转化率'；而在父组件中也有这一功能的代码

```
// 初始化表头数据
if(data.periodType !=0){
    uvValue = '日均UV';
    tpValue = "平均转化率";
}else{
    uvValue = 'UV';
    tpValue = "转化率";
}

...

//监听时间段 修改交易构成表头
self.$watch('periodType',function(newValue,oldValue){
    if (newValue != undefined && oldValue != undefined) {
        if(newValue != 0){
            self.data.tradeMetrics[3].dimensionName = '日均UV';
            self.data.tradeMetrics[5].dimensionName = '平均转化率';
        }else{
            self.data.tradeMetrics[3].dimensionName = 'UV';
            self.data.tradeMetrics[5].dimensionName = '转化率';
        }
    }
});
```
checkBoxHub和父组件都没有把功能、逻辑做完整，代码不健壮，耦合度高；

后面watch里的赋值也直接引发了第4、5个线上bug。重构时将这一部分的代码全部在checkBoxHub中处理，降低耦合，只留periodType做接口

### ！！能不watch就不watch，尽量避免在watch中发请求

## 总结
1. 该页面有几个明显的问题：功能代码混乱，部分函数较长，大量冗余代码和错误代码，在线上还有明显bug。针对这些问题，着手理清功能和模块，拆解函数，组件间降低耦合，根据cleancode的原则整理代码，遵守静态代码规范。
2. 造成代码质量不高的原因可能是最初开发的人员对于regular的书写不够娴熟，代码也没有清晰的设计，组件接口设计不合理。除此之外，不能忽视的是后来的维护者。面对需求更改或者bugfix的时候只是想显式地快速地实现功能（但是也引发了bug），而忽略了代码本身可优化的空间，错过了当时重构的时机。
4. 之前芳芳总结重构的时候说的很对——“重构从来不是一次性行为，是我们需要不断进行的工作。多人维护以及不断的功能迭代之后，代码多多少少都会有优化的空间，所以在你看到不合理或任何值得重构的地方时，行动起来吧。”