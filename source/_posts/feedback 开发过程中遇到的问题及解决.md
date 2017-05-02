---
title: feedback 开发过程中遇到的问题及解决
date: 2017-03-22
---

## 由来:
后台系统多, 用户也多.用户在使用过程中对系统中不合理或影响工作效率的地方, 需要有一个反馈的渠道.

如果每个系统都做一个反馈收集页, 将是重复的工作, 会耗费大量的人力.

因此, 最好能有一个统一的东西, 插入到需要收集反馈的系统中.

<!-- more -->

## 术语:

`宿主系统`: 用户实际使用的系统;

`反馈系统`: 收集用户反馈并统一存放供开发人员调查的系统;

`插件脚本`: 由反馈系统对外提供的一个js文件, 由宿主系统在页面中引入.

## 需求描述:

宿主系统中的每个页面中, 要出现一个按钮, 点击按钮, 弹出一个弹窗, 用户在弹窗中编辑反馈信息. 反馈信息包括页面地址, 所属系统, 当前用户, 反馈的文本, 截图, 以及点赞或踩等. 其中所属系统, 页面地址和用户名, 不需要用户手动输入. 用户编辑完这些信息后, 点击确定, 统一提交到反馈系统, 然后弹窗自动关闭.

## 问题与解决方案

### 按钮嵌入宿主系统全部页面
向宿主系统的全部页面中嵌入一个按钮, 肯定不能手动编辑所有的页面文件, 鉴于系统中所有的页面都会有导航条()这一特点, 决定将添加按钮的逻辑写到一个js文件中, 即插件脚本. 将此js放到公用组件的节点中, 即可实现全部页面嵌入按钮. 至于这个js文件, 因为js支持跨域请求,  可存放在反馈系统中.这样做的好处是避免每个系统拷贝一份这个js文件的代码, 统一管理, 统一修改. 

### 系统间通信

如何将反馈信息在不同系统之间传送? 想到如下几种方案:

1.将反馈信息提交到宿主系统后端, 宿主系统再调用反馈系统的接口, 进行通信;

2.通过前端发送跨域请求, 直接将信息发送到反馈系统;

3.将弹窗中加入一个反馈系统的iframe, 用户在iframe中编辑信息, 直接提交;

第一种方案的缺点是需要每个系统添加后端通信的逻辑, 工作量大. 如果反馈系统的通信逻辑以后有改动, 每个系统都要改;

第二个方案的缺点是需要有人去配置nginx的跨域信息, 比较麻烦. 而且表单校验, 发送数据以及一些弹窗校验等工作, 放在一个需要没有依赖的js中去完成, 相当于裸着写逻辑, 代码量将会很大, 而且不好维护. 

第三种方案相当于打开了一个反馈系统的页面, 只是看起来是一个宿主系统的弹窗. 遇到的问题是, 用户提交完成后iframe向外部传送信息比较麻烦. 单这个比起上面两种方案来说, 更加容易解决. 

因此选择第三种方案.

### modal模态弹窗

一下是比较常见的弹窗使用场景和需求:

1.用户在提交反馈信息时, 如果有必须要填的字段没有填, 比如反馈的问题类型没选, 需要中断提交并提醒用户.

2.管理员在删除一个反馈问题时, 需要二次确认. 

3.不同的弹窗, 内容可能会不一样, 但外框和遮罩层的样式以及打开和关闭的逻辑在一般情况下是一样的, 因此重复的逻辑和样式需要可继承; 

