---
title: MediumEditor富文本编辑器介绍与实战
date: 2018-01-26
---
> [MediumEditor](https://yabwe.github.io/medium-editor/)是一款开源又轻量的行内工具栏编辑器，有三大特点：纯js实现不依赖js库，压缩后紧28k很轻量，兼容高级浏览器及IE9+，另外TA的扩展性非常强，堪称神器了，在github上收获1w+个赞，1.4k+的fork，活跃度也是非常高了。在近期的项目中使用了此插件，下面就来分享一些使用体验和总结。

[MediumEditor]兼容性：

![image](https://cloud.githubusercontent.com/assets/2444240/12874138/d3960a04-cd9b-11e5-8cc5-8136d82cf5f6.png)


## 认识

![image](https://raw.github.com/yabwe/medium-editor/master/demo/img/medium-editor.jpg)

上图展示了`MediumEditor`这款行内工具式编辑器，比较轻量简单，跟传统的菜单式的富文本编辑器使用体验上有很大差异，更试用于一些**重内容轻排版**的创作平台。支持对文本的加粗、设为标题、设为超链接等基本功能，TA提供很全面的API，来实现对内容的个性化编辑，其强大的扩展性可以很方便的集成你自定义的extensions。


## 使用
`MediumEditor`上手使用基本无压力，文档与接口说明（虽然是英文）也很清晰。
近期的项目是基于vuejs框架，下面也就介绍下此款编辑器在实际项目的使用姿势。

- ### 安装

当然首先记得install下,通过npm:
`npm install medium-editor`
or 通过bower: `bower install medium-editor`

- ### 实例化

```
import MEditor from 'medium-editor'
import 'medium-editor/dist/css/medium-editor.css'
import 'medium-editor/dist/css/themes/default.css' // 此款编辑器的默认主题样式文件

export default {
 name: 'meditor',
 //..
 methods: {
  initEditor () {
    let option = {}
    // 实例化时，传入要生成编辑器的容器节点selector，以及编辑器的配置项option
    this.editor = new MEditor('.meditor', option)      
  }
 }

```
The above code will transform all the elements with the .editable class into HTML5 editable contents and add the medium editor toolbar to them.
实例化之后，会将'.meditor'选择器对应的所有元素设为HTML5的可编辑器内容，即`contenteditable="true"`，同时给这些元素添加编辑工具栏，当在元素中输入文本再选中一些文本后，就会在所选文本的上方出现行内工具条啦。

- ### options配置

`MediumEditor`的配置项很强大，在 初始化时传入基于自己业务的配置项，基本就能满足常规使用。`MediumEditor`完整的option配置项[戳我](https://github.com/yabwe/medium-editor/blob/master/OPTIONS.md)。
下面来解密项目中使用到的一些配置项：


```
import { Message } from 'element-ui'
import PasteExtension from './pasteExtension'

export default {
  // 工具栏或链接预览框出现的延时，默认：0
  delay: 100, 
  // 工具栏配置，若想禁用工具栏：toolbar: false
  toolbar: {
    // 是否允许多段落可选
    allowMultiParagraphSelection: true,
    // 设置工具栏中按钮列表，默认值：['bold', 'italic', 'underline', 'anchor', 'h2', 'h3', 'quote']    
    buttons: [
      {
        name: 'h2',
        aria: '标题',
        tagNames: ['h2'],
        contentDefault: '<b>H</b>', // 可以设置工具栏里按钮的内容
      },
      {
        name: 'bold',
        aria: '加粗'
      },
      {
        name: 'anchor',
        aria: '链接',
        tagNames: ['a'],
        // 是否要对添加的超链接进行验证
        linkValidation: true,
        // 可配置超链接保存按钮的结构
        formSaveLabel: '<i class="icn-baocun"></i>',
        placeholderText: '请添加考拉内链接',
        // override 超链接的保存功能
        doFormSave: function () { 
          var opts = this.getFormOpts()
          opts.target = '_blank'
          opts.buttonClass = 'u-alink'
          if (opts.value) {
            this.completeFormSave(opts)
          }
        },
        // 根据业务，重载链接校验功能
        checkLinkFormat: function (val) { 
          // 自定义校验逻辑...
        }
      }
    ],
    // 是否要支持图片拖拽，如为true，往编辑器里拖一张图片，会转成img标签插入；默认为true
    imageDragging: false,
    // 是否将工具栏固定在编辑器容器的相对位置
    static: false,
    // 当static为trus时才有效，用来设置工具栏的对齐方式
    align: 'center', 
    // 当static为true时有效，设置工具栏在视窗固定位置且一直可见
    sticky: false,
    // 当static为true时有效，设置空选区（没有选区，只有光标）时是否更新工具栏按钮状态
    updateOnEmptySelection: false
  },
  // hover超链接时的预览框配置
  anchorPreview: { 
    // 空链接(href无值)是否要预览
    showOnEmptyLinks: false,
    // 重写链接点击事件，直接打开该链接地址
    handleClick: function (event) { 
      let anchorExtension = this.base.getExtensionByName('anchor')
      let activeAnchor = this.activeAnchor

      if (anchorExtension && activeAnchor) {
        event.preventDefault()
        let elm = event.target
        let link = elm.href
        if (elm.tagName == 'A' && link) {
          window.open(link, '_blank')
        }
      }
      this.hidePreview()
    }
  },
  // 设置文件拖拽
  fileDragging: {
    // 设置允许拖拽的文件类型
    allowedTypes: [] 
  },
  // 编辑器的占位文案，设置是否要在点击时隐藏等
  placeholder: {
    text: '请输入内容...',
    hideOnClick: false
  },
  // 设置自定义的扩展，比如粘贴扩展；如果不自定义，也可以设置一些粘贴的可配置项，详情可查阅：(https://github.com/yabwe/medium-editor#paste-options)
  extensions: {
    'paste': new PasteExtension()
  }
}

```

- ### 重要API

`MediumEditor`提供的接口也很全面，能满足常规的开发使用。完整的API可查阅[github](https://github.com/yabwe/medium-editor/blob/master/API.md)上的介绍。挑选几个重要的API来介绍下：

**初始化类**

- new MediumEditor(elements, options) 初始化
- destroy() 回收函数，会删除所有生成的元素、属性，解绑通过编辑器添加的事件和自定义事件
- setup() 如果实例被回收了，可以根据实例化过的的元素和option来重新初始化



**事件类**

- on(targets, event, listener, useCapture)
- off(targets, event, listener, useCapture)

给目标元素绑定和解绑浏览器事件，内部基于`addEventListener`和`removeEventListener`。如：

```
this.editor.on(editorElm, 'keyup', _.debounce(this.onSelect, 100))
this.editor.on(editorElm, 'click', this.onEditorClik)
this.editor.off(editorElm, 'click', this.onEditorClik)
```


- subscribe(name, listener)
- unsubscribe(name, listener)

订阅和解除订阅自定义事件，如：

```
// 订阅内置的editableInput，监听编辑器内容的变化
this.editor.subscribe('editableInput', _.debounce(this.onEditorChange, 200))

```

**选区类**

- restoreSelection()：恢复由上一次`saveSelection()`保存的选区
- saveSelection()：保存当前选区
- selectAllContents()：选中当前focus的编辑器元素里的所有内容


**编辑器功能类**

- cleanPaste(text) ： 转化参数为纯文本内容并在当前选区中插入
- createLink(opts) ： 通过原生的document.execCommand('createLink')来创建链接
- execAction(action, opts) ：通过原生的document.execCommand来执行内建的命令
- pasteHTML(html, options)：在当前选区插入html内容

**帮助类**
- serialize()：将编辑里的所有内容元素进行序列化，返回json对象
- setContent(html, index)：设置编辑器内容，传入html结构


- ### 扩展功能extension

`MediumEditor`内置的插件可查看安装包里的[extension目录](https://github.com/yabwe/medium-editor/tree/master/src/js/extensions)，包括了超链接、超链接预览、粘贴等等。另外，还有一些自定义的可集成[扩展插件](https://github.com/yabwe/medium-editor/wiki/Extensions-Plugins)，比如图片上传和插入、表格插件、支持markdown的插件等等，只要在初始化的option参数里配置下就可以使用了。

如果以上的插件，还不满足你的业务场景，那么就来编写一款自定义的插件吧。`MediumEditor`提供了完备的接口来让开发者自己动手写插件。

```
var MyExtension = MediumEditor.Extension.extend({
  name: 'myextension'
});

var myExt = new MyExtension();

var editor = new MediumEditor('.editor', {
  extensions: {
    'myextension': myExt
  }
});

editor.getExtensionByName(`myextension`) === myExt //true

```

在实际的业务开发中，内置paste插件无法满足需求，就需要进行二次开发来适用于项目需求了。对于paste扩展，后续再细说。


## 总结

在项目中实际应用下来，**MediumEditor**确实是一款非常灵活好用的编辑器，无论是从直接使用还是二次扩展来说，开发体验都很棒！细读一下源码，会对富文本编辑器领域以及如何来设计一款好用的编辑器方面，将会有很大收获。


-----

【参考】
- [前端开发：一款开源且轻量级的行内工具栏编辑器（MediumEditor）](http://www.360doc.com/content/17/1006/19/47869400_692672684.shtml)


by lzf


