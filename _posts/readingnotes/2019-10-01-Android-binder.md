---
layout: post
title: Android Binder 总结
category: 读书笔记
tags: Android开发艺术探索 Binder
---

* content
{:toc}

Binder（胶水，粘合剂 ）

Binder模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行与同一个面向对象的程序之中，形形色色的Binder对象以及星罗棋布的引用仿佛是粘结整个应用程序的胶水，这也是Binder在英文中的原意。

Android开发中，Binder主要用于在Service中，包括AIDL和Messager，其中普通服务中的Binder不涉及到进程通信，而Messager底层是AIDL技术。

## Binder 说明

* 是Android系统最重要的特性之一。
* 是系统间各个组建的桥梁。
* Android系统的开放式很大程度上得益于这种极其方便的跨进程通信机制
* 理解Binder对于理解整个Android系统有着重要的作用，Android系统的四大组件，AMS，PMS等系统服务无一不与Binder挂钩。可以说`无Binder不Android`

Binder 从不同的角度，可以有不同的解释：
* 是Android的一个类，即Binder.java，实现了IBinder接口，
* 从IPC角度说，是Android的一种跨进程通信方式，这种方式在Linux中没有
* 还可以理解为一种虚拟的物理设备，Binder驱动，设备驱动在/dev/binder,
* 从Android Framework角度来说，是ServiceManager连接各种Manager（ActivityManager，WindowManager等）和形影的ManagerService的桥梁
* 从Android应用层来说，是客户端和服务端进行通信的媒介，当bindService()的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务器提供的服务或者数据了，这里的服务包括普通的服务和基于AIDL的服务
* 对于Server进程来说，Binder指的是Binder本地对象。
* 对于Client来说，Binder指的是Binder代理对象，**对于一个拥有Binder对象的使用者而言，它无须关心这是一个Binder代理对象还是Binder本地对象；对于代理对象的操作和对本地对象的操作对它来说没有区别。**
* 对于传输过程而言，Binder是可以进行跨进程传递的对象；Binder驱动会对具有跨进程传递能力的对象做特殊处理：自动完成代理对象和本地对象的转换。

## Binder 分类
Binder分为 **Binder对象** 和 **Binder驱动**。

Binder驱动就是主要的内核模块，而整个Binder对象就是通讯载体。可以自由的通过Binder驱动穿梭任意进程。所以客户端或者服务器可以把数据放入Binder对象里，然后进行调用和通讯。

### Binder 驱动
在Android系统中，这个运行在内核空间的，负责各个用户进程通过Binder通信的内核模块叫做Binder驱动。Binder驱动虽然默默无闻，却是通信的核心。尽管名叫驱动，实际上和硬件设备没有任何关系，只是实现方式和设备驱动程序是一样的。
* 不是Linux内核的一部分，它可以访问内核空间是因为Linux的动态可加载内核模块（Loadable Kernel Module，LKM）机制。
* 是具有独立功能的程序，它可以被单独编译，但不能独立运行。
* 它在运行时被链接到内核作为内核的一部分在内核空间运行。

### Binder 对象
面向对象思想的引入将进程间通信转化为通过某个Binder对象的引用调用该对象的方法。该对象具有以下几个特征：
* 是一个可以夸进程引用的对象。
* 实现位于一个进程中，而它的引用却遍布与系统的各个进程之中。
* 和java引用一样，既可以是强类型，也可以是弱类型。
* 可以从一个进程传递给其他进程，让大家都能访问同一个Server，就像将一个对象或引用赋值给另一个一样。

分为本地对象和代理对象
1. Binder 本地对像： AIDL接口实现端的对象
2. Binder 代理对象： 它只是Binder本地对象的一个远程代理；对这个Binder代理对象的操作，会通过Binder驱动最终转发到Binder本地对象上去完成


## Binder框架
### 小明告状
我们来看个例子。小明同学要打电话的向教育局反馈老师打人这件事。
#### 前提条件：
1. 教育局 能处理老师打人这件事。                    
  * IBookManager 中 有 getBookList()
2. 教育局中有具体人负责处理打人这事，比如是老王同志。   
  * 创建一个Stub,实现 IBookManager接口，
