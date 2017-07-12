---
title: Web表单
date: 2017-06-30
---

<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如：by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->

## Web表单

Web表单是前端开发领域十分重要的一块内容。本文将针对Web表单相关的各方面内容进行介绍。

<!-- more -->

### 表单验证

使用web应用或浏览页面时，经常会遇到的场景，比如用户注册，填写点评反馈等，需要用户按照一定的要求来填写数据信息。典型的会伴随如下提醒：

- 请填写合法的邮箱地址或手机号码
- 设置的密码必要包含一个大写字母、数字和特殊符号

这些就是所谓的 **表单验证** - 即对用户输入的数据信息进行一定规则的验证。最终输入的信息符合正确的规则，才会被允许提交到服务器端。

表单验证一定程度上会导致页面交互变得复杂，但又是必不可少的一个环节，主要有以下考虑：

- 按照预期的格式获取预期的数据

  对于web应用，不可能考虑到所有场景，所以通常来说，开发人员总是会对数据的格式有一定的要求；
  
- 保证用户数据的安全

  典型的对于用户过于简单的密码设置，甚至不设置密码，这样很容易导致用户的账户信息泄露；

- 保证web应用的安全

  可能会有居心不良者，通过非法的表单数据对web应用进行攻击破坏；

考虑到上述表单验证的必要性，开发人员能做的就是尽量使得表单验证变得简单，从而减少用户的流失。

一般来讲，表单验证既可以在客户端表单提交前进行，也可以在服务器端表单提交后进行，现实开发时通常是客户端和服务器端同时对表单进行验证。本部分内容主要针对前端页面的表单验证。

对于现代浏览器的表单，HTML5针对表单数据验证提供了便利的特性。基于这些特性，我们能很方便地完成表单的验证。

HTML5对应表单验证的部分主要基于表单元素的[validation属性](https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/HTML5/Constraint_validation)，这些属性指定了相应表单元素的验证规则。主要包括：

<table>
  <thead>
    <tr>
      <td>属性名</td>
      <td>支持该属性的表单元素</td>
      <td>可能的值</td>
      <td>描述</td>
      <td>不满足规则的提示</td>
    </tr>
  <thead>
  <tbody>
    <tr>
      <td>pattern</td>
      <td>text, search, url, tel, email, password</td>
      <td>JavaScript正则表达式</td>
      <td>元素的值必须匹配该值的模式</td>
      <td><b>Parttern mismatch</b> constraint violation</td>
    </tr>
    <tr>
      <td rowspan="3">min</td>
      <td>range, number</td>
      <td>合法的数字</td>
      <td rowspan="3">元素的值必须小于等于该值</td>
      <td rowspan="3"><b>Underflow</b> constraint violation</td>
    </tr>
    <tr>
      <td>date, month, week</td>
      <td>合法的日期</td>
    </tr>
    <tr>
      <td>datetime, datetime-local, time</td>
      <td>合法的日期和时间</td>
    </tr>
    <tr>
      <td rowspan="3">max</td>
      <td>range, number</td>
      <td>合法的数字</td>
      <td rowspan="3">元素的值必须大于等于该值</td>
      <td rowspan="3"><b>Overflow</b> constraint violation</td>
    </tr>
    <tr>
      <td>date, month, week</td>
      <td>合法的日期</td>
    </tr>
    <tr>
      <td>datetime, datetime-local, time</td>
      <td>合法的日期和时间</td>
    </tr>
    <tr>
      <td>required </td>
      <td>text, search, url, tel, email, password, date, datetime, datetime-local, month, week, time, number, checkbox, radio, file; also on the <select> and <textarea> elements</td>
      <td>值存在表示true, 不存在代表false</td>
      <td>元素必须设置值</td>
      <td><b>Missing</b> constraint violation</td>
    </tr>
    <tr>
      <td rowspan="5">max</td>
      <td>date</td>
      <td>整型的天数</td>
      <td rowspan="5">元素的值必须是min + 该值的整数倍</td>
      <td rowspan="5"><b>Step mimatch</b> constraint violation</td>
    </tr>
    <tr>
      <td>month</td>
      <td>整型的月数</td>
    </tr>
    <tr>
      <td>week</td>
      <td>整型的周数</td>
    </tr>
    <tr>
      <td>datetime, datetime-local, time</td>
      <td>整型的秒数</td>
    </tr>
    <tr>
      <td>range, number</td>
      <td>整数</td>
    </tr>
    <tr>
      <td>maxlength</td>
      <td>text, search, url, tel, email, password; also on the <textarea> element</td>
      <td>整数指定的长度</td>
      <td>元素中字符(code points)的个数不能超过该值</td>
      <td><b>Too long</b> constraint violation</td>
    </tr>
  </tbody>
</table>

如果表单元素满足所有指定的属性规则，则代表该表单元素验证合法，同时该元素会匹配 **:valid** 的伪类，此时默认浏览器会允许进行表单提交；否则代表该表单元素未通过验证，为非法状态，会匹配
 **:invalid** 的伪类，浏览器默认不允许表单提交。

#### [借助HTML5表单验证的实例](https://codepen.io/llwanghong/pen/bRvpbo)