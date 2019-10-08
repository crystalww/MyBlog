---
title: HTML基础
date: 2018-10-25 20:54:05
categories:
- 网页相关
tags:
- HTML
---

## HTML文件基本结构
html中的标签一般都是成对出现的，分开始标签和结束标签，标签由英文尖括号`<`和`>`括起来。
```HTML
<html>
    <!-- <html></html>称为根标签，所有的网页标签都在<html></html>中 -->
    <head>
        <!-- 定义文档的头部，头部元素有<title>、<script>、<style>、<link>、<meta> -->
    </head>
    <body>
        <!-- 网页的主要内容，如<h1>、<p>、<a>、<img>等 -->
    </body>
</html>
```
<!-- more -->
## 常用标签
HTML语义化：根据内容的结构化（内容语义化），选择合适的标签（代码语义化），便于开发者阅读和写出更优雅的代码的同时，让浏览器的爬虫和机器很好的解析。

`<hx>`：标题标签一共有6个，并且依据重要性递减。`<h1>`为一级标题，是最高的等级。`<h6>`为六级标题。<br>
`<h1>标题文本</h1>`<br>
`<em>`：表示强调，默认用斜体表示。<br>
`<strong>`：表示更强烈的强调，用粗体表示。<br>
`<span>`：没有语义，作用是为了设置单独的样式。<br>
`<q>`：短文本引用，浏览器会对q标签自动添加双引号。<br>
`<blockquote>`：长文本引用。<br>
`<br />`：回车换行。<br>
`&nbsp;`：空格。<br>
`<hr>`：添加水平横线。<br>
`<address>`：地址信息。<br>
`<code>`：单行代码。如果是多行代码，可以使用`<pre>`标签。<br>
`<pre>`：被包围在pre元素中的文本通常会保留空格和换行符。<br>

无序列表
```
<ul>
  <li>信息</li>
   ...
</ul>
```

有序列表
```
<ol>
   <li>零基础学习html</li>
   ...
</ol>
```

`<div>`：相当于一个容器，存放独立的逻辑部分。用id属性为`<div>`提供唯一的名称。

`<a>`标签作用：
- 实现超链接
- 链接email地址，使用mailto能让访问者便捷向网站管理者发送电子邮件。如果mailto后面同时有多个参数的话，第一个参数必须以“?”开头，后面的参数每一个都以“&”分隔。
```HTML
<a href="目标网址" title="鼠标滑过显示的文本">链接显示的文本</a>

<!-- 在新的浏览器窗口中打开网页 -->
<a href="目标网址" target="_blank">click here!</a>

<!-- 链接email地址 -->
<a href="mailto:yy@xxx.com?subject=主题&body=内容。">发送邮件给我</a>
```
`<img>`标签：插入图片（可以是GIF，PNG，JPEG格式）。
- src标识图像的位置；
- alt指定图像的描述性文本，当图像不可见时（下载不成功时），可看到该属性指定的文本；
- title提供在图像可见时对图像的描述(鼠标滑过图片时显示的文本)；
```
<img src = "myimage.gif" alt = "My Image" title = "My Image" />
```

创建表格:
- table表格在没有添加css样式之前，在浏览器中显示是没有表格线的
- `<tbody>…</tbody>`：如果不加`<thead><tbody><tfooter>`,table表格加载完后才显示。加上这些表格结构，tbody包含行的内容下载完优先显示，不必等待表格结束后再显示。
- `<tr>…</tr>`：表格的一行，所以有几对tr，表格就有几行。
- `<td>…</td>`：表格的一个单元格，一行中包含几对<td>...</td>，说明一行中就有几列。
- `<th>…</th>`：表格表头，th标签中的文本默认为粗体居中显示。
- 表格中列的个数，取决于一行中数据单元格的个数。
```
<table>
  <tbody>
    <tr>
      <th>班级</th>
      <th>学生数</th>
      <th>平均成绩</th>
    </tr>
    <tr>
      <td>一班</td>
      <td>30</td>
      <td>89</td>
    </tr>
    <tr>
      <td>二班</td>
      <td>35</td>
      <td>85</td>
    </tr>
  </tbody>
```

