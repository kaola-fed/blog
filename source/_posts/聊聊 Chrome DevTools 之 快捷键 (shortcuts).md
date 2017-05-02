---
title: 聊聊 Chrome DevTools 之 快捷键 (shortcuts)
date: 2017-02-28
---

# 快捷键（shortcuts）

> 工欲善其事，必先利其器。——《论语·卫灵公》

Chrome DevTools 的快捷键，可以帮助开发者在日常开发的过程中节约时间（甚至可以说是大量的时间，具体看天赋咯）。

下面使用表格的方式，列举每个快捷方式在Windows/Linux和Mac下相应的快捷按键。

* 注：有些快捷键是在全局有效的，而有些只是在某一个面板生效。

<!-- more -->

***

### 1、打开DevTools

功能       | Windows / Linux | Mac
:------------- | :--- | :-------------------------------------
打开 Chrome DevTools          | F12, Ctrl + Shift + I   | Cmd + Opt + I
打开／切换 审查元素模式和浏览模式   | Ctrl + Shift + C   | Cmd + Shift + C
打开 Chrome DevTools ,并聚焦在 console 上 | Ctrl + Shift + J   | Cmd + Opt + J
审查审查器 (取消第一个审查器的停靠后再按键)  | Ctrl + Shift + J   | Cmd + Opt + J

*	注：
    * 非快捷键打开DevTools
        1. 打开浏览器窗口右上方的菜单（三个点），选择 -> **更多工具 > 开发者工具**
        2. 在任意的页面元素中右键，选择 -> **检查**

***

### 2、DevTools下全局方式

