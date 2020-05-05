---
layout: post
title: Android 四大组件之 Activity
category: 读书笔记
tags: Android开发艺术探索 Activity
---

<!-- * content -->
<!-- {:toc} -->
# 类名解释
## AMS
ActivityManagerService（简称AMS）继承自`ActivityManagerNative（简称AMN）`，而 AMN 继承自 Binder 并实现了 IActivityManager 这个 Binder 接口，因此 AMS 也是一个 Binder ，它是 IActivityManager 的具体实现。    
由于**ActivityManagerNative.getDefault()其实是一个 IActivityManager 类型的 Binder 对象，因此它的具体实现是 AMS 。**

```java
public final class ActivityManagerService extends ActivityManagerNative {...}

public abstract class ActivityManagerNative extends Binder implements IActivityManager{...}

public interface IActivityManager extends IInterface {...}
```
在 AMN 中， AMS 这个 Binder 对象采用的是单例模式对外提供， Singleton 是一个单例的封装类，第一次调用它的 get() 方法会通过 create() 方法初始化 AMS 这个 Binder 对象，在后续的调用中则直接返回之前创建的对象

```java
// AMN
static public IActivityManager getDefault() {
  return gDefault.get();
}

private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        IActivityManager am = asInterface(b);
        return am;
    }
};

static public IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
      return null;
    }
    IActivityManager in = (IActivityManager) obj.queryLocalInterface(descriptor);
    if (in != null) {
      return in;//同一进程，返回 Stub 本地对象。
    }
    //从 ServiceManager 中获取 AMS 中 Binder 的引用对象，
    //然后将它转换成 ActivityManagerProxy 对象（简称AMP）， AMP 就是 AMS 的代理对象。
    return new ActivityManagerProxy(obj);//跨进程，返回代理对象。
  }

//Singleton
public abstract class Singleton<T> {
  private T mInstance;
  protected abstract T create();
  public final T get() {
    synchronized (this) {
      if (mInstance == null) {
        mInstance = create();
      }
      return mInstance;
    }
  }
}
```
### AMS的启动过程
在 SystemServer 中，通过 main() 方法，创建 SystemServer 对象并且调用了 run() 方法
```java
// SystemServer
public static void main(String[] args) {
  new SystemServer().run();
}
```
在run（）方法中会执行 startBootstrapServices() 方法启动引导服务。
在引导服务中，会创建 AMS 对象并且启动
```java
private void startBootstrapServices() {
  Installer installer = mSystemServiceManager.startService(Installer.class);

  // Activity manager runs the show.
  mActivityManagerService = mSystemServiceManager.startService(
                        ActivityManagerService.Lifecycle.class).getService();
  mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
  mActivityManagerService.setInstaller(installer);
  ...
}
```
SystemServiceManager 的 startService() 方法中会通过类加载器的方式，创建一个 SystemService ,并且调用开启该服务
```java
public SystemService startService(String className) {
  final Class<SystemService> serviceClass;
  try {
    serviceClass = (Class<SystemService>) Class.forName(className);
  } catch (ClassNotFoundException ex) {
    ...
  }
  return startService(serviceClass);
}

public <T extends SystemService> T startService(Class<T> serviceClass) {
  final String name = serviceClass.getName();

  // Create the service.
  if (!SystemService.class.isAssignableFrom(serviceClass)) {
    throw new RuntimeException("Failed to create " + name + ": service must extend " + SystemService.class.getName());
  }
  final T service;
  try {
    Constructor<T> constructor = serviceClass.getConstructor(Context.class);
    service = constructor.newInstance(mContext);
  } catch (InstantiationException ex) {
    ...
  }

  // Register it.
  mServices.add(service);

  // Start it.
  try {
    service.onStart();
  } catch (RuntimeException ex) {
    throw new RuntimeException("Failed to start service " + name + ": onStart threw an exception", ex);
  }
  return service;
}
```
1. 根据类加载器创建 SystemService 子类对象
2. 把该 SystemService 子类对象添加到 mServices 中，也就是注册到 mService 中。      
`private final ArrayList<SystemService> mServices = new ArrayList<SystemService>();`
3. 执行 onStart() 方法

### AMS.Lifecycle
然后我们再查看ActivityManagerService.Lifecycle 是个啥玩意
```java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;
    public Lifecycle(Context context) {
      super(context);
      mService = new ActivityManagerService(context);
    }
    @Override
    public void onStart() {
      mService.start();
    }
    public ActivityManagerService getService() {
      return mService;
    }
  }
```
这个类很简单
1. 在构造函数中创建了一个 AMS 对象，所以上面在通过类加载器创建ActivityManagerService.Lifecycle对象的时候， AMS 对象也就相应的创建了，在 AMS 中会创建 ActivityStackSupervisor 对象
```java
 public ActivityManagerService(Context systemContext) {
    ...
    mStackSupervisor = new ActivityStackSupervisor(this);
    ...
 }
```
因为一个系统只启动一个 AMS ，那么也就只会创建一个 ActivityStackSupervisor 对象

