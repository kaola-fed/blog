---
title: chrome插件开发简介（二）——如何添“加浏览器扩展白名单”
date: 2017-03-31
---

# chrome插件开发简介（二）——如何添“加浏览器扩展白名单”

> 没有在Chrome应用商店web store上架发布的插件，如果没有添加到白名单里，下一次重启Chrome就会被禁用，而且无法手动启用，除非删掉重新添加。
可以通过添加白名单可以一劳永逸地解决这个问题。

下面我们以“将日报插件添加到白名单”为例，讲解步骤

<!-- more -->

### Mac下

mac系统下设置白名单比较简单，下载[com.google.Chrome.mobileconfig](https://github.com/Froguard/crxs/blob/master/doc/res/mac-install-chrome-extension-whitelist/com.google.Chrome.mobileconfig)，后，双击安装即可（过程中可能会要求输入系统管理员密码）

> 当然，为适配本插件，该文件内容已经更改:

> 1.文本编辑打开文件 com.google.Chrome.mobileconfig

> 2.找到 &lt;array&gt; 之下的 &lt;string&gt; 标签，然后在里面输入插件的id

> 3.保存之后，双击运行这个文件，期间可能会要求输入管理员密码，输入即可

> 4.重启浏览器

### windows下

1.```Win``` + ```R```（或者打开左下角运行）输入```gpedit.msc```

2.本地计算机策略 > 计算机配置 > 管理模板，右键管理模板，选择添加/删除模板。

![image](https://github.com/Froguard/crxs/raw/master/doc/res/step2.jpg)

&nbsp;<br>&nbsp;

3.点击添加，将下载的chrome.adm（下载地址：[res/chrome.adm](https://github.com/Froguard/crxs/blob/master/doc/res/chrome.adm)）添加进来。

![image](https://github.com/Froguard/crxs/raw/master/doc/res/step3.jpg)

&nbsp;<br>&nbsp;

4.添加模板完成后，找到经典管理模板（ADM），点击进入，选择Google > Google Chrome

![image](https://github.com/Froguard/crxs/raw/master/doc/res/step4.jpg)

&nbsp;<br>&nbsp;

5.打开后如下图所示，选择扩展程序，点击进入，配置扩展程序安装白名单

![image](https://github.com/Froguard/crxs/raw/master/doc/res/step5-1.jpg)

&nbsp;&nbsp;↓↓↓

![image](https://github.com/Froguard/crxs/raw/master/doc/res/step5-2.jpg)

&nbsp;<br>&nbsp;

6.点击下图中的显示按钮，将我们安装插件后的ID添加到那个值列表中，点击确定返回即可。比如我们的是 **nldeakmmeccgpaiacgpmabnjaenfdbkn**

![image](https://github.com/Froguard/crxs/raw/master/doc/res/step6-1.jpg)

id信息如下：打开chrome的插件管理界面（地址栏 [chrome://extensions](chrome://extensions/)），找到这个插件

![image](https://github.com/Froguard/crxs/raw/master/doc/res/step6-2.png)

> 这里需要提一点，如果未开启开发者模式，可能会看不到这个id

![image](https://github.com/Froguard/crxs/raw/master/doc/res/dev-mode.png)

&nbsp;<br>&nbsp;