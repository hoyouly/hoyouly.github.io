---
layout: post
title: Android Binder 总结
category: 读书笔记
tags:  Android Binder
---

<!-- * content -->
<!-- {:toc} -->

# 基本概念
我们知道， Android 系统是基于 Linux 内核的。

## 进程隔离
```
进程隔离是为保护操作系统中进程互不干扰而设计的一组不同硬件和软件的技术。
这个技术是为了避免进程 A 写入进程 B 的情况发生。 进程的隔离实现，使用了虚拟地址空间。
进程 A 的虚拟地址和进程 B 的虚拟地址不同，这样就防止进程 A 将数据信息写入进程 B 。
```
操作系统的不同进程之间，数据是不共享，但是如果一个进程想要和另外一个进程进行通信，咋办呢，这就涉及到进程通信的问题。

## 用户空间/内核空间
内核空间： kernel space， kernel 是 Linux 操作的核心，独立于普通程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。
对于 Kernel 这么一个高安全级别的东西，显然是不容许其它的应用程序随便调用或访问的，所以需要对 Kernel 提供一定的保护机制，这个保护机制用来告诉那些应用程序，你只可以访问某些许可的资源，不许可的资源是拒绝被访问的，于是就把 Kernel 和上层的应用程序抽像的隔离开，分别称之为 Kernel Space 和 User Space 。

所以这里有两个隔离，<span style="border-bottom:1px solid red;">一个进程间是相互隔离的，二是进程内有用户和内核的隔离。</span> 即然有隔离，那么它们之前要相互配合时就得有合作（交互）。<span style="border-bottom:1px solid red;">进程间的交互就叫进程间通信（或称跨进程通信，简称IPC），而进程内的用户和内核的交互就是系统调用。</span>

