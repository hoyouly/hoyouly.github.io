---
layout: post
title: Android 四大组件之 Broadcast
category: 读书笔记
tags: Android开发艺术探索   Broadcast
---
* content
{:toc}
## 类名解释
### AMS
详情参照 [Android 四大组件之 Activity](../../../../2019/03/15/Android-Activity-Core/)

### ReceiverDispatcher
广播分发者,同时保留 BroadcastReceiver 和 InnerReceiver.
### ReceiverDispatcher. InnerReceiver

InnerReceiver 继承于IIntentReceiver.Stub。

IIntentReceiver是一个Binder接口,所以InnerReceiver 是一个Binde类，可以用于用于Binder IPC通信如下图，
![Alt text](../../../../images/broadcast_1.png)

### BroadcastRecord

### ReceiverList
接收者队列
### BroadcastFilter
广播过滤者

### BroadcastQueue
里面有两个队列，分别用来存储无序广播和有序广播的

```java
// 存储无序广播
final ArrayList<BroadcastRecord> mParallelBroadcasts = new ArrayList<BroadcastRecord>();
//存储有序广播
final ArrayList<BroadcastRecord> mOrderedBroadcasts = new ArrayList<BroadcastRecord>();
```
在AMS创建的时候，会创建两个这样的队列，通过intent可以判断前台广播队列和后台广播队列

```java
public ActivityManagerService(Context systemContext) {
    ...
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler, "foreground", BROADCAST_FG_TIMEOUT, false);
  mBgBroadcastQueue = new BroadcastQueue(this, mHandler, "background", BROADCAST_BG_TIMEOUT, true);
    ...
}
```
通过 Intent.FLAG_RECEIVER_FOREGROUND 可以区分是前台广播还是后台广播。

当发送广播的时候设置了这个标志，会允许接收者以前台的优先级运行，有更短的时间间隔。正常广播的接受者是后台优先级，不会被自动提升。


## Broadcast 的工作流程
### registerReceiver()

我们注册广播，最常用的就是在Activity/Service中调用 registerReceiver()方法，最终会执行到 ContextImpl 的
registerReceiver(BroadcastReceiver receiver, IntentFilter filter,String broadcastPermission, Handler scheduler)

* receiver 要注册的广播
* filter 就是Filter
* broadcastPermission  拥有广播的权限控制
* scheduler  用于指定接收到广播时onRecive执行线程，当scheduler=null则默认代表在主线程中执行

```java
@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,String broadcastPermission, Handler scheduler) {
    return registerReceiverInternal(receiver, getUserId(),filter, broadcastPermission, scheduler, getOuterContext());
}

private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
      ...
      if (scheduler == null) {
          //将主线程Handler赋予scheuler
          scheduler = mMainThread.getHandler();
      }
      rd = mPackageInfo.getReceiverDispatcher(receiver, context, scheduler,
          mMainThread.getInstrumentation(), true);  
      ...
      return ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
    }
```
1. 执行 LoadedApk # getReceiverDispatcher()，得到一个 InnerReceiver对象 。在这个getReceiverDispatcher()主要流程是这样的。
    1. mReceivers是一个以Context为key，以BroadcastReceiver(广播接收者)为key，LoadedApk.ReceiverDispatcher(分发者)为value的ArrayMap
    2. 根据 我们要注册的广播，从 mReceivers 集合中查询对应的 ReceiverDispatcher对象， mReceivers是一个以Context为key，以BroadcastReceiver(广播接收者)为key，LoadedApk.ReceiverDispatcher(分发者)为value的ArrayMap
      * 如果得到该对象，则直接通过该对象返回得到的IIntentReceiver对象，
      * 如果没有得到该对象，则根据 BroadcastReceiver 创建一个 ReceiverDispatcher对象，创建 ReceiverDispatcher 对象的同时就创建了IIntentReceiver对象。**这样ReceiverDispatcher 同时保留了 BroadcastReceiver 和 InnerReceiver。Service 中也有一个类似的，ServiceDispatcher 中保留 ServiceConnection 和 InnerConnection**
      * 返回得到的IIntentReceiver对象即ServiceDispatcher.InnerReceiver对象。