2. 然后在 onStart() 中调用了 AMS 的 start() 方法，所以在上面执行 onStart() 的时候， AMS 的 start() 方法也就相应执行了

总结： mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class) 得到的是一个ActivityManagerService.Lifecycle，而这个对象的 getService() 返回的是一个 AMS 对象，所以最后得到的就是一个 AMS 对象即 mActivityManagerService ，


## ActivityThread
是主线程，也是 UI 线程，它是在 APP 启动时创建的，它代表了 App 应用程序
<font  color="#ff000"  >Context是 Activity 的上下文。 Application 是 ActivityThread 的上下文 </font>

### ApplicationThread
ActivityThread 的一个内部类，是一个 Binder 对象，说明它的作用就是用于进程间通讯的 Binder 对象。

```java
public final class ActivityThread {//没有继承或者实现其他类。

    final ApplicationThread mAppThread = new ApplicationThread();

    public ApplicationThread getApplicationThread()
    {
        return mAppThread;
    }

    //ActivityThread的内部类ApplicationThread
    private class ApplicationThread extends ApplicationThreadNative {
    ...
    }
}
```
ApplicationThreadNative 就是相当于 AIDL 通信中的 Stub ，也就是服务端。

ApplicationThreadProxy 就是相当于 AIDL 通信中的 Proxy ，也就是客户端。

所以`ApplicationThread是通信的具体实现类`

<font color="#ff000" >Activity的启动实际上是多次进程通信的成果， ActivityThread 通过内部类 ApplicationThread 来进行进程间通信</font>
```java
public abstract class ApplicationThreadNative extends Binder implements IApplicationThread {

    static public IApplicationThread asInterface(IBinder obj) {...}

    @Override
    public boolean onTransact(int code, Parcel data , Parcel reply , int flags)
            throws RemoteException {...}

    public IBinder asBinder(){return this;}
}

class ApplicationThreadProxy implements IApplicationThread {
    private final IBinder mRemote;

    public ApplicationThreadProxy(IBinder remote) {
        mRemote = remote;
    }
    ...
}
```

### H
H 类是 ActivityThread 的内部类，相当于 ActivityThread 和 ApplicationThread 的中介人， ActivityThread 通过 ApplicationThread 与 AMS 通讯，
ApplicationThread 通过 H 与 ActivityThread 通讯，处理 Activity 事务

H 类存在的意义：
* 便于集中管理，方便打印 Log 日志等， H 就是这其中的管家
* ActivityThread通过 ApplicationThread 和 AMS 进行进程间通信， AMS 以进程通讯的方式完成 ActivityThread 的请求后调用 ApplicationThread 的 Binder 方法，然后 ApplicationThread 会向 H 发送消息， H 收到消息后会将 ApplicationThread 中的逻辑切换到 ActivityThread 中执行，即切换到主线程中执行，这个过程就是主线程的消息循环模型

这个 ActivityThread 并不是一个线程 Thread ，它是 final 类并且无继承或者实现其它类，它的作用就是在 main 方法内消息循环，处理主线程事务。（还需了解 Looper 及消息机制）

## ContextImpl
ContextImpl 是一个很重要的数据结构，它是 Context 的具体实现， Context 中的大部分逻辑都是由 ContextImpl 来完成的。

ContextImpl 是通过 Activity 的 attach() 方法来和 Activity 建立关联的.

除此之外，在 attach() 方法中 Activity 还会完成 Window 的创建并建立自己和 Window 的关联，这样当 Window 接收到外部输入事件后就可以将事件传递给 Activity 。

## Instrumentation
管理 Activity 的一个工具类

包括创建和启动 Activity ， Activity 的生命周期方法都是由 Instrumentation 这个仪器来控制

一个进程中只用一个 Instrumentation 实例

## ActivityRecord  
用来记录一个 Activity 的所有信息，归 system_server 进程使用，在 ActivityStackSupervisor 中创建。

在 AMS 中，将用 ActivityRecord 来作为 Activity 的记录者,每次启动一个 Actvity 会有一个对应的 ActivityRecord 对象，表示 Activity 的一个记录
## ActivityClientRecord
用来记录一个 Activity 的所有信息，归 App 进程使用。在 ApplicationThread 中创建，并添加到了 ActivityThread 的 mActivities 中
```java
final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<IBinder, ActivityClientRecord>();
```
这个 IBinder 就是一个 Token ，在 scheduleLaunchActivity() 的时候，通过ActivityRecord.apptoken传递过来的，而这个 appToken ，是在 ActivityRecord 的构造函数中创建的
```java
//ActivityRecord
ActivityRecord(ActivityManagerService _service ...) {
    service = _service;
    appToken = new Token(this);
    ...
}
```
也就是每一个 ActivityRecord 都会有一个 appToken ，而 ActivityRecord 创建是在 AMS 中 startActivityLocked() 中，这个是在我们 startActivity() 的时候会调用到的，这个后面会讲到