通过系统调用，用户空间可以访问内核空间，可是如果一个用户空间想要访问另外一个用户空间呢，这就需要用到进程间通信了，我们知道 Android 是基于 Linux 内核的，为啥 Android 系统没有使用 Linux 中已有的 IPC 手段，例如管道， system V IPC ， socket ，却要重复造轮子，自己重新写一套 IPC 机制呢，即 Binder ，可以参照知乎上的回答，[为什么 Android 要采用 Binder 作为 IPC 机制？](https://www.zhihu.com/question/39440766/answer/93550572)。

## Binder 的优点
1. 性能方面      
  在移动设备上（性能受限制的设备，比如要省电），广泛地使用跨进程通信对通信机制的性能有严格的要求， Binder 相对出传统的 Socket 方式，更加高效。 Binder 数据拷贝只需要一次，而管道、消息队列、Socket都需要 2 次，共享内存方式一次内存拷贝都不需要，但实现方式又比较复杂。
2. 安全方面   
  传统的进程通信方式对于通信双方的身份并没有做出严格的验证，比如 Socket 通信 ip 地址是客户端手动填入，很容易进行伪造，而 Binder 机制从协议本身就支持对通信双方做身份校检，因而大大提升了安全性。
3. 调用方式，使用 Binder 时就和调用一个本地实例一样方便。

那么到底什么是 Binder 呢？
# 什么是Binder

* 是 Android 系统最重要的特性之一。
* 是系统间各个组建的桥梁。
* Android系统的开放式很大程度上得益于这种极其方便的跨进程通信机制
* 理解 Binder 对于理解整个 Android 系统有着重要的作用， Android 系统的四大组件， AMS ， PMS 等系统服务无一不与 Binder 挂钩。可以说`无 Binder 不Android`


# Binder 分类
Binder 分为 **Binder对象** 和 **Binder驱动**。

Binder 驱动就是主要的内核模块，而整个 Binder 对象就是通讯载体。可以自由的通过 Binder 驱动穿梭任意进程。所以客户端或者服务器可以把数据放入 Binder 对象里，然后进行调用和通讯。

## Binder 驱动
我们知道用户空间访问内核空间的唯一方式就是系统调用。

虽然 Binder 不是 Linux 内核的一部分，它可以访问内核空间是因为 Linux 的动态可加载内核模块（Loadable Kernel Module，LKM）机制。该模块是具有独立功能的程序，它可以被单独编译，但不能独立运行。它在运行时被链接到内核作为内核的一部分在内核空间运行。

这样， Android 系统可以通过添加一个内核模块运行在内核空间，用户进程之间的通信通过这个模块作为桥梁，就可以完成了。

<span style="border-bottom:1px solid red;"> 在 Android 系统中，这个运行在内核空间的，负责各个用户进程通过 Binder 通信的内核模块叫做 Binder 驱动</span>

Binder 驱动虽然默默无闻，却是通信的核心，尽管名叫驱动，实际上和硬件设备没有任何关系，只是实现方式和设备驱动程序是一样的。

## Binder 对象
Binder 对象是一个可以夸进程引用的对象，它的实现位于一个进程中，而它的引用却遍布与系统的各个进程之中。

最诱人的是，这个引用和 java 引用一样，既可以是强类型，也可以是弱类型，而且可以从一个进程传递给其他进程，让大家都能访问同一个 Server ，就像将一个对象或引用赋值给另一个引用一样。

分为本地对象和代理对象
1. Binder 本地对像：AIDL接口实现端的对象
2. Binder 代理对象： 它只是 Binder 本地对象的一个远程代理；对这个 Binder 代理对象的操作，会通过 Binder 驱动最终转发到 Binder 本地对象上去完成

Binder 驱动是主要的内核模块，而整个 Binder 对象就是通讯载体。可以自由的通过 Binder 驱动穿梭任意进程。所以客户端或者服务器可以把数据放入 Binder 对象里，然后进行调用和通讯。

## 从不同角度理解Binder
从不同的角度， Binder 可以有不同的解释：
* 可以理解为是 Android 的一个类，即Binder.java，实现了 IBinder 接口，
* 从 IPC 角度说，是 Android 的一种跨进程通信方式，这种方式在 Linux 中没有
* 还可以理解为一种虚拟的物理设备，即 Binder 驱动，设备驱动在/dev/binder,
* 从 Android Framework 角度来说，是 ServiceManager 连接各种Manager（ActivityManager， WindowManager 等）和形影的 ManagerService 的桥梁
* 从 Android 应用层来说，是客户端和服务端进行通信的媒介，当 bindService 的时候，服务端会返回一个包含了服务端业务调用的 Binder 对象，通过这个 Binder 对象，客户端就可以获取服务器提供的服务或者数据了，这里的服务包括普通的服务和基于 AIDL 的服务
* 对于 Server 进程来说， Binder 指的是 Binder 本地对象。
* 对于 Client 来说， Binder 指的是 Binder 代理对象，**对于一个拥有 Binder 对象的使用者而言，它无须关心这是一个 Binder 代理对象还是 Binder 本地对象；对于代理对象的操作和对本地对象的操作对它来说没有区别。**
* 对于传输过程而言， Binder 是可以进行跨进程传递的对象；Binder驱动会对具有跨进程传递能力的对象做特殊处理：自动完成代理对象和本地对象的转换。

面向对象思想的引入将进程间通信转化为通过某个 Binder 对象的引用调用该对象的方法。

Binder 模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行与同一个面向对象的程序之中，形形色色的 Binder 对象以及星罗棋布的引用仿佛粘是结整个应用程序的胶水，这也是 Binder 在英文中的原意。

# Binder框架

虽然很多人都用访问网络来解释 Binder ，但是总感觉不太好。不过还是把这个图贴上了。
![Binder关系图](../../../../images/Binder_1.png)

Binder 的框架采用C/S架构，也包含四个角色： Server， Client ， ServiceManager 以及 Binder 驱动。其中 Server ， Client ， SM 运行用户空间，而 Binder 驱动运行在内核空间。

![Binder关系图](../../../../images/Binder_2.png)
* Server， Client ， ServiceManager 运行用户空间，而 Binder 驱动运行在内核空间。
* Binder驱动和 Service Manager 可以看做是 Android 平台的基础架构，而 Client 和 Server 是 Android 的应用层，
* 开发人员只需自定义实现client、Server端，借助 Android 的基本平台架构便可以直接进行 IPC 通信
1. 和 DNS 类似， ServiceManager 的作用是将字符形式的 Binder 转化为 Client 中对 Binder 的引用，使得 Client 能够通过 Binder 名字获得对 Server 中的 Binder 实体的引用。
2. 注册了名字的 Binder 叫实名 Binder ，就像每个网站出来有 IP 地址，还有自己的网址，
3. Server创建了 Binder 实体，为其取一个字符形式，可读易记的名字(例如张三)，
4. 将这个 Binder 连同名字以数据的形式通过 Binder 驱动发送给 ServiceManager ,通知 ServiceManager 注册了一个名叫张三的 Binder ，它位于某个 Server 中， Binder 驱动为了这个穿过进程边界的 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用。将名字以及新建的应用打包传递给ServiceManager
5. ServiceManager获得数据包后，从中取出名字和引用填入一张查找表，
6. Server向 ServiceManager 注册了 Binder 的引用以及其名字后， Client 就可以通过名字获得该 Binder 的引用了。

# Binder 原理
Binder 采用C/S架构，从组件视角来说，包含 Client ， Server ， ServiceManager 以及 Binder 驱动，其中 ServiceManager 用于管理系统中的各种服务。架构图如下所示：
![](../../../../images/Binder_3.jpg)

可以看出，无论是注册服务还是获取服务，都需要 ServiceManager ,这里的 ServiceManager 是 Native 层的 ServiceManager ,而不是 framework 层的 ServiceManager 。

<font color="#ff000" > ServiceManager是整个 Binder 通信机制的大管家，是 Android 进程间通信机制 Binder 的守护进程。</font>

要掌握 Binder 机制，首先需要了解系统是如何首次启动 ServiceManager 。当 ServiceManager 启动之后， Client 端和 Server 端通信时都需要先获取 ServiceManager 接口，才能开始通信服务。

图中Client/Server/ServiceManage之间的相互通信都是基于 Binder 机制。既然基于 Binder 机制通信，那么同样也是C/S架构，则图中的 3 大步骤都有相应的 Client 端与 Server 端。
1. 注册服务(addService)：Server进程要先注册 Service 到 ServiceManager 。该过程：Server是客户端， ServiceManager 是服务端。
2. 获取服务(getService)：Client进程使用某个 Service 前，须先向 ServiceManager 中获取相应的 Service 。该过程：Client是客户端， ServiceManager 是服务端。
3. 使用服务：Client根据得到的 Service 信息建立与 Service 所在的 Server 进程通信的通路，然后就可以直接与 Service 交互。该过程：client是客户端， server 是服务端。

图中的 Client , Server , Service Manager 之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与 Binder 驱动进行交互的，从而实现 IPC 通信方式。

![](../../../../images/binder_4.png)

整个个通信步骤如下：
1. SM 建立（通讯录建立）；首先有一个进程向驱动提出申请为 SM ，驱动同意后， SM 进程负责管理Service（注意这里是 Service 而不是 Server ，因为如果通信过程反过来的话，那么原来的客户端 Client 也会称为服务端Server）不过这个时候通讯录还是空的，一个号码都没有。
2. 各个 Server 向 SM 注册（完善通讯录）；每个 Server 端进程启动之后，向 SM 报告。我是张三，要找我请返回0X1234(这个地址没有实际意义，类比)，其他 Server 进程以此如此，这样 SM 就建立了一张表。对应着各个 Server 的名字和地址。
3. Client 想要与 Server 通信，首先询问SM；请告诉我如何联系张三， SM 收到后给它一个号码 0X1234 ， Client 收到之后，开心的用这个号码拨通了 Server 的电话，于是就开始通信了。

* Server进程里面的 Binder 对象指的是 Binder 本地对象，
* Client里面的对象指的是 Binder 的代理对象，
* 在 Binder 对象进行跨进程传递的时候， Binder 驱动会自动完成这两种类型的转换；
* Binder驱动保存了一个跨进程的 Binder 对象的相关信息，在驱动中， Binder 本地对象的代表是一个叫做 binder_node 的数据结构， Binder 代理对象是用 binder_ref 代表的，有地方把 Binder 本地对象直接称作 Binder 实体，把 Binder 代理对象称作 Binder 引用（句柄）



## 来个例子
一个 Client 进程想要调用 Server 进程中的 object 对象的一个 add() 方法。
1. 首先 Server 进程要向 SM 注册；告诉自己是谁，自己有什么能力；例如 它叫 zhangsan ，它有一个 object 对象，可以执行 add 操作；于是 SM 建立了一张表：zhangsan这个名字对应的进程Server;
2. 然后 Client 向 SM 查询：我需要联系一个名字叫做 zhangsan 的 Server 进程里面的 object 对象；这时候关键来了：<font color="#ff000" >进程之间通信的数据都会经过运行在内核空间里面的 Binder 驱动， Binder 驱动在数据流过的时候做了一点手脚，它并不会给 Client 进程返回一个真正的 object 对象，而是返回一个看起来跟 object 一模一样的代理对象 objectProxy ，这个 objectProxy 也有一个 add 方法，但是这个 add 方法只是一个傀儡，并没有 Server 进程里面 object 对象的 add 方法那个能力，它唯一做的事情就是把参数包装然后交给 Binder 驱动。但是 Client 进程并不知道 Binder 驱动返回给它的对象动过手脚，毕竟伪装的太像了，如假包换。 Client 开开心心地拿着 objectProxy 对象然后调用 add 方法；我们说过，这个 add 什么也不做，直接把参数做一些包装然后直接转发给 Binder 驱动。</font>
3. Binder驱动收到这个消息，发现是个objectProxy；一查表就明白了：之前用 objectProxy 替换了 object 发送给 Client 了，它真正应该要访问的是 object 对象的 add 方法；于是 Binder 驱动通知 Server 进程，调用你的 object 对象的 add 方法，然后把结果发给我， Sever 进程收到这个消息，执行 add() 操作，然后将结果返回 Binder 驱动， Binder 驱动然后把结果返回给 Client 进程。

于是整个过程就完成了。由于 Binder 驱动返回的 objectProxy 与 Server 进程里面原始的 object 是如此相似，**给人感觉好像是直接把 Server 进程里面的对象 object 传递到了 Client 进程；而实际上 Client 进程只不过是持有了 Server 端的代理对象而已；代理对象协助驱动完成了跨进程通信。因此，我们可以说 Binder 对象是可以进行跨进程传递的对象**

# Binder使用场景
Android 开发中， Binder 主要用于在 Service 中，包括 AIDL 和 Messager ，其中普通服务中的 Binder 不涉及到进程通信，另外， ContentProvider 底层实现也是 Binder ，
所以 Binder 的使用场景如下：
![](../../../../images/binder_5.png)

# 其他

一个进程的 Binder 线程数默认最大是 16 ，超过的请求会被阻塞等待空闲的 Binder 线程。所以使用 ContentProvider 时（又一个使用 Binder 机制的组件），你就很清楚它的CRUD（创建、检索、更新和删除）方法只能同时有 16 个线程在跑。

进程 1 和进程 2 希望进行通信，所以必须借助 Binder 驱动来交互，而参与的进程需要一个类似 ip 地址的唯一标志，其中 binder 中的“ip地址”是动态的，所以为了防止频繁的获取“ip地址”，带保存、映射机制的 Service Manager 顺利成章地出现了，而 Service Manager 它在 binder 通信中的“ip地址”则永远是 0 。


Client 中的 Binder 也可以看作是 Server Binder 的‘代理’，在本地代表远端 Server 为 Client 提供服务。
面向对象思想的引入**将进程间通信转化为通过对某个 Binder 对象的引用调用该对象的方法**，而其独特之处在于**Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中。**


Binder 基于Client-Server通信模式，传输过程只需一次拷贝，为发送发添加UID/PID身份，既支持实名 Binder 也支持匿名 Binder ，安全性高。
要想实现Client-Server通信据必须实现以下两点：
1. server必须有确定的访问接入点或者说地址来接受 Client 的请求，并且 Client 可以通过某种途径获知 Server 的地址；
2. 制定Command-Reply协议来传输数据。例如在网络通信中 Server 的访问接入点就是 Server 主机的 IP 地址+端口号，传输协议为 TCP 协议。

对 Server 而言， Binder 可以看成 Server 提供的实现某个特定服务的访问接入点， Client 通过这个‘地址’向 Server 发送请求来使用该服务；对 Client 而言， Binder 可以看成是通向 Server 的管道入口，要想和某个 Server 通信首先必须建立这个管道并获得管道入口。

---
参考链接：

Android 开发艺术探索

[Android Binder之应用层总结与分析](http://blog.csdn.net/qian520ao/article/details/78089877)

[Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)

[Android面试一天一题（Day 35：神秘的 Binder 机制）](https://www.jianshu.com/p/c7bcb4c96b38)
