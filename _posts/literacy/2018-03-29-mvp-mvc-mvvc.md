---
layout: post
title: 扫盲系列 - MVP, MVC , MVVC 介绍
category: 扫盲系列
tags: MVP MVC MVVC
---
<!-- * content -->
<!-- {:toc} -->

# MVC
是一个框架模式，而非设计模式， GOF 把 MVC 看作是三个设计模式：观察者模式，策略模式和组合模式的合体，其中核心在观察者模式。也就是基于发布/订阅者模型的框架。

## 框架模式和设计模式的区别
* 框架面向于一系列相同行为代码的重用。是大智慧，用来对软件设计进行分工。
* 设计面向于一系列相同结构代码的重用。是小技巧，对具体问题提出解决方案。以提高代码复用率，降低耦合度。

架构介于框架和设计之间。
软件开发中三种级别的重用：
1. 内部重用： 即在同一应用中能公共使用的抽象块
2. 代码重用： 通用模块组合成库或工具集，以便在多应用和领域都能使用
3. 应用框架的重用：专用领用提供通用的或者现成的基础结构，已获得最后级别的重用性。

这个我还比较熟悉，或者说听说过，大概知道是什么意思，
MVC 就是 Model（模型）-View（视图）-Controller（控制器）的缩写，一种软件设计典范，用一种业务逻辑，数据，界面显示分离的方法组织代码，在改进和定制化界面的同时，不需要重新编写业务逻辑。
* M 层 处理数据，业务逻辑
* V 层 处理界面显示结果
* C 层 起到桥梁作用，用来控制 V 层和 M 层通信已达到分离视图显示和业务逻辑层。

![](../../../../images/mvc.png)  

原理就是：当用户出发事件的时候， View 层会发送指令到 Controller 层，接着 Controller 去通知 Model 层更新数据， Model 层更新完数据以后直接显示在 view 层上，这就是 MVC 的工作原理。
所有通信都是单项的。
## Android中的MVC
* V 层 一般采用 XML 文件进行界面描述，这些 XML 文件可以理解为 app 的 View ,使用的时候也非常方便，同时便于后期修改，逻辑中与界面对应的 id 不变化，则不需要修改，大大增强了代码的可维护性。
* C 层 通常落在 Activity 上，不能在 Activity 中写代码，通过 Activity 交给 Model 业务逻辑层处理，因为 Activity 响应时间是 5s ,
* M 层 我们针对业务模型，建立的的数据结构和相关的类， Model 与 View 无关，而与业务相关，对数据库，网络等操作，通常都应该在 Model 中处理，当然对业务计算等操作也必须放在该层，

## 优点
1. 比较容易理解，技术含量不高，成本降低，也易于维护和修改
2. 耦合性不高，表现层和业务层分离实现各司其职。
3. 比较适合于规模较大的项目。比如 Android 的 UI 系统框架。
## 缺点：
1. 没有明确的定义，完全理解 MVC 模式不容易
2. 内部原理复杂，使用需要精心设计，
3. Model和 View 严格分离，给调试带来一定的困难。
4. 由于将应用程序分为 3 部分，意味着同一个工程将包含比以前更多的文件。因此对于小规模项目， MVC 反而会带来更大的工作量和复杂性。

# MVP
在 Android 开发中，通常我们发现 Activity 中代码太多，因为 View 的控制能力太弱了，一些对 View 的操作不能在 xml 文件中去做，只能放到 Activity 中，所以 Activity 本身需要负责与用户的交互，界面展示，不单纯是一个 Controller 或者 View ，为了解决 Activity 臃肿的问题，引入的 MVP 模式

MVP  Model（模型）-View（视图）-Presenter（主持人）是从 MVC 演变过来的，与 MVC 有一定的相似， Controller 和 Presenter 负责业务逻辑处理， Model 负责提供数据， View 负责显示，但是却改变了通信方向

其核心就是 通过一个抽象的 View 接口（不是真正的 View 层）将 Presenter 与真正的 View 进行解耦。 Presenter 持有该 View 接口，对该接口进行操作，而不是直接操作 View 层，这样就可以把视图操作和业务逻辑解耦，从而让 Activity 称为真正的 View 层。

![](../../../../images/mvp.png)
## MVP和 MVC 区别
1. 各部分之间的通信都是双向的，但是 MVC 却是单项的。
2. View与 Model 不发生通信，都是通过 Presenter 传递，但是 MVC 中 View 是可以直接和 Model 通信的
3. View 非常薄，不部署任何逻辑，称为被动视图（Passive VIew），即没有任何主动性，而 Presenter 非常厚，所有逻辑都部署在那里。
4. 通常 View 和 Presenter 是一对一的，但是复杂的 View 也可以和多个 Presenter 进行绑定来处理逻辑，而 Controller 是基于行为的，并且可以和多个 View 共享， Controller 可以负责决定显示哪个View
5. Presenter与 View 是通过接口来进行的，更有利于单元测试。

View 和 Presenter 并没有耦合到一块，对于 VIew 和 Presenter 的通信，我们可以通过接口实现，具体的意思就是我们的 Activity 或者 Fragment 可以去实现好定义的接口，而在 Presenter 中通过接口调用的方法，

