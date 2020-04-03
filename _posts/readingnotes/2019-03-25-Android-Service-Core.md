---
layout: post
title: Android 四大组件之 Service
category: 读书笔记
tags: Android开发艺术探索  Service 
---
* content
{:toc}
## 类名解释
### AMS
详情参照 [Android 四大组件之 Activity](../../../../../article-detail/2019/03/15/Android-Activity-Core/)

### ActivityService
辅助AMS进行Service管理的的类，包括Service的启动，绑定和停止等

### ServiceRecord
一个Service的记录类，可以理解为Service的栈吧。

还是之前的原则，能上图绝不贴代码
## Service 的工作流程
### startService()
都知道，startService()是Context中的方法，而ContextImpl 是Context的子类。所以流程图如下。

![Alt text](../../../../../article-detail/images/startservice.png)

又看到AMS了，就知道和这个有关，startService()应该和startActivity()有很多类似的地方吧。
两次IPC通信

肯定会和ApplicationThread还有ActivityThread有关，又肯定会套用我之前的公式吧

第一个IPC通信是在ContextImpl的startServiceCommon()中。

第二次的IPC通信是在ActivityService的realStartServiceLocked(),(好像startActivity()的第二次IPC通信名称是在realStartActivityLocked(),名称好像啊。)
看红色字体，熟悉吧，onStartCommand(),执行了

然后再详细看看handleCreateService()的流程吧
#### ActivityThread # handleCreateService()
![Alt text](../../../../../article-detail/images/handleCreateService.png)

在handleCreateService（）
* 创建了Service对象
* 如果Application没有创建，创建Application对象并执行onCreate()方法
* 执行Service的attch(),把Service和ContextImpl关联起来
* 执行Service 的onCreate()  
* 把Service添加到一个队列中。（干嘛用的？？）

不是说多次startService(),Service 的onCreat()方法只会执行一次啊，没看到相应的逻辑。留个疑问。

这样startService()的流程就完成了


### bindService()

![Alt text](../../../../../article-detail/images/bindService.png)

关注的重点：
1. 在 getServiceDispatcher()中，将ServiceConnection对象转换为ServiceDispatcher.InnerConnection，并添加到集合mService中。因为服务的绑定可能是夸进程的，因此ServiceConnection 必须得借助Binder才能让远程的服务端回调自己的方法。ServiceDispatcher.InnerConnection 正好充当Binder的角色。这样当Service和客户端建立连接后，系统就会通过InnerConnection来调用ServiceConnection的onServcieConnected()
2. ServiceDispatcher的作用就是连接ServiceConnection和InnerConnection
3. 和startService()一样，都会执行到 realStartServiceLocked()

这就产生疑问了，怎么这个也执行到了 realStartServiceLocked()。可是后面为啥又不一样了呢？分久必合合久必分？这个先放以下，先继续看 handleBindService()都做了啥操作吧

#### ActivityThread # handleBindService()
流程图来一份
![Alt text](../../../../../article-detail/images/handleBindService.png)

看到了熟悉的onBind()和onServiceConnected() 了

因为ServiceDispatcher.InnerConnection 的connect()是在Binder线程池中运行的，所以在ServiceDispatcher
connected()才会post 一个Runnable对象，切换到主线程中。然后再执行Connection.onServiceConnected()

这样bindService()的流程也就结束了。

现在来看分久必合合久必分的问题。
### bindService()和 startService()怎么区分
其实是一样的。这个就要贴代码了realStartServiceLocked()三个流程都执行了，只不过不同的场景，在不同的方法中返回不一样而已。

```java
private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInFg) throws RemoteException {
   //创建Service对象，并且调用onCreat()
   app.thread.scheduleCreateService(r, r.serviceInfo, mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo), app.repProcState);
   ...
   requestServiceBindingsLocked(r, execInFg);
   ...
   //通过sendServiceArgsLocked（）调用Service的其他方法，比如onStatCommand()
   sendServiceArgsLocked(r, execInFg, true);
   ...
}
```

既然 bindService()也执行了 调用了 sendServiceArgsLocked(),为啥没执行onStartCommand()呢。原因如下：

从 startSrvice()那个图也看出来了。在 执行 realStartServiceLocked()之前，startServiceLocked()中会有
```java
r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(), service, neededGrants));
```

这样一段代码，
创建一个 ServiceRecord.StartItem 对象，然后添加到了集合pendingStarts中
可以理解把startService()要启动的Service添加到了集合pendingStarts中
```java
//ServiceRecord.java
final ArrayList<StartItem> pendingStarts = new ArrayList<StartItem>();
```
而在sendServiceArgsLocked() 中就会有对pendingStarts这个集合的操作。
```java
private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg, boolean oomAdjusted){
    final int N = r.pendingStarts.size();
    if (N == 0) {
      return;
    }
    //会执行到onStartCommond()方法
}
```
bindService()没执行 startServiceLocked(),所以pendingStarts 集合是空的，那么就直接return了。

同理，在bindService()流程图中，在执行realStartServiceLocked()之前， bindServiceLocked()也有一个类似的操作。会执行到ServiceRecord的 retrieveAppBindingLocked()，在这里会创建一个IntentBindRecord对象，然后添加到集合bindings中，
```java
//ServiceRecord.java
final ArrayMap<Intent.FilterComparison, IntentBindRecord> bindings = new ArrayMap<Intent.FilterComparison, IntentBindRecord>();

public AppBindRecord retrieveAppBindingLocked(Intent intent, ProcessRecord app) {
       Intent.FilterComparison filter = new Intent.FilterComparison(intent);
       IntentBindRecord i = bindings.get(filter);
       if (i == null) {
           i = new IntentBindRecord(this, filter);
           bindings.put(filter, i);
       }
       AppBindRecord a = i.apps.get(app);
       if (a != null) {
           return a;
       }
       a = new AppBindRecord(this, i, app);
       i.apps.put(app, a);
       return a;
   }
```
而在 requestServiceBindingsLocked()中，也有对bindings的操作。
```java
private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg) {
    for (int i = r.bindings.size() - 1; i >= 0; i--) {
       IntentBindRecord ibr = r.bindings.valueAt(i);
       if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
           break;
       }
    }
}
```
因为startService() 不会执行到bindServiceLocked()中，所以 bindings这个集合就是空的，那么for循环就不会执行，也就执行不到requestServiceBindingLocked(),从而不会调用onBind()方法。


---
搬运地址：    

Android 开发艺术探索