3. 教育局的电话是可以在电话本里面查找到。             
  * onBind()中返回该Stub对象。
4. 老王处理老师打人这事已经上报给了客户服务中心。
5. 小明知道 教育局能处理老师打人这事        
  * 客户端 知道能通过IBookManager的 getBookList()得到书本列表

#### 整个流程
一. 小明同学通过电话本找到教育局的电话。例如 123456   ，
  * 其实很多时候，服务端会告诉你怎么连接服务，比如通过某个Action即可连接上去。
```Java
Intent commonIntent = new Intent();
commonIntent.setAction("com.hoyouly.android_art.service");
commonIntent.setPackage(getContext().getPackageName());
```
二. 拿起电话，开始拨号 123456     
  * 知道Action后，开始连接服务器。
```java
getContext().bindService(commonIntent, mConnection, Context.BIND_AUTO_CREATE);
```
三. 这时会有两种可能，电话接通，或者电话未接通。
    1. 电话接通 对方会说，你好，这里是***教育局客服服务中心，有什么可以帮到你的吗
      * 执行到 onServiceConnected(),这里面会有一个iBinder对象
    2. 电话未接通
      * 执行到 onServiceDisconnected（）

```Java
private ServiceConnection mConnection = new ServiceConnection() {
		@Override
		public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
			mService = IBookManager.Stub.asInterface(iBinder);
		}

		@Override
		public void onServiceDisconnected(ComponentName componentName) {
			mService = null;
		}
};
```
我们只看电话接通的情况。
四. 小明说我要找处理老师打人这件事
  * 得到mService对象。
```java
mService = IBookManager.Stub.asInterface(iBinder);
```
五. 转接成功。小明同学以为是接的电话的人就能处理老师打人这事。于是开始说老师打人这件事（我是XXX学校的学生，上课期间无缘无语被XX老师打了一顿。很伤心..）可是实际上接电话的只是一个客服。  
  *  执行 mService.getBookList(),mService 只是一个代理对象。
六. 客服自己知道没有处理老师打人这件事的能力，可是他会把这件事记录下来，发送给客户服务中心。并且会说，请稍等。正在处理。
  * mService 把数据封装起来，然后执行transact()通过Binder驱动发送给服务端，客户服务中心就相当于Binder驱动
```java
public java.util.List<com.hoyouly.android_art.Book> getBookList() throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    java.util.List<com.hoyouly.android_art.Book> _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
        _reply.readException();
        _result = _reply.createTypedArrayList(com.hoyouly.android_art.Book.CREATOR);
    } finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```
七. 客户服务中心知道这需要老王处理，于是告诉让老王处理老师打人这件事，经过核实，是老师不对，然后把老师停职。于是把结果告诉客服。这期间，小明和客服都处于等待中。   
  * 服务端开始处理，然后通过onTransact()把结果给代理对象。这期间，客户端处于等待状态
```java
//IBookManager # Stub
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
    switch (code) {
      ...
        case TRANSACTION_getBookList: {
            data.enforceInterface(DESCRIPTOR);
            java.util.List<com.hoyouly.android_art.Book> _result = this.getBookList();
            reply.writeNoException();
            reply.writeTypedList(_result);
            return true;
        }
      ...
    }
    return super.onTransact(code, data, reply, flags);
}
```
```java
// BookManagerService # Stub
@Override
public List<Book> getBookList() throws RemoteException {
   SystemClock.sleep(5000);
   return mBookList;
}
```
8. 客服得到结果，然后把结果告诉给小明同学。老师被停职了。  小明愉快的挂断了电话，
* 断开连接
```java  
getContext().unbindService(mConnection);
```
这里面小明就是客户端，教育局就是服务端，电话本就是ServiceManager,客户服务中心就是Binder驱动。
而老王呢，则是Stub类，客服就是Proxy.