## MVP的优点
1. View和 Model 完全隔离，我们可以修改 View 却不影响Model
2. 可以更高校使用 Model ，因为所有的交互都发送在一个地方--Presenter
3. 可以将一个 Presenter 应用与多个 View ，而不需要改变 Presenter 的逻辑，这个特性非常有用，因为 View 的变换总比 Model 频繁
4. 如果把逻辑放到 Presenter 中，那么就可以脱离用户接口来测试这些逻辑（单元测试）

## 具体到 Android 项目中
根据 APP 的结构 进行纵向划分：
* 模型层 （M）  包含着具体的数据请求，数据源。
* UI 层 （V） 一般包括 Activity ， Fragment ， Adapter 等直接和 UI 相关的类
* 逻辑层 （P）  为业务处理层，既能调用 UI 逻辑，又能请求数据，该层为纯 Java 类，不涉及任何 Android API 。

UI 层的 Activity 启动后实例化 Presenter ，然后 APP 的控制权后移交给 Presenter ，两者通过 BroadCast , Handler 或者接口 完成通信，只传递事件和 结果
三层之间调用顺序为view->presenter->model，为了调用安全着想不可反向调用！不可跨级调用！



## MVP 的变种 Passview View
MVP 变种很多，最常见的就是 Passview View ,即被动视图。
这种模式：
View 和 Model 直接不能直接交互
View 通过 Presenter 与 Model 打交道，
Presenter 接受 View 的 UI 请求，完成简单的 UI 处理逻辑，并调用 Model 进行业务处理，并调用 View 将结果反应出来，
View 直接以来 Presenter ，但是 Presenter 间接以来 View ，它直接以来的是 View 的实现接口

相对于被动的 View ，那 Presenter 就是主动一方：
* Presenter是整个 MVP 的控制中心，而不是单纯的处理 View 请求的人
* View 仅仅是用户交互请求的汇报着，对于响应用户交互的逻辑和流程， View 不参与，真正的决策者是Presenter
* View想 Presenter 发送用户交互请求应该采用的口吻是：我现在将用户交互请求发送给你，你看着办，需要我的时候我会协助你的。而不是“我现在处理用户交互请求，我知道该怎么办，但是我需要你的支持，因为实现业务的逻辑 Model 只信任你”
* 对于绑定到 View 数据，不应该是 View 从 Presenter 上拉回来的，而是 Presenter 主动推给 View 的
* View 尽可能不维护数据状态，因为本身仅仅实现了单纯的，独立的 UI 操作， Presenter 才是整个系统的协调者，它根据处理用于交互的逻辑给 View 和 Model 安排工作。

## MVP的缺点
由于使用的是接口方式连接 View 和 Presenter ，这样就导致一个问题，如果你有一个逻辑很复杂的页面，你的接口会很多，十几二十个不足为奇，想像一下，如果一个 APP 中有很多这样负责的界面，那么接口维护的成本就会非常高。
解决方案就是根据业务逻辑去斟酌写接口，也可以定义一个基础接口，把一些公共的逻辑，比如网络请求成功，失败， Toast 等放到里面，之后定义新的接口继承这个基础接口，
* 以 UI 为驱动的模型，更新 UI 都需要保证能取到控件的引用，同时更新 UI 的时候要考虑当前是否是 UI 线程，也要考虑 Activity 的声明周期（是否已经销毁等）
* 以 UI 和事件为驱动的传统模型，数据都是被动的通过 UI 控件展示，但是由于数据的时变形，我们希望数据能转被动为主动，希望数据能更有活性，由数据来驱动 UI ，
* Presenter 层与 V 层通过接口进行交互，接口粒度不好控制，粒度太小，就会存在大量接口的情况，是代码过于碎版化，粒度太大，解耦效果不好，同时对 UI 的输入和数据的变化，需要手动调用 V 层或者 P 层相关的接口，相对来说缺乏自动性，监听性，如果数据的变换能自动相应 UI ， UI 的输入能自动更新数据，那多好
* V层和 P 层还是有一定的耦合度的，一旦 V 层某个 UI 元素更改，那么对应的接口就必须得改，数据如何映射到 UI 上，事件监听接口这些都需要转变，牵一发而动全身，如果这一层也能解耦就更好了，
* 复杂的业务同时也可能会导致 P 层太大，代码臃肿的问题依然不能解决。


# MVVM
算是 MVP 的升级版吧，由于使用 MVP 会使 Presenter 中的代码逻辑变得臃肿，所以用 ViewModel 代替。 ViewModel 和 View 之间不是通信的关系而是绑定的关系。
Model（模型）-View（视图）-ViewModle(),这个 ViewModel 可以理解为 View 的数据模型和 Presenter 的合体， ViewModel 和 View 直接的交互通过 Data Binding 完成，而 Data Binding 可以实现双向的交互 这就是视图和控制层的耦合进一步降低，关注点分离的更彻底，同时减轻 Activity 的压力。

