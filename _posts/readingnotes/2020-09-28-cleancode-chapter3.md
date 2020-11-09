---
layout: post
title: 代码整洁之道 - 函数
category: 读书笔记
tags: 代码整洁之道
---
<!-- * content -->
<!-- {:toc} -->

这一章主要是说对函数的整洁

简单概况一下，一个整洁的函数包含以下几点
* 一个好函数名
* 只做一件事，并且做好这件事
* 短小精悍
* 参数尽量少
* 避免出现输出参数
* 避免重复代码

## 一个好函数名
好名称的标准：长而具有描述性的名称，要比短而令人费解的名称好，长而具有描述性的名称，要比描述性的长注释好。

使用某种命名规范，让函数名称的多个单次容易阅读。

一个好的函数名，应该能解释函数的意图，以及参数的顺序和意图

例如一元函数：函数和参数应该形成一种非常良好的动词/名称对形式。例如writeField(name),它就明确告诉我们，name是一个field,

## 只做一件事，并且做好这件事
判断是否做一件事一个方法就是看能否再拆出来一个函数。该函数不仅仅是单纯的重新诠释其实现

可以从以下几点入手
* 分割指令与询问   
函数要么做什么事，要是回答什么事，不能二者兼得。函数要么修改对象的状态，或者返回对象的有关信息，两样都干会导致混乱。
* 无副作用   
在一个函数中，做了两件事，但是名字上并没有体现出来，所以就导致可能出现副作用。
* 尽量避免标识参数   
  向函数中传入boolean值就是骇人听闻的，这样做，方法签名就变得复杂了。并且也说明了函数不止做一件事，true是这样做，false是那样做。

## 短小精悍
短小的好处就是一目了然。函数越短小，功能越集中，就越能取于一个好名称。建议函数长度20行封顶

可以从下面几点入手

* if ,else语句，while语句，代码块只有一行，这行大抵是一个函数调用语句。   
这样不但保持函数短小，热情因为块内调用的函数具有较明确的说明名称，增加了文档上的价值
* 函数的缩进层级不应该多余两层，就是不应该内嵌两个{}，最多嵌套一个
* 确保每个switch语句都处于在较低的抽象层级，并且永远不重复   
可以使用抽象工厂解决switch语句的问题。创建多态对象，并且隐藏在某个继承关系中。在系统其它部分看不到。
* 使用异常替代返回错误码，这样可以减少深层次的嵌套
  * 把try/catch代码块的主体抽离出来，另外形成函数。
  * 错误处理也是一件事。一个函数中不能即处理正确的事情，又处理错误的事情。如果关键字try在某个函数中存在，它就该是这个函数的第一个单词，而且在catch/finally代码块后面不该有其他内容。

## 参数尽量少
零个最优，其次是一个，再次是两个，尽量避免三个，有足够特殊的理由才能用三个以上参数   

因为参数不易理解，带有太多概念性。一个参数扫一眼就明白了，第二个参数就得思考一下才能明白。如果出现三个或者以上，就说明其中一些参数该进行封装了。

## 避免出现输出参数    
因为我们习惯信息通过参数输入函数，通过返回值输出。而不希望通过参数输出。如果非要对输入参数进行转换的，转换结果体现到返回值上

## 避免重复代码
即DRY原则。
  * 重复可能是软件中一切邪恶的根源。许多原则与实践都是为了控制与消除重复而创建的。
  * 自子程序发明以来，软件开发领域的所有创新都是在不断尝试从源代码中消灭重复
  * 遇到重复就开始封装吧

## 如何写这样函数
* 打磨代码
* 分解函数
* 修改名称
* 消除重复
* 缩短和重新安置方法
* 拆散类

每个函数，函数中的代码块，都应该有一个入口，一个出口，简单来说，就是函数只有一个return语句，循环中不能出现break或者continue，并且禁止出现goto语句。如果函数保持较小，偶尔出现return，break，continue，也不是可以的。

如果每个例子都让你深合己意，那就是整洁代码

---
搬运地址：    

代码整洁之道