总结一下：
我们 startActivity 的时候，会调用到 AMS 的 startActivityLocked() ，然后创建一个 ActivityRecord 供 ActivityStackSupervisor 使用，这里面会有一个 appToken ,然后在 ApplicationThread 的 scheduleLaunchActivity() 时候，把 ActivityRecord 转换成 ActivityClientRecord ，然后放到集合 mActivities 中。 key 就是appToken.

## TaskRecord
Activity 栈，内部维护一个`ArrayList<ActivityRecord>`。

App 启动一个 Activity ，会不会新建一个 TaskRecord 取决于 launchMode ， launchMode 详情查看 [Activity 的生命周期和启动模式](../../../../2018/03/17/Activity-lifecycle-task)。  默认的 standard 模式不会创建新的TaskRecord
## ActivityStack
并不是一个 Activity 栈，真正意义上的 Activity 栈是 TaskRecord 。

这个类是负责管理各个 Activity 栈，内部维护一个`ArrayList<TaskRecord>`
## ActivityStackSupervisor  
管理整个手机任务栈。

内部持有一个 ActivityStack ，而 ActivityStack 内部也持有 ActivityStackSupervisor ，相当于 ActivityStack 的辅助管理类。

**总结：**
* 每一个 ActivityRecord 都会有一个 Activity 与之对应，
* 一个 Activity 可能会有多个 ActivityRecord ，因为 Activity 可以被多次实例化，取决于其 launchmode 。
* 一系列相关的 ActivityRecord 组成了一个 TaskRecord ，
* TaskRecord是存在于 ActivityStack 中，
* ActivityStackSupervisor是用来管理这些 ActivityStack 的。
* 整个系统中只有一个 AMS ，在 AMS 的构造函数中创建了 ActivityStackSupervisor ，所以 ActivityStackSupervisor 的 也只有一个。
* 一个应用程序中只有一个 Instrumentation 对象
* ActivityStack中会保存有要启动的 Activity 的信息，即 ActivityRecord ，

![Alt text](../../../../images/activityrecord.png)

# Activity 工作流程
### Activity # startActivity()
先不贴代码，来张流程图再说：
![Alt text](../../../../images/startActivity.png)

我们需要记住以下几点即可
1. startActivity()有好几种重载，最终调用到了 startActivityForResult() 。然后执行到了 Instrumentation.execStartActivity()。
2. 启动 Activity 的真正实现是由ActivityManagerNative.getDefault()的 startActivity() 方法来完成的。这个ActivityManagerNative.getDefault() 就是 AMS 。<font color="#ff000" > 第一次跨进程。</font>
3. AMS.startActivity() 最终会执行到 ActivityStackSupervisor 的 realStartActivityLocked()
4. realStartActivityLocked()会通过 ProcessRecord.thread.scheduleLaunchActivity()。ProcessRecord.thread 就是 ApplicationThread 。<font color="#ff000" > 第二次跨进程调用。</font>
5. Activity的生命周期几个方法，可以使用这个套路啊,虽然不全是但是也差不离的，后面会具体说到，是不是这样。自己往里套。
```java
ApplicationThread.scheduleXXXActivity()
->ActivityThread.handleXXXActivity()->ActivityThread.performXXXActivity()
->Instrument.callActivityOnXXX()
->Activity.performXXX()->Activity.onXXX()
```
6. Activity A 中启动 Activity B ,流程是 : A  onPause()-> B  onCreate()-> B onStart() -> B onResume() -> A onStop()

### ApplicationThread # schedulePauseActivity()
流程图走起
![Alt text](../../../../images/schedulePauseActivity.png)

用箭头表示就是如下：
```java
ApplicationThread # schedulePauseActivity()
->通过 H 发送消息
->ActivityThread.handlePauseActivity()->ActivityThread.performPauseActivity()
      1.->Instrumentation.callActivityOnSaveInstanceState()->Activity#preformOnSaveInstanceState()->Activity#onOnSaveInstanceState()
      2.->Instrumentation.callActivityOnPause()->Activity#preformPause()->Activity#onPause()
```
注意是先执行 onOnSaveInstanceState() ,在执行 onPause()
### ApplicationThread # scheduleLaunchActivity()
流程图如下
![Alt text](../../../../images/scheduleLaunchActivity.png)
用箭头表示就是如下：

```java
ApplicationThread # scheduleLaunchActivity()
->通过 H 发送消息
->ActivityThread.handleLaunchActivity()
    1.->ActivityThread.performLaunchActivity()
    2.->ActivityThread.handleResumeActivity() ->ActivityThread.performResumeActivity()
```