## 再来个例子
一个 Client进程想要调用Server进程中的object对象的一个add()方法。
1. 首先Server进程要向SM注册；告诉自己是谁，自己有什么能力；例如 它叫zhangsan，它有一个object对象，可以执行add操作；于是SM建立了一张表：zhangsan这个名字对应的进程Server;
2. 然后Client向SM查询：我需要联系一个名字叫做zhangsan的Server 进程里面的object对象；这时候关键来了：<font color="#ff000" >进程之间通信的数据都会经过运行在内核空间里面的Binder驱动，Binder驱动在数据流过的时候做了一点手脚，它并不会给Client进程返回一个真正的object对象，而是返回一个看起来跟object一模一样的代理对象objectProxy，这个objectProxy也有一个add方法，但是这个add方法只是一个傀儡，并没有Server进程里面object对象的add方法那个能力，它唯一做的事情就是把参数包装然后交给Binder驱动。但是Client进程并不知道Binder驱动返回给它的对象动过手脚，毕竟伪装的太像了，如假包换。Client开开心心地拿着objectProxy对象然后调用add方法；我们说过，这个add什么也不做，直接把参数做一些包装然后直接转发给Binder驱动。</font>
3. Binder驱动收到这个消息，发现是个objectProxy；一查表就明白了：之前用objectProxy替换了object发送给Client了，它真正应该要访问的是object对象的add方法；于是Binder驱动通知Server进程，调用你的object对象的add方法，然后把结果发给我，Sever进程收到这个消息，执行add()操作，然后将结果返回Binder驱动，Binder驱动然后把结果返回给Client进程。

于是整个过程就完成了。由于Binder驱动返回的objectProxy与Server进程里面原始的object是如此相似，**给人感觉好像是直接把Server进程里面的对象object传递到了Client进程；而实际上Client进程只不过是持有了Server端的代理对象而已；代理对象协助驱动完成了跨进程通信。因此，我们可以说Binder对象是可以进行跨进程传递的对象**

