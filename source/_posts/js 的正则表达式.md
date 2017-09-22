---
title: js 的正则表达式
date: 2017-08-10
---
## 正则表达式
一种几乎可以在所有的程序设计语言里和所有的计算机平台上使用的文字处理工具。它可以用来查找特定的信息（搜索），也可以用来查找并编辑特定的信息（替换）。  
核心是 匹配，匹配位置或者匹配字符

### 先简单的介绍一下语法
#### 基本元字符
0.  `.` ： 匹配除了换行符之外的任何单个字符
1.  `\` ： 在非特殊字符之前的反斜杠表示下一个字符是特殊的，不能从字面上解释。例如，没有前\\的'b'通常匹配小写'b'，无论它们出现在哪里。如果加了'\',这个字符变成了一个特殊意义的字符，反斜杠也可以将其后的特殊字符，转义为字面量。例如，模式 `/a*/` 代表会匹配 0 个或者多个 a。相反，模式 `/a\*/` 将 '\*' 的特殊性移除，从而可以匹配像 `"a*"` 这样的字符串。
2.  `|` ： 逻辑或操作符
3. `[]` ：定义一个字符集合，匹配字符集合中的一个字符，在字符集合里面像 `.`，`\`这些字符都表示其本身
4. `[^]`：对上面一个集合取非
5.  `-` ：定义一个区间，例如`[A-Z]`，其首尾字符在 ASCII 字符集里面
#### 数量元字符
1.  `{m,n}` ：匹配前面一个字符至少 m 次至多 n 次重复，还有`{m}`表示匹配 m 次，`{m,}`表示至少 m 次
2.  `+` ： 匹配前面一个表达式一次或者多次，相当于 `{1,}`，记忆方式追加(+)，起码得有一次
3.  `*` ： 匹配前面一个表达式零次或者多次，相当于 `{0,}`，记忆方式乘法(*)，可以一次都没有
6.  `?` ： 单独使用匹配前面一个表达式零次或者一次，相当于 `{0,1}`，记忆方式，有吗？，有(1)或者没有(1)，如果跟在任何量词`*`,`+`,`?`,`{}`后面的时候将会使量词变为非贪婪模式（尽量匹配少的字符），默认是使用贪婪模式。比如对 "123abc" 应用 `/\d+/` 将会返回 "123"，如果使用 `/\d+?/`,那么就只会匹配到 "1"。
#### 位置元字符
1.  `^` ： 单独使用匹配表达式的开始
5.  \$ ： 匹配表达式的结束
6. `\b`：匹配单词边界
8. `\B`：匹配非单词边界
1. `(?=p)`：匹配 p 前面的位置
2. `(?!p)`：匹配不是 p 前面的位置

#### 特殊元字符
1. `\d`：`[0-9]`，表示一位数字，记忆方式 digit
2. `\D`：`[^0-9]`，表示一位非数字
3. `\s`：`[\t\v\n\r\f]`，表示空白符，包括空格，水平制表符（`\t`），垂直制表符（`\v`），换行符（`\n`），回车符（`\r`），换页符（`\f`），记忆方式 space character
4. `\S`：`[^\t\v\n\r\f]`，表示非空白符
5. `\w`：`[0-9a-zA-Z]`，表示数字大小写字母和下划线，记忆方式 word
6. `\W`：`[^0-9a-zA-Z]`，表示非单词字符
#### 标志字符
1. `g` : 全局搜索 记忆方式global
2. `i` ：不区分大小写 记忆方式 ignore
3. `m` ：多行搜索

### 在 js 中的使用
#### 支持正则的 String 对象的方法
1. search
search 接受一个正则作为参数，如果参入的参数不是正则会隐式的使用 new RegExp(obj)将其转换成一个正则，返回匹配到子串的起始位置，匹配不到返回-1
2. match
接受参数和上面的方法一致。返回值是依赖传入的正则是否包含 g ，如果没有 g 标识，那么 match 方法对 string 做一次匹配，如果没有找到任何匹配的文本时，match 会返回 null ，否则，会返回一个数组，数组第 0 个元素包含匹配到的文本，其余元素放的是正则捕获的文本，数组还包含两个对象，index 表示匹配文本在字符串中的位置，input 表示被解析的原始字符串。如果有 g 标识，则返回一个数组，包含每一次的匹配结果
```
var str = 'For more information, see Chapter 3.4.5.1';
var re = /see (chapter \d+(\.\d)*)/i;
var found = str.match(re);

