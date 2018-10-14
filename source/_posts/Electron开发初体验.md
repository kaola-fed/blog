#### 需求背景
平时总会写markdown，markdown整体语法用起来很方便，但依然有晦涩的地方，比如表格。markdown的表格语法写起来很容易出错，而且每行每列单元格里的内容长短不一编辑器里就很容易乱掉，所以我在写表格时候都是借助[Tables Generator](https://www.tablesgenerator.com/markdown_tables)来写的，但是这个网站不能保存多个模板，每次写不同的表格都要把列数，表头信息来回改，很麻烦，于是打算自己照着Table Generator写一个简单的，能保存表头信息的东西出来。先看一哈大致的样子：

![](https://user-gold-cdn.xitu.io/2018/3/12/1621ab0a0adf8aaf?w=2000&h=1120&f=png&s=100111)


#### 动手前的思考
最初在考虑的是要不要写一个差不多的简单的页面，但是我个人不太喜欢总开新的tab来回切换，所以突然想到可以做成一个简单的桌面应用，想用的时候可以直接从Dock启动，而且Electron有了解过但没实际用过，也可以尝尝鲜，就决定用Electron直接做成一个小应用了。

#### 工程搭建
在这个"不用脚手架不舒服" + "不用框架不舒服"的时代，搭建工程当然是选择一款靠谱的脚手架了，开发环境 + 打包构建都能通过命令行搞定，极大程度地节省了时间，感谢开源贡献者吧~这里我选择了[electron-vue](https://github.com/SimulatedGREG/electron-vue)这个模板，基于vue-cli的，初始化项目很简单，直接执行:
```
vue init simulatedgreg/electron-vue my-project
```

然后根据提示输入完项目名，项目描述，依赖和构建工具(`electron-builder`或者`electron-packager`)后，一个项目就搭建完成了，进入目录执行：
```
yarn && yarn dev
```
然后项目就以开发模式运行起来了

![](https://user-gold-cdn.xitu.io/2018/3/12/1621aafbb59b83fb?w=2000&h=1170&f=png&s=282860)

后续的开发工作，如果你的应用对于系统级别的API需求不大，事实上和开发网页的体验并没有什么区别。比如我要完成的这个小工具就和开发网页的体验差不多。

#### 核心概念
实现的思路比较简单：为表格的每一个单元格设置`contenteditable`，这样整个表格的内容都是可以随意编辑的，然后再通过[MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)监听表格内容的变化，构造出正确的数据结构即可。

为th和td添加 contenteditable 使其内容可以编辑：
```html
<table id="table">
    <tbody>
      <tr>
        <th class="col-mark"></th>
        <th data-row="0" :data-col="index" v-title="item.text"
          v-for="(item, index) in columns" :key="item.key" contenteditable
          :class="{'active': index === acCol}"
        ></th>
      </tr>
    </tbody>
</table>
```
然后在渲染完成后使用MutationObserver监听变化：
```js
mounted () {
    this.targetNode = document.getElementById('table')
    // this.handleMutation是具体处理回调
    this.observer = new MutationObserver(this.handleMutation)
    this.observer.observe(this.targetNode, this.config)
}
```
这里的MutationObserver的配置是`{ subtree: true, characterData: true, childList: true }`作用分别是：
- characterData: 目标的数据被修改时触发，比如直接编辑td,th里的内容时，就是对其`textContent`的修改。
- subtree: 使MutationObserver也能响应后代元素内容的变化，因为我们只在table上绑定一个MutationObserver，所以使用这个属性。
- childList: 目标(包括后代节点)的子元素(包括文本元素)添加或者被删除时触发。比如单元格中的内容从无到有和从有到无的两种边界情况。

核心概念主要就用到了这两个概念，样式上选择了[papercss](https://github.com/papercss/papercss)，简单的功能搭配简洁的风格。

#### 复制到剪贴板
表格信息写好之后，最后的功能就是生成对应的markdown内容然后复制到粘贴板了。Electron提供了`clipboard API`，直接调用`clipboard.writeText`就能把内容写入粘贴板：

```js
import { clipboard } from 'electron'

let text = 'xxx'
clipboard.writeText(text)
```
这里复制成功后可以给出一个提示信息，我们在Electron开发的内容一般是在render进程的，在render进程中可以直接使用HTML5 Notification API来实现提示：

```js
let myNotification = new Notification('Table Generator', {
    body: 'Copy successfully~'
})
setTimeout(() => {
    myNotification.close()
}, 2000)
```
实际效果为一个两秒后自动消失的提示框：

![](https://user-gold-cdn.xitu.io/2018/3/12/1621ad404bcf17fd?w=1442&h=696&f=png&s=181383)

#### 主进程与渲染进程的通信
这里要实现的效果是通过自定义的快捷键删除左侧模板列表中的模板，但是快捷键注册只能在主进程通过`globalShortcut`注册，而对于删除行为的响应(二次确认的弹窗)是在渲染进程，所以设计到了主进程和渲染进程的通信。

渲染进程是主进程中创建的一个`BrowserWindow`实例，实例的`webContents`属性是对渲染进程的引用，所以主进程可以直接通过`webContents`发送事件：
```js
// 创建的渲染进程
mainWindow = new BrowserWindow({
    height: 560,
    minHeight: 450,
    width: 1000,
    minWidth: 760,
    titleBarStyle: 'hiddenInset',
    show: false,
    backgroundColor: '#fff'
})
// 注册快捷键
globalShortcut.register('Cmd+D', () => {
    // 直接通过mainWindow.webContents发送事件
    mainWindow.webContents.send('del-tpl')
})
```
而在对应的render进程，可以通过`ipcRenderer`监听消息：
```js
ipcRenderer.on('del-tpl', () => {
  // 触发modal弹出
  this.$refs['del-btn'].click()
})
```

#### 一些简单的优化
事实上通过一些简单的配置，就可以让你的应用体验更好：
- 创建无边框窗口：无边框窗口会让应用整体变得更美观，在创建BrowserWindow时通过`titleBarStyle: 'hiddenInset'`实现(针对Mac系统)
- 在创建BrowserWindow时通过`show: false`隐藏窗口,在`'ready-to-show'`事件触发时手动调用窗口实例的`show()`方法，保证窗口渲染完再展示。
- 通过将窗口背景颜色设置成渲染进程背景一样的颜色，让应用显得更快。

#### 总结
本次初步的尝试并没有用到太多系统级别的API，基本和开发Web页面体验一样，文中提到的API和优化点都是文档上可以找到的，本次实践只是对Electron的一次涉猎，后续可以考虑将各种操作和提示都迁移到原生的API，或者再加入其它功能，不过用来生成markdown内容“初心”已经达到了~

源码在[GayHub](https://github.com/showonne/md-table-generator)上，有兴趣的同学也可以自己安装依赖构建体验一哈，顺便点个star~ 