为表格加入边框
```
<style type="text/css">
table tr td,th{border:1px solid #000;}
</style>
```

为表格添加标题和摘要
- 摘要的内容是不会在浏览器中显示出来的。它的作用是增加表格的可读性(语义化)，使搜索引擎更好的读懂表格内容，还可以使屏幕阅读器更好的帮助特殊用户读取表格内容。
- 标题用以描述表格内容，标题的显示位置：表格上方。
```
<table summary="本表格记录2012年到2013年库存记录，记录包括U盘和耳机库存量">
  <caption>2012年到2013年库存记录</caption>
  <tr>
    <th>产品名称 </th>
    <th>品牌 </th>
    <th>库存量（个） </th>
    <th>入库时间 </th>
  </tr>
  <tr>
    <td>耳机 </td>
    <td>联想 </td>
    <td>500</td>
    <td>2013-1-2</td>
  </tr>
  <tr>
    <td>U盘 </td>
    <td>金士顿 </td>
    <td>120</td>
    <td>2013-8-10</td>
  </tr>
</table>
```

使用表单标签与用户交互
- 所有表单控件（文本框、文本域、按钮、单选框、复选框等）都必须放在 <form></form> 标签之间，否则用户输入的信息提交不到服务器上。
- method ： 数据传送的方式（get/post）。
- action ：浏览者输入的数据被传送到的地方,比如一个PHP页面(save.php)。
```
<form method="post" action="save.php">
    <label for="username">用户名:</label>
    <input type="text" name="username" />
    <label for="pass">密码:</label>
    <input type="password" name="pass" />
</form>
```

文本输入框
- 当type="text"时，输入框为文本输入框；当type="password"时, 输入框为密码输入框。
- name：为文本框命名
- value：为文本输入框设置默认值。(一般起到提示作用)
```
<input type="text/password" name="名称" value="文本" />
```

文本域：支持多行文本输入。在css中col可用width、row用height来代替。<br>
```
<textarea  rows="行数" cols="列数">文本</textarea>
```

单选/复选框
- 当type="radio"时，控件为单选框；当type="checkbox"时，控件为复选框。
- value：提交数据到服务器的值。
- name：为控件命名
- 当设置 checked="checked" 时，该选项被默认选中
- 同一组的单选按钮，name 取值一定要一致，这样同一组的单选按钮才可以起到单选的作用。
```
<input type="radio/checkbox" value="值" name="名称" checked="checked"/>
```

下拉列表框
- 设置selected="selected"属性，则该选项被默认选中。
- 在`<select>`标签中设置multiple="multiple"属性可以实现多选功能。
- 在windows下进行多选时按下Ctrl键同时进行单击（在Mac下使用Command+单击）可以选择多个选项。
```
<form action="save.php" method="post" >
    <label>爱好:</label>
    <select>
      <option value="向服务器提交的值">显示的值</option>
      <option value="旅游">旅游</option>
      <option value="运动">运动</option>
      <option value="购物" selected="selected">购物</option>
    </select>
</form>
```

提交按钮：提交表单信息到服务器。<br>
`<input   type="submit"   value="提交">`

重置按钮：重置表单信息。<br>
`<input type="reset" value="重置">`

form表单中的label标签
- 不会向用户呈现任何特殊效果
- 作用是为鼠标用户改进可用性。当用户单击选中该label标签时，自动选中和该label标签相关连的表单控件。
`<label for="控件id名称">`
```
<form>
   <label for="male">男</label>
  <input type="radio" name="gender" id="male" />
  <br />
  <label for="female">女</label>
  <input type="radio" name="gender" id="female" />
  <br />
  <label for="email">输入你的邮箱地址</label>
  <input type="email" id="email" placeholder="Enter email">
</form>
```