---
title: JS计算精度及舍去几位小数点后的值
date: 2017-6-25
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->


#### 加减乘除精度处理

JS的加减乘除，在一些对浮点数进行操作时情况下会出现精度问题。  
简单的方法是对数字进行扩大一定倍数之后，再进行运算，测试发现进行**10000**倍最合适

<!-- more -->

    mathPrecisionFixed = function(a, b, type){
        type = type || 1;
        if(type === 1){  //加法
            return (a*10000 + b*10000)/10000;
        }else if(type === 2){    //减法
            return (a*10000 - b*10000)/10000;
        }else if(type === 3){    //乘法
            return a * 10000 * b/10000;
        }else if(type === 4){    //除法
            return (a*10000) / (b*10000)
        }else{
            return '';
        }
    }

还有另一种方法，根据2个数值的倍数来进行计算来处理
    
    //乘法
    function mul(a, b){
        //TODO 校验a, b正确性
    
        var timesA, timesB;
        try{timesA = a.toString().split('.')[1].length;}catch(e){timesA = 0};
        try{timesB = b.toString().split('.')[1].length;}catch(e){timesB = 0};
    
        var r1 = Number(a.toString().replace('.', '')),
            r2 = Number(b.toString().replace('.', ''));
    
        return r1*r2/Math.pow(10,timesA+timesB);
    }
    //除法
    function divis(a, b){
        var timesA, timesB, m;
        try{timesA = a.toString().split('.')[1].length;}catch(e){timesA = 0};
        try{timesB = b.toString().split('.')[1].length;}catch(e){timesB = 0};
    
        var r1 = Number(a.toString().replace('.', '')),
            r2 = Number(b.toString().replace('.', ''));
    
        return (r1/r2)/Math.pow(10,timesA-timesB);
    }
    //加法
    function add(a, b){
        var timesA, timesB, m;
        try{timesA = a.toString().split('.')[1].length;}catch(e){timesA = 0};
        try{timesB = b.toString().split('.')[1].length;}catch(e){timesB = 0};
    
        m = Math.pow(10, Math.max(timesA, timesB))
    
        return (a*m + b*m)/m;
    }
    //减法
    function sub(a, b){
        var timesA, timesB, m;
        try{timesA = a.toString().split('.')[1].length;}catch(e){timesA = 0};
        try{timesB = b.toString().split('.')[1].length;}catch(e){timesB = 0};
    
        m = Math.pow(10, Math.max(timesA, timesB))
    
        return (a*m - b*m)/m;
    }



#### 指定舍去几位小数点后的值

    mathFloorFixed = function(number, fixedDigit){
        fixedDigit = fixedDigit || 2;
        var list = (number||0).toString().split('.');
        if (list[1] && list[1].length > fixedDigit) {
            return Number(list[0] + '.' + list[1].substr(0, fixedDigit))
        }else{
            return number;
        }
    }
    
by jiangren