performLaunchActivity() 和 performResumeActivity() 后面会详细说

接下来我们先看 performLaunchActivity() 的流程
### ActivityThread # performLaunchActivity()
![Alt text](../../../../images/performLunchActivity.png)
可以猜出来，这里面会执行通过 Instrument 执行 Activity 的 onCreat() , onStart() 方法

但是我们没想到的是

这里面还如果 Application 对象 不存在的话，创建了 Application 对象，并且执行了 Application 的 onCreat() ,

以及通过 Instrument 创建 Activity 对象，创建 ContextImpl 对象等等

里面红色的字体都是我们比较熟悉的方法吧

箭头表示入下

```java
ActivityThread.performLaunchActivity()
    1.->Instrumentation.newActivity()
    2.->Instrumentation.newApplication()->Instrumentation.callApplicationOnCreate ->Application.onCreate()
    3.->Activity.attch()
    4.->Instrumentation.callActivityOnCreate() -> Activity.performCreate()-> Activity.onCreate()
    5.->Activity.performStart()->Instrumentation.callActivityOnCreate()-> Activity.onStart()  //有点不按照套路出牌
    6.->Instrumentation.callActivityOnRestoreInstanceState() -> Activity.performRestoreInstanceState()-> Activity.onRestoreInstanceState()
```

### ActivityThread # handleResumeActivity()
我们大概可以猜出来里面的流程，举一反三呗
```java
perforResumeActivity()
->Instrument.callActivityOnResume()
->Activity.performOnResume() -> Activity.onResume()
```

然而事实是：
![Alt text](../../../../images/handleResumeActivity.png)
意外的点：
1. 竟然是<span style="border-bottom:1px solid red;"> Activity.preformOnResume()-> Instrument.callActivityOnResume()-> Activity.onResume()</span>,又不按照套路出牌啊。为毛线啊？没想明白
2. 执行 ApplicationThread 的 scheduleStopActivity() ,这里面我们就不详说了，肯定发送消息通知 ActivityThread ，然后执行 Activity 的 perforStopActivity() ,最后通过 Instrument 执行上一个 Activity 的 onStop() 方法，不过这也符合整个生命周期跳转流程

## AMS # startProcessLocked()
这里面还有一个没说到，就是如果是首次启动某个应用，那么这个进程都不存在，又进行了什么操作呢？
将会通过AMS.startProcessLocked()创建一个进程，里面的操作就是就是通过 socket 发送消息给 zygote , zygote 将派生出一个子进程，
子进程将通过反射调用 ActivityThread 的 main() 。具体的流程图如下
![Alt text](../../../../images/startProcessLocked.png)

通过反射调用了ActivityThread.main()方法， main() 方法开始执行，
* 创建 Looper 对象，
* 创建ActivityThread
* 调用 attch() 绑定到 AMS 上去,在这里会得到 AMS 对象，然后执行 AMS 的 attachApplicationLocked() 方法。
* 开启 Looper 循环消息。
我们熟悉的 Looper 就不说了，只说 ActivityThread 创建成功后。调用 attch() 得到 AMS 之后，执行的 AMS 的attachApplicationLocked()

## AMS # attachApplicationLocked()
流程图如下
![Alt text](../../../../images/attachApplicationLocked.png)
注意两点即可
1. 通过 ApplicationThread 的 bindApplication() 方法 绑定 Application ，还是通过发送 handler 消息，执行到 ActivityThread 中的 handleApplication() ,这个才是重点
2. 创建 Application 之后，这样 Application 就启动了，就可以启动 Activity ，执行了 ActivityStackSupervisor 的attachApplicationLocked(app)，它会调用 realStartActivityLocked() ,这就到了上面那个流程图里面了

## ActivityThread # handleApplication()
![Alt text](../../../../images/handleBindApplication.png)
重点：
1. 使用类加载器创建 Instrument 对象， handleApplication() 只在进程创建的时候调用一次，一个应用中也只有一个 Instrument 对象。
2. ContentProvider 的 onCreate() 比 Application 的 onCreat() 执行的更早

## 启发
通过上面分析，我忽然灵光乍现了一下。以前一直为起方法名称头疼，这不有现成的例子吗，照猫画虎呗。以后遇到这种发消息的情况，就这么办啊。
* 发送消息的方法就叫 scheduleXXX
* handleMessage()中接收消息的方法名就叫 handleXXX
* 真正处理消息的方法名称就叫： performXXX  (perform 执行)。     
Google 大神就是这样干的，绝对没错啊。
而且好像 Android 源码中有好多这样开头的方法名，例如 View 绘制的时候的 scheduleTraversals()->doTraversals()->performTraversals。


---
搬运地址：    

Android 开发艺术探索