MVVM 利用数据绑定（Data Binding） ,依赖属性（Dependency Property）,命令（Command），路由事件（Routed Event） 等新特性，打造一个更加灵活高效的架构

![](../../../../images/mvvm.png)

采用双向绑定(Data Binding) , View 的变动，自动反映在 ViewModel ，反之依然，
这种模式的关键就是 Data Binding ，就是数据绑定， View 的变换直接影响 ViewModel ， ViewModel 的变换或者内容也会体现在 View 上，这种模式实际上就是框架应用开发者做了一些工作，开发者只需要少量的代码就能实现比较复杂的交互。
数据绑定 你可以认为是 Observer 模式或者是Public/Subscribe 模式，原理都是为了用一种集中的方式实现频繁需要被实现的数据更新问题。

MVVM 模式有点像 ListView 与 Adapter ,数据集的关系，这个 Adapter 就是 ViewModel 角色，它与 View 进行了绑定，又与数据集进行了绑定。当书记集合发生变化事，调用 Adapter 的 notifyDataSetChanged 之后 View 就直接更新，他们直接没有直接的耦合，使得 ListView 变得更为灵活。

# AOP
AOP Aspect-Oriented-Programming 面向切面编程，是对OOP （Object-Oriented-Programming）面向对象编程的补充和完善，
OOP 引入封装，继承和多态性等概念来建立一种对象层次结构，用以模拟功能行为的一个集合，
OOP 允许你定义从上到下的关系，但是并不适合从左到右的关系，例如日志功能，日志功能往往水平的散步在所有对象层次中，而与它所散步到的对象的核心功能毫无关系，对于其他类型的 diam ，如安全性，异常处理和透明的持续性也是如此，这种散布在各处的无关的代码被称为横切（Cross-Cuting）代码，在 OOP 设计中，它导致了大量代码重复，而且不利于各个模块的重用，而 AOP 技术恰恰相反，它利用一个号称为横切的技术，解剖开封装的对象内部，并将那些影响了多个类的 公共行为封装到一个可重用的模块，并将其名为 Aspect ,即方面，
简单来说，就是将那些与业务无关，却为业务模块共同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于为了的可操作行和可维护性。

AOP 把软件系统分为两个部分:
* 核心关注点  业务处理的主要流程
* 横切关注点  与之关系不大的部分 。例如 HTTP ， SharedPreference ， JSON ， XML ， File ， Device ， System ， Log ，格式转换等。

横切关注点的一个特点是，经常发生在核心关注点的多出，而各处都是基本相似， AOP 的作用就在于分离系统中的各种关注点，将核心关注点和横切关注点分离开来。一般的 APP 工程中的应该有一个 Util 包，存放相关的切面操作，

在使用 MVP 和 AOP 对 APP 进行纵向和横向的切割之后，能够使得 APP 整体的结构更清晰合理，避免局部的 diam 臃肿，方便开发，测试以及后续的维护。

# 整体架构
UI 层内部多用模板方法，以 Activity 为例，一般有 BaseActivity ,提供包括一些基础样式， Dialog ， ActionBar 等内容，展现的 Activity 都会继承 BaseActivity ，并且实现预留的接口， Activity 之间的继承不能超过 3 层，
为了避免 Activity 内代码过多，将 APP 的整体控制权后移，也借鉴 IOC 做法，大量的逻辑操作放到逻辑层，逻辑层和 UI 层通过接口或者 Broadcast 等实现通信，只传递结果，

逻辑层实现的绝大部分的逻辑操作。由 UI 层启动，在内部通过接口调用模型层的方法，在逻辑层内大量使用了代理，打个比方， UI 层告诉逻辑层我需要做的事，逻辑层去找相应的人（模型层）去做，最后只告诉 UI 这件事的结果

模型层没声母好说的，这部分一般由大量的 Package 组成，代码量是三层中最大的，需要在内部进行分层

横向的分割线根据 AOP 面向切面的思想，主要提取公用方法作为一个单独的 Util ，这写 Util 会在 APP 中穿插使用，这样横纵两次对 APP 代码的切割，已经能使程序不会过多的堆积在一个 java 文件里面，


---
搬运地址：    

[MVC， MVP 和 MVVM 的图示](http://www.ruanyifeng.com/blog/2015/02/mvcmvp_mvvm.html)  

[Android App的设计架构：MVC, MVP , MVVM 与架构经验谈](https://www.tianmaying.com/tutorial/AndroidMVC)  

[Android App的设计模式概述：MVC、MVP、MVVM](https://www.jianshu.com/p/effad2e593df)  

[选择恐惧症的福音！教你认清 MVC ， MVP 和MVVM](http://zjutkz.net/2016/04/13/%E9%80%89%E6%8B%A9%E6%81%90%E6%83%A7%E7%97%87%E7%9A%84%E7%A6%8F%E9%9F%B3%EF%BC%81%E6%95%99%E4%BD%A0%E8%AE%A4%E6%B8%85MVC%EF%BC%8CMVP%E5%92%8CMVVM/)  

[如何构建 Android MVVM 应用框架](https://tech.meituan.com/android_mvvm.html)