console.log(found);


// (3) ["see Chapter 3.4.5.1", "Chapter 3.4.5.1", ".1", index: 22, input: "For more information, see Chapter 3.4.5.1"]
// 0:"see Chapter 3.4.5.1"
// 1:"Chapter 3.4.5.1"
// 2:".1"
// index:22
// input:"For more information, see Chapter 3.4.5.1"
// length:3
// __proto__:Array(0)

// 'see Chapter 3.4.5.1' 是整个匹配。
// 'Chapter 3.4.5.1' 被'(chapter \d+(\.\d)*)'捕获。
// '.1' 是被'(\.\d)'捕获的最后一个值。
// 'index' 属性(22) 是整个匹配从零开始的索引。
// 'input' 属性是被解析的原始字符串。
```
```
var str = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz';
var regexp = /[A-E]/gi;
var matches_array = str.match(regexp);

console.log(matches_array);
// ['A', 'B', 'C', 'D', 'E', 'a', 'b', 'c', 'd', 'e']
```
3. replace
接受两个参数，第一个是要被替换的文本，可以是正则也可以是字符串，如果是字符串的时候不会被转换成正则，而是作为检索的直接量文本。第二个是替换成的文本，可以是字符串或者函数，字符串可以使用一些特殊的变量来替代前面捕获到的子串
变量名 | 代表的值
---|---
$$ | 插入一个 "$"。
$& | 插入匹配的子串。
$` | 插入当前匹配的子串左边的内容。
$' | 插入当前匹配的子串右边的内容。
$n | 假如第一个参数是 RegExp对象，并且 n 是个小于100的非负整数，那么插入第 n 个括号匹配的字符串。
```
var re = /(\w+)\s(\w+)/;
var str = "John Smith";
var newstr = str.replace(re, "$2, $1");
// Smith, John
console.log(newstr);
```
如果是函数的话，函数入参如下，返回替换成的文本
变量名 | 代表的值
---|---
match | 匹配的子串。（对应于上述的$&。）
p1,p2,... | 假如replace()方法的第一个参数是一个RegExp 对象，则代表第n个括号匹配的字符串。（对应于上述的$1，$2等。）
offset | 匹配到的子字符串在原字符串中的偏移量。（比如，如果原字符串是“abcd”，匹配到的子字符串是“bc”，那么这个参数将是1）
string | 被匹配的原字符串。
```
function replacer(match, p1, p2, p3, offset, string) {
  // p1 is nondigits, p2 digits, and p3 non-alphanumerics
  return [p1, p2, p3].join(' - ');
}
var newString = 'abc12345#$*%'.replace(/([^\d]*)(\d*)([^\w]*)/, replacer);
// newString   abc - 12345 - #$*%
```
4. split
接受两个参数，返回一个数组。第一个是用来分割字符串的字符或者正则，如果是空字符串则会将元字符串中的每个字符以数组形式返回，第二个参数可选作为限制分割多少个字符，也是返回的数组的长度限制。有一个地方需要注意，用捕获括号的时候会将匹配结果也包含在返回的数组中
```
var myString = "Hello 1 word. Sentence number 2.";
var splits = myString.split(/\d/);

console.log(splits);
// [ "Hello ", " word. Sentence number ", "." ]

splits = myString.split(/(\d)/);
console.log(splits);
// [ "Hello ", "1", " word. Sentence number ", "2", "." ]
```
#### 正则对象的方法
1. test
接受一个字符串参数，如果正则表达式与指定的字符串匹配返回 true 否则返回 false
2. exec
同样接受一个字符串为参数，返回一个数组，其中存放匹配的结果。如果未找到匹配，则返回值为 null。
匹配时，返回值跟 match 方法没有 g 标识时是一样的。数组第 0 个表示与正则相匹配的文本，后面 n 个是对应的 n 个捕获的文本，最后两个是对象 index 和 input  
同时它会在正则实例的 lastIndex 属性指定的字符处开始检索字符串 string。当 exec() 找到了与表达式相匹配的文本时，在匹配后，它将把正则实例的 lastIndex 属性设置为匹配文本的最后一个字符的下一个位置。
有没有 g 标识对单词执行 exec 方法是没有影响的，只是有 g 标识的时候可以反复调用 exec() 方法来遍历字符串中的所有匹配文本。当 exec() 再也找不到匹配的文本时，它将返回 null，并把 lastIndex 属性重置为 0。
```
var string = "2017.06.27";
var regex2 = /\b(\d+)\b/g;
console.log( regex2.exec(string) );
console.log( regex2.lastIndex);
console.log( regex2.exec(string) );
console.log( regex2.lastIndex);
console.log( regex2.exec(string) );
console.log( regex2.lastIndex);
console.log( regex2.exec(string) );
console.log( regex2.lastIndex);
// => ["2017", "2017", index: 0, input: "2017.06.27"]
// => 4
// => ["06", "06", index: 5, input: "2017.06.27"]
// => 7
// => ["27", "27", index: 8, input: "2017.06.27"]
// => 10
// => null
// => 0
```
其中正则实例lastIndex属性，表示下一次匹配开始的位置。