2. 调用AMS的 registerReceiver()注册广播
   * ActivityManagerNative.getDefault() 是AMS，可以进行跨进程调用。
   * mMainThread.getApplicationThread()返回的是ApplicationThread，这是Binder的Bn端，用于system_server进程与该进程的通信，
   * rd 就是 ReceiverDispatcher.InnerReceiver,也是可以实现跨进程调用的。那就开始第一次跨进程吧

#### ActivityManagerService #registerReceiver()
```java
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
            ...
            //获取IntentFilter中的actions
            Iterator actions = filter.actionsIterator();
            ...
            //当IIntentReceiver为空，则直接返回第一个sticky Intent，
            Intent sticky = allSticky != null ? (Intent)allSticky.get(0) : null;
            if (receiver == null) {
                return sticky;
            }
            //mRegisteredReceivers 记录着所有已注册的广播，以receiver IBinder为key, ReceiverList为value为HashMap
            ReceiverList rl  = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                //对于没有注册的广播，则创建接收者队列
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app != null) {
                    rl.app.receivers.add(rl);
                } else {
                    try {
                        //注册死亡通知
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath = true;
                }
                //新创建的接收者队列，添加到已注册广播队列。
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            ...
            //创建BroadcastFilter对象，并添加到接收者队列
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);
            rl.add(bf);
            //新创建的广播过滤者，添加到ReceiverResolver队列
            mReceiverResolver.addFilter(bf);
            ...
            return sticky;
        }
    }
```
创建一个 创建BroadcastFilter对象，里面包含有你注册这个广播中的 IntentFilter 和 InnerReciver，packageName，requiredPermission，UID，和userid
并添加到 mReceiverResolver 这个队列中。

这样就完成了广播的注册过程。


小结
注册广播的过程，主要功能：
* 创建ReceiverList(接收者队列)，并添加到AMS.mRegisteredReceivers(已注册广播队列)；
* 创建BroadcastFilter(广播过滤者)，并添加到AMS.mReceiverResolver(接收者的解析人)；
* 当注册的是Sticky广播，则创建BroadcastRecord，并添加到BroadcastQueue mParallelBroadcasts(并行广播队列)，注册后调用AMS来尽快处理该广播。




来份简单的流程图
![Alt text](../../../../images/regeistreciver.png)

### sendBroadcast()

我们知道，注册广播方式有以上三种：
* 发送普通广播/无序广播 Context.sendBroadcast()
![Alt text](../../../../images/sendbroadcast.png)

* 发送有序广播 Context.sendOrderedBroadcast()
![Alt text](../../../../images/sendorderBroadcast.png)

* 发送粘性广播 Context.sendStickyBroadcast()
![Alt text](../../../../images/sendStickyBroadcast.png)

这三种方式的实现都是在ContextImpl这个类中，查看源码可知，这三种注册方式最终都是调用了ActivityManagerNative.getDefault().broadcastIntent()这个方法，如上图，注意我红框里面的
从上图可知，唯一不同的就是红框中的，serialized和sticky来共同决定是普通广播，有序广播，还是Sticky广播，而这两个boolean值就可以表示三种类型的广播,具体如下表格
![Alt text](../../../../images/broadcast_2.png)

#### AMS #broadcastIntent()
```java
public final int broadcastIntent(...) {
        enforceNotIsolatedCaller("broadcastIntent");
        synchronized(this) {
            //验证广播intent是否有效
            intent = verifyBroadcastLocked(intent);
            //获取调用者进程记录对象
            final ProcessRecord callerApp = getRecordForAppLocked(caller);
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            int res = broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null,
                    intent, resolvedType, resultTo,
                    resultCode, resultData, map, requiredPermission, appOp, serialized, sticky,
                    callingPid, callingUid, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }
```
然后执行到了broadcastIntentLocked()方法中,这个才是关键
#### AMS # broadcastIntentLocked（）

