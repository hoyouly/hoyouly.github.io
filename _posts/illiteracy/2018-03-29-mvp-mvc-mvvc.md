---
layout: post
title: 扫盲系列之---MVP,MVC,MVVC 介绍
category: 扫盲
tags: MVP MVC MVVC
---
* content
{:toc}

# MVC
这个我还比较熟悉，或者说听说过，大概知道是什么意思，
MVC就是 Model（模型）-View（视图）-Controller（控制器）的缩写，一种软件设计典范，用一种业务逻辑，数据，界面显示分离的方法组织代码，在改进和定制化界面的同时，不需要重新编写业务逻辑。
* M 层 处理数据，业务逻辑
* V 层 处理界面显示结果
* C 层 起到桥梁作用，用来控制V 层和M 层通信已达到分离视图显示和业务逻辑层。

![](http://p5sfwb51p.bkt.clouddn.com/mvc.png)  


原理就是：当用户出发事件的时候，View层会发送指令到Controller层，接着Controller去通知Model层更新数据，Model层更新完数据以后直接显示在view层上，这就是MVC的工作原理。
所有通信都是单项的。
## Android中的MVC
* V 层 一般采用XML文件进行界面描述，这些XML文件可以理解为app的View,使用的时候也非常方便，同时便于后期修改，逻辑中与界面对应的id不变化，则不需要修改，大大增强了代码的可维护性。
* C 层 通常落在Activity上，不能在Activity中写代码，通过Activity交给 Model业务逻辑层处理，因为Activity响应时间是5s,
* M 层 我们针对业务模型，建立的的数据结构和相关的类，Model与View无关，而与业务相关，对数据库，网络等操作，通常都应该在Model中处理，当然对业务计算等操作也必须放在该层，


# MVP
在Android开发中，通常我们发现Activity中代码太多，因为View的控制能力太弱了，一些对View的操作不能在xml文件中去做，只能放到Activity中，所以Activity本身需要负责与用户的交互，界面展示，不单纯是一个Controller或者View，为了解决Activity臃肿的问题，引入的MVP 模式

MVP  Model（模型）-View（视图）-Presenter（主持人）是从MVC演变过来的，与MVC有一定的相似，Controller和Presenter负责业务逻辑处理，Model负责提供数据，View负责显示，但是却改变了通信方向

其核心就是 通过一个抽象的View接口（不是真正的View层）将Presenter与真正的View进行解耦。Presenter持有该View接口，对该接口进行操作，而不是直接操作View层，这样就可以把视图操作和业务逻辑解耦，从而让Activity称为真正的View层。

![](http://p5sfwb51p.bkt.clouddn.com/mvp.png)
## MVP和MVC区别
1. 各部分之间的通信都是双向的，但是MVC却是单项的。
2. View与Model不发生通信，都是通过Presenter传递，但是MVC中View是可以直接和Model通信的
3. View 非常薄，不部署任何逻辑，称为被动视图（Passive VIew），即没有任何主动性，而Presenter非常厚，所有逻辑都部署在那里。
4. 通常View和Presenter是一对一的，但是复杂的View也可以和多个Presenter进行绑定来处理逻辑，而Controller是基于行为的，并且可以和多个View共享，Controller可以负责决定显示哪个View
5. Presenter与View是通过接口来进行的，更有利于单元测试。

View 和Presenter并没有耦合到一块，对于VIew和Presenter的通信，我们可以通过接口实现，具体的意思就是我们的Activity或者Fragment可以去实现好定义的接口，而在Presenter中通过接口调用的方法，

## MVP的优点
1. View和Model完全隔离，我们可以修改View却不影响Model
2. 可以更高校使用Model，因为所有的交互都发送在一个地方--Presenter
3. 可以将一个Presenter应用与多个View，而不需要改变Presenter的逻辑，这个特性非常有用，因为View的变换总比Model频繁
4. 如果把逻辑放到Presenter中，那么就可以脱离用户接口来测试这些逻辑（单元测试）

## 具体到Android项目中
根据APP的结构 进行纵向划分：
* 模型层 （M）  包含着具体的数据请求，数据源。
* UI 层 （V） 一般包括Activity，Fragment，Adapter等直接和UI相关的类
* 逻辑层 （P）  为业务处理层，既能调用UI逻辑，又能请求数据，该层为纯Java类，不涉及任何Android API。

UI 层的Activity启动后实例化Presenter，然后APP的控制权后移交给Presenter，两者通过BroadCast,Handler或者接口 完成通信，只传递事件和 结果
三层之间调用顺序为view->presenter->model，为了调用安全着想不可反向调用！不可跨级调用！



## MVP 的变种 Passview View
MVP 变种很多，最常见的就是Passview View ,即被动视图。
这种模式：
View和Model直接不能直接交互
View通过Presenter与Model打交道，
Presenter接受View的UI请求，完成简单的UI处理逻辑，并调用Model 进行业务处理，并调用View将结果反应出来，
View直接以来Presenter，但是Presenter间接以来View，它直接以来的是View的实现接口

相对于被动的View，那Presenter就是主动一方：
* Presenter是整个MVP的控制中心，而不是单纯的处理View请求的人
* View 仅仅是用户交互请求的汇报着，对于响应用户交互的逻辑和流程，View不参与，真正的决策者是Presenter
* View想Presenter发送用户交互请求应该采用的口吻是：我现在将用户交互请求发送给你，你看着办，需要我的时候我会协助你的。而不是“我现在处理用户交互请求，我知道该怎么办，但是我需要你的支持，因为实现业务的逻辑Model只信任你”
* 对于绑定到View数据，不应该是View从Presenter上拉回来的，而是Presenter主动推给View的
* View 尽可能不维护数据状态，因为本身仅仅实现了单纯的，独立的UI操作，Presenter才是整个系统的协调者，它根据处理用于交互的逻辑给View和Model安排工作。

## MVP的缺点
由于使用的是接口方式连接View和Presenter，这样就导致一个问题，如果你有一个逻辑很复杂的页面，你的接口会很多，十几二十个不足为奇，想像一下，如果一个APP中有很多这样负责的界面，那么接口维护的成本就会非常高。
解决方案就是根据业务逻辑去斟酌写接口，也可以定义一个基础接口，把一些公共的逻辑，比如网络请求成功，失败，Toast等放到里面，之后定义新的接口继承这个基础接口，
* 以UI为驱动的模型，更新UI都需要保证能取到控件的引用，同时更新UI的时候要考虑当前是否是UI线程，也要考虑Activity的声明周期（是否已经销毁等）
* 以UI和事件为驱动的传统模型，数据都是被动的通过UI控件展示，但是由于数据的时变形，我们希望数据能转被动为主动，希望数据能更有活性，由数据来驱动UI，
* Presenter 层与V层通过接口进行交互，接口粒度不好控制，粒度太小，就会存在大量接口的情况，是代码过于碎版化，粒度太大，解耦效果不好，同时对UI的输入和数据的变化，需要手动调用V层或者P层相关的接口，相对来说缺乏自动性，监听性，如果数据的变换能自动相应UI，UI的输入能自动更新数据，那多好
* V层和P层还是有一定的耦合度的，一旦V层某个UI元素更改，那么对应的接口就必须得改，数据如何映射到UI上，事件监听接口这些都需要转变，牵一发而动全身，如果这一层也能解耦就更好了，
* 复杂的业务同时也可能会导致P层太大，代码臃肿的问题依然不能解决。


# MVVM
算是MVP的升级版吧，由于使用MVP会使Presenter中的代码逻辑变得臃肿，所以用ViewModel代替。ViewModel和View之间不是通信的关系而是绑定的关系。
Model（模型）-View（视图）-ViewModle(),这个ViewModel可以理解为View的数据模型和Presenter的合体，ViewModel和View直接的交互通过Data Binding完成，而Data Binding可以实现双向的交互 这就是视图和控制层的耦合进一步降低，关注点分离的更彻底，同时减轻Activity的压力。

MVVM 利用数据绑定（Data Binding） ,依赖属性（Dependency Property）,命令（Command），路由事件（Routed Event） 等新特性，打造一个更加灵活高效的架构

![](http://p5sfwb51p.bkt.clouddn.com/mvvm.png)

采用双向绑定(Data Binding) ,View的变动，自动反映在ViewModel，反之依然，
这种模式的关键就是 Data Binding，就是数据绑定，View的变换直接影响ViewModel，ViewModel的变换或者内容也会体现在View上，这种模式实际上就是框架应用开发者做了一些工作，开发者只需要少量的代码就能实现比较复杂的交互。
数据绑定 你可以认为是Observer模式或者是Public/Subscribe 模式，原理都是为了用一种集中的方式实现频繁需要被实现的数据更新问题。


# AOP
AOP Aspect-Oriented-Programming 面向切面编程，是对OOP （Object-Oriented-Programming）面向对象编程的补充和完善，
OOP 引入封装，继承和多态性等概念来建立一种对象层次结构，用以模拟功能行为的一个集合，
OOP 允许你定义从上到下的关系，但是并不适合从左到右的关系，例如日志功能，日志功能往往水平的散步在所有对象层次中，而与它所散步到的对象的核心功能毫无关系，对于其他类型的diam，如安全性，异常处理和透明的持续性也是如此，这种散布在各处的无关的代码被称为横切（Cross-Cuting）代码，在OOP设计中，它导致了大量代码重复，而且不利于各个模块的重用，而AOP技术恰恰相反，它利用一个号称为横切的技术，解剖开封装的对象内部，并将那些影响了多个类的 公共行为封装到一个可重用的模块，并将其名为 Aspect,即方面，
简单来说，就是将那些与业务无关，却为业务模块共同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于为了的可操作行和可维护性。

AOP 把软件系统分为两个部分:
* 核心关注点  业务处理的主要流程
* 横切关注点  与之关系不大的部分 。例如HTTP，SharedPreference，JSON，XML，File，Device，System，Log，格式转换等。

横切关注点的一个特点是，经常发生在核心关注点的多出，而各处都是基本相似，AOP的作用就在于分离系统中的各种关注点，将核心关注点和横切关注点分离开来。一般的APP工程中的应该有一个Util 包，存放相关的切面操作，

在使用MVP和AOP对APP进行纵向和横向的切割之后，能够使得APP整体的结构更清晰合理，避免局部的diam臃肿，方便开发，测试以及后续的维护。

# 整体架构
UI层内部多用模板方法，以Activity为例，一般有BaseActivity,提供包括一些基础样式，Dialog，ActionBar等内容，展现的Activity都会继承BaseActivity，并且实现预留的接口，Activity之间的继承不能超过3层，
为了避免Activity内代码过多，将APP的整体控制权后移，也借鉴IOC做法，大量的逻辑操作放到逻辑层，逻辑层和UI层通过接口或者Broadcast等实现通信，只传递结果，

逻辑层实现的绝大部分的逻辑操作。由UI层启动，在内部通过接口调用模型层的方法，在逻辑层内大量使用了代理，打个比方，UI层告诉逻辑层我需要做的事，逻辑层去找相应的人（模型层）去做，最后只告诉UI这件事的结果

模型层没声母好说的，这部分一般由大量的Package组成，代码量是三层中最大的，需要在内部进行分层

横向的分割线根据AOP面向切面的思想，主要提取公用方法作为一个单独的Util，这写Util会在APP中穿插使用，这样横纵两次对APP代码的切割，已经能使程序不会过多的堆积在一个java文件里面，


---
搬运地址：  
[MVC，MVP 和 MVVM 的图示](http://www.ruanyifeng.com/blog/2015/02/mvcmvp_mvvm.html)  
[Android App的设计架构：MVC,MVP,MVVM与架构经验谈](https://www.tianmaying.com/tutorial/AndroidMVC)  
[Android App的设计模式概述：MVC、MVP、MVVM](https://www.jianshu.com/p/effad2e593df)  
[选择恐惧症的福音！教你认清MVC，MVP和MVVM](http://zjutkz.net/2016/04/13/%E9%80%89%E6%8B%A9%E6%81%90%E6%83%A7%E7%97%87%E7%9A%84%E7%A6%8F%E9%9F%B3%EF%BC%81%E6%95%99%E4%BD%A0%E8%AE%A4%E6%B8%85MVC%EF%BC%8CMVP%E5%92%8CMVVM/)  
[如何构建Android MVVM 应用框架](https://tech.meituan.com/android_mvvm.html)