比如第一次匹配了“2017”，开始下标是0，共4个字符，因此这次匹配结束的位置是3，下一次开始匹配的位置是4。

从上述代码看出，在使用exec时，经常需要配合使用while循环：
```
var string = "2017.06.27";
var regex2 = /\b(\d+)\b/g;
var result;
while ( result = regex2.exec(string) ) {
	console.log( result, regex2.lastIndex );
}
// => ["2017", "2017", index: 0, input: "2017.06.27"] 4
// => ["06", "06", index: 5, input: "2017.06.27"] 7
// => ["27", "27", index: 8, input: "2017.06.27"] 10
```
### 正则的匹配
#### 字符匹配
精确匹配就不说了，比如/hello/，也只能匹配字符串中的"hello"这个子串。  
正则表达式之所以强大，是因为其能实现模糊匹配。  
##### 匹配多种数量
用`{m,n}`来匹配多种数量，其他几种形式(`+*?`)都可以等价成这种。比如
```
var regex = /ab{2,5}c/g;
var string = "abc abbc abbbc abbbbc abbbbbc abbbbbbc";
console.log( string.match(regex) ); // ["abbc", "abbbc", "abbbbc", "abbbbbc"]
```
贪婪和非贪婪:  
默认贪婪
```
var regex = /\d{2,5}/g;
var string = "123 1234 12345 123456";
console.log( string.match(regex) ); // ["123", "1234", "12345", "12345"]
```
两次后面加一个 ？ 就可以表示非贪婪，非贪婪时
```
var regex = /\d{2,5}?/g;
var string = "123 1234 12345 123456";
console.log( string.match(regex) ); // ["12", "12", "34", "12", "34", "12", "34", "56"]
```
##### 匹配多种情况
用字符组`[]`来匹配多种情况，其他几种形式(`\d\D\s\S\w\W`)都可以等价成这种。比如
```
var regex = /a[123]b/g;
var string = "a0b a1b a2b a3b a4b";
console.log( string.match(regex) ); // ["a1b", "a2b", "a3b"]
```
如果字符组里面字符特别多的话可以用`-`来表示范围，比如[123456abcdefGHIJKLM]，可以写成[1-6a-fG-M]，用[^0-9]表示非除了数字以外的字符   
多种情况还可以是多种分支，用管道符来连接`|`，比如  
```
var regex = /good|goodbye/g;
var string = "goodbye";
console.log( string.match(regex) ); // ["good"]
```
这个例子可以看出分支结构也是惰性的，匹配到了就不再往后尝试了。
##### 例子
掌握这两种方式就可以解决比较简单的正则问题了。
1. 最多保留2位小数的数字  
`/^([1-9]\d*|0)(\.\d{1,2})?$/`
2. 电话号码
`/(\+86)?1\d{10}/`
3. 身份证
`/^(\d{15}|\d{17}([xX]|\d))$/`