#####  设置flag
```java
intent = new Intent(intent);
//增加该flag，则广播不会发送给已停止的package
intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
```
在Android5.0的时候，默认添加这个Flag，表示广播不会发送给已经停止的应用，
Android3.1 添加了两个新的标志
* FLAG_EXCLUDE_STOPPED_PACKAGES  不包含已经停止的应用。这个时候广播不会发送给已停止的应用
* FLAG_INCLUDE_STOPPED_PACKAGES  包含已经停止的应用。这个时候广播会发送给已停止的应用

防止广播有意或者无意间调起已停止的应用。如果这两个FLAG共存的话，以FLAG_INCLUDE_STOPPED_PACKAGES为主。

一个应用处于停止状态由以下两种情形：
*  应用安装后未启动
* 被手动或者其他应用强行停止。

从Android3.1 开始，处于停止状态的应用无法接受到开机广播

发送广播之前,根据Intent-Filter查找匹配的广播并经过一系列的条件过滤，最终满足条件的广播接受者添加到BroadQueue中

##### 处理普通广播

```java
  ...
  List<BroadcastFilter> registeredReceivers = null;
  registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false, userId);
  ...

  //根据intent来得到是前台广播还是后台广播
  final BroadcastQueue queue = broadcastQueueForIntent(intent);
  //创建 BroadcastRecord 对象
  BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
          callerPackage, callingPid, callingUid, resolvedType, requiredPermission,
          appOp, registeredReceivers, resultTo, resultCode, resultData, map,
          ordered, sticky, false, userId);
  final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
  if (!replaced) {
      //将BroadcastRecord加入到并行广播队列
      queue.enqueueParallelBroadcastLocked(r);
      //处理广播
      queue.scheduleBroadcastsLocked();
  }
}
```
我们在registReceiver() 的时候，会创建BroadcastFilter对象，然后添加到这个mReceiverResolver，
1. 在发送广播的时候，就会根据传递过来的Intent，找到所有匹配的 BroadcastFilter，
2. 然后创建一个 BroadcastRecord对象，并把该 registeredReceivers 封装到里面，
3. 将得到的registeredReceivers 对象添加到 BroadcastQueue的无序广播队列 mParallelBroadcasts 中。
4. 执行scheduleBroadcastsLocked()

#### BroadcastQueue #scheduleBroadcastsLocked()
scheduleXXXX,根据之前的惯例，应该就是发送消息，切换到主线程中。然后执行 proformXXX,
```java
public void scheduleBroadcastsLocked() {
     // 正在处理BROADCAST_INTENT_MSG消息
     if (mBroadcastsScheduled) {
         return;
     }
     //发送BROADCAST_INTENT_MSG消息
     mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
     mBroadcastsScheduled = true;
 }

 final Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    processNextBroadcast(true);
                } break;
                ...
            }
        }
    };
```
果真是发送消息，不过执行的方法名称改了，叫 processNextBroadcast()
#### BroadcastQueue # processNextBroadcast()
```java
final void processNextBroadcast(boolean fromMsg) {
    //synchronized(),加锁，mService为AMS，锁对象唯一。全程持有AMS锁，
    //整个流程还是比较长的，所以广播效率低的情况下，
    //直接会严重影响这个手机的性能与流畅度，这里应该考虑细化同步锁的粒度。
    synchronized(mService) {
        //step1: 处理并行广播
        //step2: 处理当前有序广播
        //step3: 获取下条有序广播
        //step4: 处理下条有序广播
    }
}
```
step1: 处理并行广播
```java
BroadcastRecord r;
  //step1: 处理并行广播        
  while (mParallelBroadcasts.size() > 0) {
        r = mParallelBroadcasts.remove(0);
            。。。
        final int N = r.receivers.size();
        for (int i=0; i<N; i++) {
          Object target = r.receivers.get(i);
          //分发广播给已注册的receiver
          deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
      }
      addBroadcastToHistoryLocked(r);//将广播添加历史统计
  }
```
* while循环遍历，通过remove方法，每次都是得到队列头部的BroadcastRecord 对象，
* BroadcastRecord 对象中保存有所有的接受者，
* 然后再通过for循环遍历，通过deliverToRegisteredReceiverLocked()发送给每一位 BroadcastRecord的接受者，
* deliverToRegisteredReceiverLocked()里面会执行到 performReceiveLocked()中，还是有perform方法的，这个才是核心,而根据以往的经验，这里面会进行第二次跨进程通信，直到ApplicationThread中。