4.用于二次确认的弹窗, 需要知道用户点击的是确定还是取消. 

 在regular中, 继承和组合是不冲突的, 即可以在继承父组件的同时, 只重写父组件中的slot部分, 然后塞到父组件的模板中, 并在使用中也可以轻松的拿到new出来的弹窗引用, 如下:
 ```
 //父组件 baseModal.js
 define([
    'pro/baseComponent',
    'text./modalTemplate.html'
 ], function(base, template) {
     return base.extend({
         template: template
         //...
     });
 });
 
 //父组件模板 modalTemplate.html
 <div class="mask">
    <div class="head">{title}</div>
    <div class="body">{slot}</div>
    <div class="footer">
        <button>Confirm</button>
        <button>Cancel</button>
    </div>
 </div>
 
 
 //子组件
 define([
    'pro/baseModal',
    'text!./slot.html'
 ], function(base, tpl) {
     return base.extend({
         content: tpl
         //...
     })
 })
 
 //使用时 main.js
 
 //...
 var modal = new Modal({
     //...
 });
 modal.$on('confirm', someFun);
 modal.$on('cancel', someFun);
 //...
 ```
 
 但是在vue中, 子组件继承父组件后, 并不继承父组件的模板, 即template需要完全重新写. 这样的拷贝背离了我们继承的初衷, 因此只能使用组合, 即所谓的子组件, 并不是继承了父组件, 而是在模板中组合进一个父组件, 然后将自己的模板塞到父组件的slot中. 
 vue推荐将modal事先写好在template中, 用的时候设置一个flag使其显示出来, 作者解释说之所以这样, 是为了时模板看起来清晰明了, 详见[vue作者的解释](https://www.zhihu.com/question/35820643)
 但这样产生一个问题, 直接将使用了slot的modal写到模板中, 无法拿到起引用, 即下面这样写是无效的.具体原因官方文档的解释并没有看懂.
 ```
 <div>
    <modal ref="xxx"></modal>
 </div>
 ```
 既然拿不到引用, modal关闭时用户点击的是确定还是取消, 自然不知道了. 而且在使用时设置显示modal的逻辑比较大同小异, 因此将控制弹窗打开关闭的逻辑抽出来放到mixin中. 
 至于弹窗关闭的事件, 刚开始做的时候自己也没想到好的解决方法, 参考了一下饿了么的弹窗组件, 他们的做法是, 在打开弹窗的方法中return一个promise, 将promise的resolve和reject方法包装到一个对象中压一个任务栈中, 当组合的父弹窗发出Confirm或cancel事件时, 从栈中弹出最上面的任务, 如果confirm, 则执行任务对象的resolve方法.
 代码如下:
 
 ```
 //父组件 modal.vue
 <style>
    //...
 </style>
 <template>
    <div class="mask">
        <div class="head">{title}</div>
        <div class="body">
            <slot></slot>
        </div>
        <div class="footer">
        <button @click="confirm">Confirm</button>
        <button @click="cancel">Cancel</button>
        </div>
    </div>
 </template>
 <script>
    import xxx
    
    export default {
        //xxx
    }
 </script>
 
 //mixin modal.js
module.exports = {
    data() {
        return {
            modalConfig: {
                status: 'modal-info',
                title: '提示',
                show: false,
                onlyClose: false
            },
            questionQueue: []
        };
    },
    methods: {
        /**
         *
         * @param msg 文本内容
         * @param status modal-warning modal-success modal-error modal-info
         */
        showWithOnlyCloseBtn(msg, status) {
            this.modalConfig = {
                show: true,
                status: status,
                title: '提示',
                content: msg,
                onlyClose: true
            };
        },
        showWarning(msg) {
            this.showWithOnlyCloseBtn(msg, 'modal-warning');
        },
        showError(msg) {
            this.showWithOnlyCloseBtn(msg, 'modal-danger');
        },
        showInfo(msg) {
            this.showWithOnlyCloseBtn(msg, 'modal-info');
        },
        showSuccess(msg) {
            this.showWithOnlyCloseBtn(msg, 'modal-success');
        },
        ask4Sure(msg) {
            this.modalConfig = {
                show: true,
                status: 'modal-info',
                title: '提示',
                content: msg,
                onlyClose: true
            };
            return new Promise((resolve, reject) => {
                this.questionQueue.push({
                    confirm: resolve,
                    cancel: reject
                });
            });
        },
        onConfirm() {
            this.destroy();
            if (!this.questionQueue.length) return;
            let task = this.questionQueue.shift();
            task.confirm();
        },
        onCancel() {
            this.destroy();
            if (!this.questionQueue.length) return;
            let task = this.questionQueue.shift();
            task.cancel();
        },
        destroy() {
            this.modalConfig.show = false;
        }
    }
};

 
 //子组件 subModal.vue
  <style>
    //...
 </style>
 <template>
    <modal @cancel="cancel" @confirm="confirm">
        <div>xxxxxx</div>
    </modal>
 </template>
 <script>
    import Modal from './modal.vue';
    
    export default {
        components: {
            modal
        },
        methods: {
            cancel() {
                this.$emit('cancel');
            },
            confirm() {
                this.$emit('confirm');
            }
        }
    }
    
 </script>
 
 //使用时 main.js
 //...
 methods: {
    onRemoveSystem(system) {
        this.ask4Sure( '确定删除此系统吗?')
            .then(() => {
                this.removeSystem(system);
            })
            .catch(() => {
                console.info('取消删除');
            });
    },
},
 //...
 
 ```
 ###自动获取当前环境信息
 
 一般来说, 每个系统的导航组件上都会有一句欢迎语, xx用户, 你好blabla... 于是, 只要将用户名所在的标签加上一个id, 再将这个id放到插件脚本的dataset中, 插件脚本就会在执行时获取到用户名. 然后在打开iframe时, 将获取到的用户名, 系统一名等信息以参数的方式放到iframe的src中, 这样反馈系统即可从location.search中获取到这些信息.
 
 ### 提交反馈后自动关闭弹窗
 
 反馈的提交动作是在iframe中触发的. 不同源的iframe无法向其父window发送信息.
 
 在网上查到一种比较繁琐的方式, 在子iframe中再嵌入一个与父容器同源的iframe, 由这个第三者iframe充当中转. 考虑到这种方式也比较复杂, 于是再想其他的解决方案. 
 
 iframe有一个onload事件可以执行父容器传入的回调. 可不可以将要发送的信息放在iframe的src中然后通过onload来传输呢, 这样不就可以实现父容器知道用户何时提交了么.
 
 只要在用户提交信息后, 将iframe跳转到另一个页面, 就可以触发onload了, 然后把要传出来的信息放到页面url中, 父容器通过event对象取获取, 一切大功告成.
 
 然而经过实验, 在iframe跳转页面时, 的确触发了onload. 但是event对象中的src还是新建iframe时传入的src, 不会更新成新的src.
 
 再回来结合业务场景思考, iframe在触发第一次onload时, 一定是刚新建好弹窗, 而第二次, 一定是用户提交成功后的跳转. 再往下就是我们将iframe隐藏起来并跳转会返回编辑页等待用户下一次打开"弹窗"提交反馈.
 
 简而言之, 奇数次的onload是跳转到反馈编辑页, 偶数次的onload是用户提交了反馈信息. 于是, 在偶数次触发onload的时候, 在插件脚本中将iframe隐藏起来, 即达到了用户提交反馈后自动关闭弹窗的效果.
 
 
 ## 后记
 
 目前的feedback还只是实现一些基本功能的状态, 很多细节没有覆盖到, 如果后期增加业务, 可能会遇到更多问题. to be continued...