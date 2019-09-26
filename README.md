# notes
work as my notebook
grammar:
#### chrome升到62%时碰到0x80040902号错误：

按照方法测试了一下：

1、在windows10下运行cmd

2、键入如下命令：

taskkill /im chrome.exe /f
 taskkill /im googleupdate.exe /f
 taskkill /im google*.exe /fi “STATUS eq RUNNING” /f
 taskkill /im google*.exe /fi “STATUS eq UNKNOWN” /f
 taskkill /im google*.exe /fi “STATUS eq NOT RESPONDING” /f

3、在运行时或报错，忽略，如果开着chrome，运行完以上命令后会自动关闭

4、等待一会，再次运行chrome

5、在url栏中输入“chrome://settings/help”*这是倾斜的文字*`

***这是斜体加粗的文字***
~~这是加删除线的文字~~

>这是引用的内容
>>这是引用的内容
>>
>>>>>>>>>>这是引用的内容

---
----
***
*****

图片：
![图片alt](图片地址 ''图片title'')

图片alt就是显示在图片下面的文字，相当于对图片内容的解释。
图片title是图片的标题，当鼠标移到图片上时显示的内容。title可加可不加

示例：
![blockchain](https://ss0.bdstatic.com/70cFvHSh_Q1YnxGkpoWK1HF6hhy/it/
u=702257389,1274025419&fm=27&gp=0.jpg "区块链")

[超链接名](超链接地址 "超链接title")
title可加可不加
[简书](http://jianshu.com)
[百度](http://baidu.com)

<a href="超链接地址" target="_blank">超链接名</a>

示例
<a href="https://www.jianshu.com/u/1f5ac0cf6a8b" target="_blank">简书</a>

列表：
- 列表内容
+ 列表内容
* 列表内容

注意：- + * 跟内容之间都要有一个空格

1.列表内容
2.列表内容
3.列表内容
列表嵌套
上一级和下一级之间敲三个空格即可

第二行分割表头和内容。
- 有一个就行，为了对齐，多加了几个
文字默认居左
-两边加：表示文字居中
-右边加：表示文字居右
注：原生的语法两边都要用 | 包起来。

流程：
```flow
st=>start: 开始
op=>operation: My Operation
cond=>condition: Yes or No?
e=>end
st->op->cond
cond(yes)->e
cond(no)->op
&```

```