#### BroadcastQueue# performReceiveLocked()
```java
private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
      Intent intent, int resultCode, String data, Bundle extras,
      boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
      ...
      //调用ApplicationThreadProxy类对应的方法
      app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
              data, extras, ordered, sticky, sendingUser, app.repProcState);
      ...
}
```
果不其然，app.thread.scheduleRegisteredReceiver()。又到了熟悉的地方。那么接下来会轻松一些。

注意这时候的receiver，IIntentReceiver对象，也就是 LoadedApk.ReceiverDispatcher.InnerReceiver，


#### ActivityThread # scheduleRegisteredReceiver(),

```java
//LoadedApk.ReceiverDispatcher.java
public void performReceive(Intent intent, int resultCode, String data, Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
  // 创建一个Args对象，并通过mActivityThread方法执行，
  //mActivityThread 是一个Handler对象，Args实现了Runnale接口，最终执行的是run()方法
  Args args = new Args(intent, resultCode, data, extras, ordered, sticky, sendingUser);
  if (!mActivityThread.post(args)) {
    if (mRegistered && ordered) {
      IActivityManager mgr = ActivityManagerNative.getDefault();
      args.sendFinished(mgr);
    }
  }
}

final class Args extends BroadcastReceiver.PendingResult implements Runnable {
    public void run() {
      ...
      ClassLoader cl = mReceiver.getClass().getClassLoader();
        intent.setExtrasClassLoader(cl);
        setExtrasClassLoader(cl);
        receiver.setPendingResult(this);
        // onReceive执行
        receiver.onReceive(mContext, intent);
        ...
      }
}
```
1. InnerReceiver 中持有一个弱引用的 ReceiverDispatcher，所以可以通过 mDispatcher.get()得到 这个ReceiverDispatcher对象，
2. 然后执行到ReceiverDispatcher.performReceive(),
3. 因为 ReceiverDispatcher中持有BroadcastReciver对象，为了保证在主线程中执行，
4. 所以就通过post 一个Runnable对象Args 切换到主线程。然后执行 BroadcastRecord.onRecive()

在LoadedApk.ReceiverDispatcher # performReceive()，还有要finish的，如下图
![Alt text](../../../../images/performreceive.png)

流程如下

args.sendFinished(mgr) ->AMS.finishReceiver() -> BroadcastQueue.processNextBroadcast(false); 执行下一个广播。如果有的话

整个流程就是
```java
  receiver.performReceive()，在 ReceiverDispatcher 中执行，所以
  -> LoadedApk.ReceiverDispatcher rd = mDispatcher.get()
  -> rd.performReceive()
          -> post 一个Runnable对象Args-> Args.run() -> BroadcastRecord.onRecive()
          -> args.sendFinished(mgr) ->AMS.finishReceiver() -> BroadcastQueue.processNextBroadcast(false);
```
这个流程图如下

![Alt text](../../../../images/sendbroadcast_2.png)




---
搬运地址：    

Android 开发艺术探索