#### 位置匹配
##### 什么是位置
位置是相邻字符之间的，比如，有一个字符串 `hello` ，这个字符串一共有6个位置 `*h*e*l*l*o*` ， *代表位置
![image](https://pic4.zhimg.com/v2-c487b9402a935625be4dd4b0e4f5fe5f_r.png)
上面说到了 6 种位置元字符
1. `^`，`$` 匹配字符的开头和结尾，比如
`/^hello$/` 匹配一个字符串，要符合这样的条件，字符串开头的位置，紧接着是 h 然后是 e,l,l,o 最后是字符串结尾的位置  
位置还可以被替换成字符串，比如  
`'hello'.replace(/^|$/g, '#')` 结果是 `#hello#`
2. `/b`，`/B` 匹配单词边界和非单词边界，单词边界具体指 `\w`(`[a-zA-Z0-9_]`) 和 `\W` 之间的位置，包括 `\w` 和 `^` 以及 `$` 之间的位置，比如
`'hello word [js]_reg.exp-01'.replace(/\b/g, '#')` 结果是 `#hello# #word# [#js#]#_reg#.#exp#-#01#`
3. `(?=p)`，`(?!p)` 匹配 p 前面的位置和不是 p 前面位置，比如  
`'hello'.replace(/(?=l)/g, '#')` 结果是 `he#l#lo`  
`'hello'.replace(/(?!l)/g, '#')` 结果是 `#h#ell#o#`  
##### 位置的特性
字符与字符之间的位置可以是多个。在理解上可以将位置理解成空字符串 ''，比如
`hello` 可以是一般的 `'' + 'h' + 'e' + 'l' + 'l' + 'o' + ''`，也可以是 `'' + '' + '' + '' + 'h' + 'e' + 'l' + 'l' + 'o' + ''`，  
所以`/^h\Be\Bl\Bl\Bo$/.test('hello')` 结果是 true，`/^^^h\B\B\Be\Bl\Bl\Bo$$$/.test('hello')` 结果也是 true
##### 例子
1. 千分位，将 123123123 转换成 123,123,123
数字是从后往前数，也就是以一个或者多个3位数字结尾的位置换成 ',' 就好了，写成正则就是  
`123123213.replace(/(?=(\d{3})+$)/g, ',')` 但是这样的话会在最前面也加一个 ',' 这明显是不对的。所以还得继续改一下正则  
要求匹配到的位置不是开头，可以用 `/(?!^)(?=(\d{3})+$)/g` 来表示。  
换种思路来想，能不能是以数字开头然后加上上面的条件呢，得出这个正则 `/\d(?=(\d{3})+$)/g`，但是这个正则匹配的结果是 `12,12,123`，发现这个正则匹配的不是位置而是字符，将数字换成了 ','  可以得出结论，如果要求一个正则是匹配位置的话，那么所有的条件必须都是位置。

#### 分组
分组主要是括号的使用
#### 分组和分支结构
在分支结构中，括号是用来表示一个整体的，(p1|p2)，比如要匹配下面的字符串
```
I love JavaScript
I love Regular Expression
```
可以用正则`/^I love (JavaScript|Regular Expression)$/` 而不是 `/^I love JavaScript|Regular Expression$/`  
表示一个整体还比如 `/(abc)+/ ` 一个或者多个 abc 字符串  
上面这些使用 () 包起来的地方就叫做分组  
```
'I love JavaScript'.match(/^I love (JavaScript|Regular Expression)$/)
// ["I love JavaScript", "JavaScript", index: 0, input: "I love JavaScript"]
```
输出的数组第二个元素，"JavaScript" 就是分组匹配到的内容
#### 引用分组
##### 提取数据
比如我们要用正则来匹配一个日期格式，yyyy-mm-dd，可以写出简单的正则`/\d{4}-\d{2}-\d{2}/`，这个正则还可以改成分组形式的`/(\d{4})-(\d{2})-(\d{2})/`  
这样我们可以分别提取出一个日期的年月日，用 String 的 match 方法或者用正则的 exec 方法都可以
```
var regex = /(\d{4})-(\d{2})-(\d{2})/;
var string = "2017-08-09";
console.log( string.match(regex) ); 
// => ["2017-08-09", "2017", "08", "09", index: 0, input: "2017-08-09"]
```
也可以用正则对象构造函数的全局属性 $1 - $9 来获取
```
var regex = /(\d{4})-(\d{2})-(\d{2})/;
var string = "2017-08-09";

regex.test(string); // 正则操作即可，例如
//regex.exec(string);
//string.match(regex);

console.log(RegExp.$1); // "2017"
console.log(RegExp.$2); // "08"
console.log(RegExp.$3); // "09"
```
##### 替换
如果想要把 yyyy-mm-dd 替换成格式 mm/dd/yyyy 应该怎么做。  
String 的 replace 方法在第二个参数里面可以用 $1 - $9 来指代相应的分组
```
var regex = /(\d{4})-(\d{2})-(\d{2})/;
var string = "2017-08-09";
var result = string.replace(regex, "$2/$3/$1");
console.log(result); // "08/09/2017"
等价
var result = string.replace(regex, function() {
	return RegExp.$2 + "/" + RegExp.$3 + "/" + RegExp.$1;
});
console.log(result); // "08/09/2017"
等价
var regex = /(\d{4})-(\d{2})-(\d{2})/;
var string = "2017-08-09";
var result = string.replace(regex, function(match, year, month, day) {
	return month + "/" + day + "/" + year;
});
console.log(result); // "08/09/2017"
```
#### 反向引用
之前匹配日期的正则在使用的时候发现还有另外两种写法，一共三种
```
2017-08-09

2017/08/09

2017.08.09
```
要匹配这三种应该怎么写正则，第一反应肯定是把上面那个正则改一下`/(\d{4})[-/.](\d{2})[-/.](\d{2})/`，把 - 改成 [-/.] 这三种都可以  
看上去没问题，我们多想想就会发现，这个正则把 `2017-08.09` 这种字符串也匹配到了，这个肯定是不符合预期的。  
这个时候我们就需要用到反向引用了，反向引用可以在匹配阶段捕获到分组的内容 `/(\d{4})([-/.])(\d{2})\2(\d{2})/`  
##### 那么出现括号嵌套怎么办，比如 
```
var regex = /^((\d)(\d(\d)))\1\2\3\4$/;
var string = "1231231233";
console.log( regex.test(string) ); // true
console.log( RegExp.$1 ); // 123
console.log( RegExp.$2 ); // 1
console.log( RegExp.$3 ); // 23
console.log( RegExp.$4 ); // 3
```
嵌套的括号以左括号为准
##### 引用了不存在的分组呢
如果在正则里面引用了前面不存在的分组，这个时候正则会匹配字符本身，比如`\1`就匹配`\1`

#### 非捕获分组
我们有时候只是想用括号原本的功能而不想捕获他们。这个时候可以用`(?:p)`表示一个非捕获分组

#### 例子
1. 驼峰改短横
```
function dash(str) {
  return str.replace(/([A-Z])/g, '-$1').toLowerCase();
}
```
2. 获取链接的 search 值
链接：`https://www.baidu.com?name=jawil&age=23`
```
function getParamName(attr) {

  let match = RegExp(`[?&]${attr}=([^&]*)`) //分组运算符是为了把结果存到exec函数返回的结果里
    .exec(window.location.search)
  //["?name=jawil", "jawil", index: 0, input: "?name=jawil&age=23"]
  return match && decodeURIComponent(match[1].replace(/\+/g, ' ')) // url中+号表示空格,要替换掉
}
  
console.log(getParamName('name'))  // "jawil"
```
3. 去掉字符串前后的空格
```
function trim(str) {
    return str.replace(/(^\s*)|(\s*$)/g, "")
}
```
4. 判断一个数是否是质数
```
function isPrime(num) {
  return !/^1?$|^(11+?)\1+$/.test(Array(num+1).join('1'))
}
```
这里首先是把一个数字变成1组成的字符串，比如11就是 '1111111111' 11个1  
然后正则分两部分，第一部分是匹配空字符串或者1  
第二部分是先匹配两个或者多个1，非贪婪模式，那么先会匹配两个1，然后将匹配的两个1分组，后面就是匹配一个或者多个 '2个1'，就相当于整除2，如果匹配成功就证明不是质数，如果不成功就会匹配3个1，然后匹配多个3个1，相当于整除3，这样一直下去会一直整除到自己本身。如果还是不行就证明这个数字是质数。  

#### 回溯
##### 正则是怎么匹配的
有这么一个字符串 'abbbc' 和这么一个正则 `/ab{1,3}bbc/` 
`/ab{1,3}bbc/.test('abbbc')` 我们一眼可以看出来是 true，但是 JavaScript 是怎么匹配的呢
![image](https://pic4.zhimg.com/v2-946e41a0f5167f52304ca2ff821f8247_r.png)

##### 回溯
例如我们上面的例子，回溯的思想是，从问题的某一种状态（初始状态）出发，搜索从这种状态出发所能达到的所有“状态”，当一条路走到“尽头”的时候（不能再前进），再后退一步或若干步，从另一种可能“状态”出发，继续搜索，直到所有的“路径”（状态）都试探过。这种不断“前进”、不断“回溯”寻找解的方法，就称作“回溯法”  
贪婪和非贪婪的匹配都会产生回溯，不同的是贪婪的是先尽量多的匹配，如果不行就吐出一个然后继续匹配，再不行就再吐出一个，非贪婪的是先尽量少的匹配。如果不行就再多匹配一个，再不行就再来一个  
分支结构也会产生回溯，比如
`/^(test|te)sts$/.test('tests')` 前面括号里面的匹配过程是先匹配到 test 然后继续往后匹配匹配到字符 `s` 的时候还是成功的，匹配到 `st` 的时候发现不能匹配， 所以会回到前面的分支结构的其他分支继续匹配，如果不行的话再换其他分支。

#### 读正则
读懂其他人写的正则也是一个很重要的方面。
##### 结构和操作符
结构：`字符字面量、字符组、量词、锚字符、分组、选择分支、反向引用。`
操作符：
1. 转义符 \
2. 括号和方括号 (...)、(?:...)、(?=...)、(?!...)、[...]
3. 量词限定符 {m}、{m,n}、{m,}、?、*、+
4. 位置和序列 ^ 、$、 \元字符、 一般字符
5. 管道符（竖杠） |  

操作符的优先级是从上到下，由高到低的，所以在分析正则的时候可以根据优先级来拆分正则，比如
`/ab?(c|de*)+|fg/`
1. 因为括号是一个整体，所以`/ab?()+|fg/`,括号里面具体是什么可以放到后面再分析
2. 根据量词和管道符的优先级，所以`a`, `b?`, `()+`和管道符后面的`f`, `g`
3. 同理分析括号里面的`c|de*` => `c`和`d`, `e*`
4. 综上，这个正则描述的是![image](https://pic3.zhimg.com/v2-3fd92f44ca68589a15ffbf110c6dea4e_r.png)  
以这种模式来分析，再复杂的正则都可以看懂。有一个可视化的[正则分析网站](https://jex.im/regulex)