虽然很多人都用访问网络来解释 Binder，但是总感觉不太好。不过还是把这个图贴上了。
![Binder关系图](https://github.com/hoyouly/BlogResource/raw/master/imges/Binder_1.png)

在Binder框架中，也定义了四个角色。Server，Client，ServiceManager(SMgr)以及Binder驱动。
这四个角色关系和上面类似，Server是服务器，Client是客户端，SMgr是域名服务器（DNS），驱动是路由器。
![](https://github.com/hoyouly/BlogResource/raw/master/imges/Binder_2.png)
* Server，Client，SMgr 运行用户空间，而Binder驱动运行在内核空间。
* Binder驱动和Service Manager可以看做是Android平台的基础架构，而Client和Server是Android的应用层，
* 开发人员只需自定义实现client、Server端，借助Android的基本平台架构便可以直接进行IPC通信
1. 和DNS类似，SMgr的作用是将字符形式的Binder转化为Client中对Binder的引用，使得Client能够通过Binder名字获得对Server中的Binder实体的引用。
2. 注册了名字的Binder叫实名Binder，就像每个网站出来有IP地址，还有自己的网址，
3. Server创建了Binder实体，为其取一个字符形式，可读易记的名字(例如张三)，
4. 将这个Binder连同名字以数据的形式通过Binder驱动发送给 SMgr,通知SMgr注册了一个名叫张三的Binder，它位于某个Server中，Binder驱动为了这个穿过进程边界的Binder创建位于内核中的实体节点以及SMgr对实体的引用。将名字以及新建的应用打包传递给SMgr
5. SMgr获得数据包后，从中取出名字和引用填入一张查找表，
6. Server向SMgr注册了Binder的引用以及其名字后，Client就可以通过名字获得该Binder的引用了。


## Binder 原理
Binder采用C/S架构，从组件视角来说，包含Client，Server，ServiceManager以及Binder驱动，其中ServiceManager用于管理系统中的各种服务。架构图如下所示：
![](https://github.com/hoyouly/BlogResource/raw/master/imges/Binder_3.jpg)

可以看出，无论是注册服务还是获取服务，都需要SMgr,这里的SMgr是Native层的SMgr,而不是framework层的SMgr，SMgr是整个Binder通信机制的大管家，是Android进程间通信机制Binder的守护进程。要掌握Binder机制，首先需要了解系统是如何首次启动Service Manager。当Service Manager启动之后，Client端和Server端通信时都需要先获取Service Manager接口，才能开始通信服务。

图中Client/Server/ServiceManage之间的相互通信都是基于Binder机制。既然基于Binder机制通信，那么同样也是C/S架构，则图中的3大步骤都有相应的Client端与Server端。
1. 注册服务(addService)：Server进程要先注册Service到ServiceManager。该过程：Server是客户端，ServiceManager是服务端。
2. 获取服务(getService)：Client进程使用某个Service前，须先向ServiceManager中获取相应的Service。该过程：Client是客户端，ServiceManager是服务端。
3. 使用服务：Client根据得到的Service信息建立与Service所在的Server进程通信的通路，然后就可以直接与Service交互。该过程：client是客户端，server是服务端。

图中的Client,Server,Service Manager之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与Binder驱动进行交互的，从而实现IPC通信方式。


![](https://github.com/hoyouly/BlogResource/raw/master/imges/binder_4.png)

整个个通信步骤如下：
1. SM 建立（通讯录建立）；首先有一个进程向驱动提出申请为SM，驱动同意后，SM进程负责管理Service（注意这里是Service而不是Server，因为如果通信过程反过来的话，那么原来的客户端Client也会称为服务端Server）不过这个时候通讯录还是空的，一个号码都没有。
2. 各个Server向SM注册（完善通讯录）；每个Server端进程启动之后，向SM报告。我是张三，要找我请返回0X1234(这个地址没有实际意义，类比)，其他Server进程以此如此，这样SM就建立了一张表。对应着各个Server的名字和地址。
3. Client 想要与Server通信，首先询问SM；请告诉我如何联系张三，SM收到后给它一个号码0X1234，Client收到之后，开心的用这个号码拨通了Server的电话，于是就开始通信了。

* Server进程里面的Binder对象指的是Binder本地对象，
* Client里面的对象指的是Binder的代理对象，
* 在Binder对象进行跨进程传递的时候，Binder驱动会自动完成这两种类型的转换；
* Binder驱动保存了一个跨进程的Binder对象的相关信息，在驱动中，Binder本地对象的代表是一个叫做binder_node的数据结构，Binder代理对象是用binder_ref代表的，有地方把Binder本地对象直接称作Binder实体，把Binder代理对象称作Binder引用（句柄）


进程1和进程2希望进行通信，所以必须借助Binder驱动来交互，而参与的进程需要一个类似ip地址的唯一标志，其中binder中的“ip地址”是动态的，所以为了防止频繁的获取“ip地址”，带保存、映射机制的Service Manager顺利成章地出现了，而Service Manager它在binder通信中的“ip地址”则永远是0。


Client中的Binder也可以看作是Server Binder的‘代理’，在本地代表远端Server为Client提供服务。
面向对象思想的引入**将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法**，而其独特之处在于**Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中。**


Binder基于Client-Server通信模式，传输过程只需一次拷贝，为发送发添加UID/PID身份，既支持实名Binder也支持匿名Binder，安全性高。
要想实现Client-Server通信据必须实现以下两点：
1. server必须有确定的访问接入点或者说地址来接受Client的请求，并且Client可以通过某种途径获知Server的地址；
2. 制定Command-Reply协议来传输数据。例如在网络通信中Server的访问接入点就是Server主机的IP地址+端口号，传输协议为TCP协议。

对Server而言，Binder可以看成Server提供的实现某个特定服务的访问接入点， Client通过这个‘地址’向Server发送请求来使用该服务；对Client而言，Binder可以看成是通向Server的管道入口，要想和某个Server通信首先必须建立这个管道并获得管道入口。

Client通过Binder的引用访问Server



# Binder使用场景
Android开发中，Binder主要用于在Service中，包括AIDL和Messager，其中普通服务中的Binder不涉及到进程通信，另外，ContentProvider底层实现也是Binder，
所以Binder的使用场景如下：
![](https://github.com/hoyouly/BlogResource/raw/master/imges/binder_5.png)

---
搬运地址：
Android 开发艺术探索

[Android Binder之应用层总结与分析](http://blog.csdn.net/qian520ao/article/details/78089877)

[Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)

[Android面试一天一题（Day 35：神秘的Binder机制）](https://www.jianshu.com/p/c7bcb4c96b38)
