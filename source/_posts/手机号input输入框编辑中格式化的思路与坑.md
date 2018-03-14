<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

产品经理提了个需求, 编辑手机号的输入框, 要在编辑过程中对手机号进行3-4-4的格式化, 即往手机号中插入空格:

格式化前: 13344445555

格式化后: 133 4444 5555

之前的交互是当输入框失去焦点后对其内容做格式化, 编辑过程中不管.

一开始的思路是, 直接监听`change`时间, 检查输入框中的号码, 如果输入的位数超过三个或七个, 则在第三个数字和第七个数字后面添加一个空格. 这个思路只考虑了从前往后输入的情况, 有不限于以下问题:

1. 对于从输入了号码, 又删除号码的操作, 如果删到剩三个或者七个号的时候, 就会出问题, 程序不知道当前是在输入还是在删除;
2. 如果用户输入完, 需要修改号码中的某几位时, 需要把光标移动到要编辑的位置, 当输入了一个数字, 光标马上回移动到整段文本的末尾, 而不是在输入数字的后面. 输入的快的话, 就会导致用户输入的数字顺序和想要的顺序不同. 怀疑是因为input的内容被js改变了导致的, 我在业务中修改了input中的model值, 框架底层可能会执行如下代码:

```javascript
input.value = xxx
```

如果执行了这个代码, input的光标自然会移动到最后. 

解决方式, 就是在框架update之后再把光标的位置改回原来的位置的后面一个位置
对于移动input中光标的操作, 在网上找到下面这段代码:
```javascript
function setCursorPosition(elem, pos) {
        if (elem.setSelectionRange) {
            elem.setSelectionRange(pos, pos);
        } else if (elem.createTextRange) {
            var range = elem.createTextRange();
            range.collapse(true);
            range.moveEnd('character', pos);
            range.moveStart('character', pos);
            range.select();
        }
    }
```

如果用户把光标移动到手机号码中第三位/第七位数字后的空格后面, 然后按删除键, 我们把这种行为视作是删除第三位/第七位数字. 判断的方法是, 监听input输入框的keyup, 对事件后的值和原来的值做对比, 如果事件后值格式化后与事件前的值格式化后不同, 但取消格式化空格之后的值相同, 说明是用户删掉了一个空格, 那么就将光标前面那个数字也删除掉, 将删掉空格和数字的值再做格式化, 作为新的值, 并记住删除后光标的位置. 用来在timeout中对光标位置进行设置

当输入的手机号位数由3变化到4或由7变化到八时,  需要判断光标的位置, 如果光标在输入前是在第三个数字之后, 说明输入的是第四个数字, 输入后要讲光标后移两位, 因为前面插入了一个用来格式化的空格. 第七位逻辑一样.  
```javascript
//删除空格的情况
            if (oldActualValue === newActualValue) {
                var newFormatedValue = oldVal.slice(0, cursorPosition - 1) + oldVal.substr(cursorPosition + 1);
                actualPhone = self.getUnformattedPhone(newFormatedValue);
                cursorPosition -= 1;
            }

// 输入第四位数和第八位数的情况
            if (this.isAddNumberAfterSpace(oldActualValue, newActualValue)) {
                if (cursorPosition === 4 || cursorPosition === 9) {
                    cursorPosition++;
                }
            }

//...
        data.phone = _.formatMobileNo(actualPhone);
        data.oldPhone = data.phone;

//...
        setTimeout(function() {
                setCursorPosition(inputRef, cursorPosition);
        }, 0);

isAddNumberAfterSpace: function(oldVal, newVal) {
            var oldLen = oldVal.length;
            var newLen = newVal.length;
            if (oldLen !== 3 && oldLen !== 7) {
                return false;
            }
            return newLen - oldLen === 1;
        }
```

这是目前能想到的解决方案, 有一个缺陷, 就是输入数字后, 和在timeout中设置光标位置的过程中, 会有一个光标跳到整个内容后面又回到正确位置的闪动. 一开始timeout的延时设置到50, 闪动比较厉害, 后来干脆设置成0, 闪动不是很明显了, 但是有些机型还是会看到. 

第二个问题, 有些机型, 从通讯录中拷贝手机号粘贴进来之后, 最后一位号码会被截断, 检查后发现是input的maxlength设置过小导致的, 还有, iphonex的通讯录复制出来的手机号, 前面会有一个非空格的空白字符, 需要在onpaste事件回调里处理一下, 此处不再赘述

此外, 为了防止用户输入特殊符号, 添加了当keyCode不在数字对应的keyCode范围内时, 做一些特殊过滤的逻辑. 但是这样有遇到一个坑, 在一些安卓设备的软键盘中, keyCode和标准的电脑键盘不一样, 总是229. 这导致这部分安卓机将用户输入数字是的keyCode也当做非法字符进行过滤, 现象就是无法输入数字. 无奈, 把根据keyCode做判断的一些逻辑去掉了.

---- by yubaoquan