功能       | Windows / Linux | Mac
:------------- | :--- | :-------------------------------------
打开 settings(console和sources下无效)|  ?, F1  |  Shift + ?
下一个面板 |  Ctrl + ]  |   Cmd + ]
上一个面板 |  Ctrl + [  |   Cmd + [
标签历史中后退    |  Ctrl + Alt + [    | Cmd + Alt + [
标签历史中前进    |   Ctrl + Alt + ]    | Cmd + Alt + ]
跳转至标签页 1-9 (需要在设置中开启)    |   Ctrl + 1~9    | Cmd + 1~9
打开/关闭 Console 或  关闭设置对话框   |   Esc  |  Esc
刷新页面     |  F5, Ctrl + R     |  Cmd + R
强制刷新页面，清除缓存内容    |   Ctrl+F5, Ctrl + Shift + R   | Cmd + Shift + R
当前文件或标签页搜索文字   |   Ctrl + F  |     Cmd + F
所有资源中搜索文字 |  Ctrl + Shift + F    |   Cmd + Alt + F
搜索文件(除了 Timeline面板)  |  Ctrl + O, Ctrl + O  |   Cmd + O, Cmd + O
恢复默认字体大小  |    Ctrl + 0   |    Shift + 0
放大  |  Ctrl + +     |  Shift + +
缩小   |    Ctrl + -     |  Shift + -

***

### 3、Elements（DOM节点面板）

功能       | Windows / Linux | Mac
:------------- | :--- | :-------------------------------------
撤销改动 | Ctrl + Z    | Cmd + Z
恢复改动 | Ctrl + Y    | Cmd + Y, Cmd + Shift + Z
选中节点（不会去展开）|  ↑，↓  |  ↑，↓
伸缩展开元素  | ←，→  | ←，→ 
编辑元素属性|   Enter|   Enter
隐藏元素   |  H |   H
Edit as HTML|  F2|

***

### 4、Elements（Styles样式侧边栏）

功能       | Windows / Linux | Mac
:------------- | :--- | :-------------------------------------
跳转到css具体行数|  Ctrl + 单击某个CSS属性/选择器 |   Cmd + 单击某个CSS属性/选择器
循环切换颜色定义（rgb/a、#、hsl）|  Shift + 单击颜色选择器  | Shift + 单击颜色选择器
查看属性提示（一般与spotlight冲突） |  Ctrl + 空格 |   Cmd + 空格
编辑下一个 / 上一个属性  |  Tab, Shift + Tab  |   Tab, Shift + Tab
增大 / 减小属性值（+1 / -1） | ↑，↓   |  ↑，↓
增大 / 减小属性值 （+10 / -10 ） |  Shift + ↑, Shift + ↓  |   Shift + ↑, Shift + ↓
增大 / 减小属性值 （+100 / -100）  | Shift + PgUp, Shift + PgDown    | Shift + PgUp, Shift + PgDown
增大 / 减小属性值 （+0.1 / -0.1）  | Alt + ↑, Alt + ↓  |   Opt + ↑, Opt + ↓

* 注
    * :hov——模拟元素伪类 (:active, :hover, :focus, :visited)
    * .cls——快速编辑元素class

***

### 5、Console

功能       | Windows / Linux | Mac
:------------- | :--- | :-------------------------------------
下一个提示| Tab | Tab
上一个提示 | Shift + Tab | Shift + Tab
使用提示 |  → |   →
上/下一个命令/行 | ↑，↓ |  ↑，↓
清除控制台记录   | Ctrl + L   |  Cmd + K, Opt + L
多行输入   |  Shift + Enter  |  Ctrl + Enter
执行|  Enter  |  Enter

* 注
    * console 中右键单击
        *	XMLHTTPRequest 记录: 打开后可查看 XHR 记录
        *	Filter 过滤: 隐藏或显示所有来自脚本文件的消息
        *	Clear console 清除: 清除所有的 console 消息

***

### 6、资源(Sources)面板

功能       | Windows / Linux | Mac
:------------- | :--- | :-------------------------------------
中断/恢复脚本执行 | F8, Ctrl + \    | F8, Cmd + \
跳过下一个函数   | F10, Ctrl + '   | F10, Cmd + '
跳入下一个函数    | F11, Ctrl + ;   | F11, Cmd + ;
跳出当前函数    | Shift + F11, Ctrl + Shift + ; |   Shift + F11, Cmd + Shift + ;
Select next call frame |  Ctrl + .  |   Opt + .
Select previous call frame |  Ctrl + ,  |   Opt + ,
切换断点状态 | 单击行数, Ctrl + B |  单击行数, Cmd + B
编辑断点调节  |  右键单击行数 |  右键单击行数
Delete individual words | Alt + Delete  |   Opt + Delete
注释某行或选择文字  | Ctrl + /   |  Cmd + /
保存本地的更改| Ctrl + S  |   Cmd + S
保存所有的更改| Ctrl + Shift + S  |   Cmd + Shift +  S
跳转到某行 |  Ctrl + G  |   Ctrl + G
跳转到某行（Jump to line number） | Ctrl + P -> :number   |  Cmd + P -> :number
按文件名搜索文件|  Ctrl + O  |   Cmd + O
跳转到某列 | Ctrl + O + :<number> + :<number>   |  Cmd + O + :<number> + :<number>
打开 member  |   Ctrl + Shift + O  |   Cmd + Shift + O
切换 console 并评估（ evaluate？） Sources 面板中选中的代码|  Ctrl + Shift + E   |  Cmd + Shift + E
关闭当前激活的标签 | Alt + W   |  Opt + W
运行代码片段 | Ctrl + Enter   |  Cmd + Enter
切换注释  | Ctrl + /   |  Cmd + /

***

### 7、Timeline Panel & Profiles Panel

功能       | Windows / Linux | Mac
:------------- | :--- | :-------------------------------------
开启/停止   记录 |  Ctrl + E  |   Cmd + E
保存时间轴数据 |  Ctrl + S  |   Cmd + S
加载时间轴数据 | Ctrl + O   |  Cmd + O
开启/停止   记录 |  Ctrl + E  |   Cmd + E

***

### 8、Chrome Browser 快捷键（非 DevTools）

以下的 Chrome 快捷键在日常使用中非常有用，它并不是特意为 DevTools开发的.

功能       | Windows / Linux | Mac
:------------- | :--- | :-------------------------------------
（页面查找）寻找下一个   |Ctrl + G    |Cmd + G
（页面查找）寻找上一个  | Ctrl + Shift + G  |  Cmd + Shift + G
在隐身模式下打开一个新窗口|Ctrl + Shift + N   | Cmd + Shift + N
开启或关闭书签栏|Ctrl + Shift + B  |  Cmd + Shift + B
查看历史记录   |Ctrl + H   | Cmd + Y
查看下载记录 |Ctrl + J  |  Cmd + Shift + J
查看任务管理器   |Shift + ESC |Shift + ESC
标签浏览历史中的下一个页面  |  Alt + Right |Alt + Right
标签浏览历史中的上一个页面 |  Backspace, Alt + Left   |Backspace, Alt + Left
高亮地址栏内容 | F6, Ctrl + L, Alt + D  | Cmd + L, Alt + D
在地址栏输入一个 ? 后可以将它作为你的默认搜索引擎使用（英文输入法下） |Ctrl + K, Ctrl + E  |Cmd + K, Cmd + E

***

### 小结

虽然不会使用或不知道快捷键并不影响开发，它只是解放了（大）部分依赖鼠标的操作，但是从开发效率以及熟练度上来说，还是非常建议使用的，毕竟，“工欲善其事，必先利其器”（首尾呼应，满分。鼓掌.jpg）。

* 附
    * [所有 Chrome 快捷键](https://support.google.com/chrome/answer/157179?hl=en&topic=25799&rd=1)


