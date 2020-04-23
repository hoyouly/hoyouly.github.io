---
layout: post
title: Android 四大组件之 Service
category: 读书笔记
tags: Android开发艺术探索  Service 
---
<!-- * content -->
<!-- {:toc} -->
## 类名解释
### AMS
详情参照 [Android 四大组件之 Activity](../../../../2019/03/15/Android-Activity-Core/)

### ActivityService
辅助 AMS 进行 Service 管理的的类，包括 Service 的启动，绑定和停止等

### ServiceRecord
一个 Service 的记录类，可以理解为 Service 的栈吧。

还是之前的原则，能上图绝不贴代码
## Service 的工作流程
### startService()
都知道， startService() 是 Context 中的方法，而 ContextImpl 是 Context 的子类。所以流程图如下。

![Alt text](../../../../images/startservice.png)

又看到 AMS 了，就知道和这个有关， startService() 应该和 startActivity() 有很多类似的地方吧。
两次 IPC 通信

肯定会和 ApplicationThread 还有 ActivityThread 有关，又肯定会套用我之前的公式吧

第一个 IPC 通信是在 ContextImpl 的 startServiceCommon() 中。

第二次的 IPC 通信是在 ActivityService 的 realStartServiceLocked() ,(好像 startActivity() 的第二次 IPC 通信名称是在 realStartActivityLocked() ,名称好像啊。)
看红色字体，熟悉吧， onStartCommand() ,执行了

然后再详细看看 handleCreateService() 的流程吧
#### ActivityThread # handleCreateService()
![Alt text](../../../../images/handleCreateService.png)

在handleCreateService（）
* 创建了 Service 对象
* 如果 Application 没有创建，创建 Application 对象并执行 onCreate() 方法
* 执行 Service 的 attch() ,把 Service 和 ContextImpl 关联起来
* 执行 Service 的onCreate()  
* 把 Service 添加到一个队列中。（干嘛用的？？）

不是说多次 startService() , Service 的 onCreat() 方法只会执行一次啊，没看到相应的逻辑。留个疑问。

这样 startService() 的流程就完成了


### bindService()

![Alt text](../../../../images/bindService.png)

关注的重点：
1. 在 getServiceDispatcher() 中，将 ServiceConnection 对象转换为ServiceDispatcher.InnerConnection，并添加到集合 mService 中。因为服务的绑定可能是夸进程的，因此 ServiceConnection 必须得借助 Binder 才能让远程的服务端回调自己的方法。ServiceDispatcher.InnerConnection 正好充当 Binder 的角色。这样当 Service 和客户端建立连接后，系统就会通过 InnerConnection 来调用 ServiceConnection 的onServcieConnected()
2. ServiceDispatcher的作用就是连接 ServiceConnection 和InnerConnection
3. 和 startService() 一样，都会执行到 realStartServiceLocked()

这就产生疑问了，怎么这个也执行到了 realStartServiceLocked() 。可是后面为啥又不一样了呢？分久必合合久必分？这个先放以下，先继续看 handleBindService() 都做了啥操作吧

#### ActivityThread # handleBindService()
流程图来一份
![Alt text](../../../../images/handleBindService.png)

看到了熟悉的 onBind() 和 onServiceConnected() 了

因为ServiceDispatcher.InnerConnection 的 connect() 是在 Binder 线程池中运行的，所以在ServiceDispatcher
connected() 才会 post 一个 Runnable 对象，切换到主线程中。然后再执行Connection.onServiceConnected()

这样 bindService() 的流程也就结束了。

现在来看分久必合合久必分的问题。
### bindService()和 startService() 怎么区分
其实是一样的。这个就要贴代码了 realStartServiceLocked() 三个流程都执行了，只不过不同的场景，在不同的方法中返回不一样而已。

```java
private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app , boolean execInFg) throws RemoteException {
   //创建 Service 对象，并且调用onCreat()
   app.thread.scheduleCreateService(r, r.serviceInfo, mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo), app.repProcState);
   ...
   requestServiceBindingsLocked(r, execInFg);
   ...
   //通过sendServiceArgsLocked（）调用 Service 的其他方法，比如onStatCommand()
   sendServiceArgsLocked(r, execInFg , true);
   ...
}
```

既然 bindService() 也执行了 调用了 sendServiceArgsLocked() ,为啥没执行 onStartCommand() 呢。原因如下：

从 startSrvice() 那个图也看出来了。在 执行 realStartServiceLocked() 之前， startServiceLocked() 中会有
```java
r.pendingStarts.add(new ServiceRecord.StartItem(r, false , r.makeNextStartId(), service , neededGrants));
```

这样一段代码，
创建一个 ServiceRecord.StartItem 对象，然后添加到了集合 pendingStarts 中
可以理解把 startService() 要启动的 Service 添加到了集合 pendingStarts 中
```java
//ServiceRecord.java
final ArrayList<StartItem> pendingStarts = new ArrayList<StartItem>();
```
而在 sendServiceArgsLocked() 中就会有对 pendingStarts 这个集合的操作。
```java
private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg , boolean oomAdjusted){
    final int N = r.pendingStarts.size();
    if (N == 0) {
      return;
    }
    //会执行到 onStartCommond() 方法
}
```
bindService() 没执行 startServiceLocked() ,所以 pendingStarts 集合是空的，那么就直接 return 了。

同理，在 bindService() 流程图中，在执行 realStartServiceLocked() 之前， bindServiceLocked() 也有一个类似的操作。会执行到 ServiceRecord 的 retrieveAppBindingLocked() ，在这里会创建一个 IntentBindRecord 对象，然后添加到集合 bindings 中，
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
       a = new AppBindRecord(this, i , app);
       i.apps.put(app, a);
       return a;
   }
```
而在 requestServiceBindingsLocked() 中，也有对 bindings 的操作。
```java
private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg) {
    for (int i = r.bindings.size() - 1; i >= 0; i--) {
       IntentBindRecord ibr = r.bindings.valueAt(i);
       if (!requestServiceBindingLocked(r, ibr , execInFg , false)) {
           break;
       }
    }
}
```
因为 startService() 不会执行到 bindServiceLocked() 中，所以 bindings 这个集合就是空的，那么 for 循环就不会执行，也就执行不到 requestServiceBindingLocked() ,从而不会调用 onBind() 方法。


---
搬运地址：    

Android 开发艺术探索
