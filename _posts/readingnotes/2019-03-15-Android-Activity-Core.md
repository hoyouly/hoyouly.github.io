---
layout: post
title: Android 四大组件之 Activity
category: 读书笔记
tags: Android开发艺术探索 Activity
---

* content
{:toc}
## 类名解释
### AMS
ActivityManagerService（简称AMS）继承自`ActivityManagerNative（简称AMN）`，而AMN继承自Binder并实现了IActivityManager这个Binder接口，因此AMS也是一个Binder，它是IActivityManager的具体实现，由于**ActivityManagerNative.getDefault()其实是一个IActivityManager类型的Binder对象，因此它的具体实现是AMS。**

```java
public final class ActivityManagerService extends ActivityManagerNative {...}

public abstract class ActivityManagerNative extends Binder implements IActivityManager{...}

public interface IActivityManager extends IInterface {...}
```
在AMN中，AMS这个Binder对象采用的是单例模式对外提供的，Singleton是一个单例的封装类，第一次调用它的get()方法会通过creat()方法初始化AMS这个Binder对象，在后续的调用中则直接返回之前创建的对象

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
			return in;//同一进程，返回Stub本地对象。
		}
		//从ServiceManager中获取AMS中Binder的引用对象，
    //然后将它转换成ActivityManagerProxy对象（简称AMP），AMP就是AMS的代理对象。
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
#### AMS的启动过程
在SystemServer中，通过main()方法，创建SystemServer对象并且调用了run()方法
```java
// SystemServer
public static void main(String[] args) {
	new SystemServer().run();
}
```
在run（）方法中会执行startBootstrapServices()方法，启动引导服务
在引导服务中，会创建AMS对象并且启动
```java
private void startBootstrapServices() {
	Installer installer = mSystemServiceManager.startService(Installer.class);

	// Activity manager runs the show.
	mActivityManagerService = mSystemServiceManager.startService(
                        ActivityManagerService.Lifecycle.class).getService();
	mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
	mActivityManagerService.setInstaller(installer);
	```
}
```
SystemServiceManager的startService()方法中会通过类加载器的方式，创建一个SystemService,并且调用开启该服务
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
		。。。
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
1. 根据类加载器创建SystemService子类对象
2. 把该SystemService子类对象添加到mServices中，也就是注册到mService中。      
`private final ArrayList<SystemService> mServices = new ArrayList<SystemService>();`
3. 执行onStart()方法

#### AMS.Lifecycle
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
1. 在构造函数中创建了一个AMS对象，所以上面在通过类加载器创建ActivityManagerService.Lifecycle对象的时候，AMS对象也就相应的创建了，在AMS中会创建ActivityStackSupervisor对象
```java
 public ActivityManagerService(Context systemContext) {
		。。。
		mStackSupervisor = new ActivityStackSupervisor(this);
		。。。
 }
```
因为个系统只启动一个AMS，那么也就只会创建一个ActivityStackSupervisor对象

2. 然后在onStart()中调用了AMS的start()方法，所以在上面执行onStart()的时候，AMS的start()方法也就相应执行了
3. mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class) 得到的是一个ActivityManagerService.Lifecycle，而这个对象的getService()返回的是一个AMS对象，所以最后得到的就是一个AMS对象即mActivityManagerService，


### ActivityThread
是主线程，也是UI线程，它是在APP启动时创建的，它代表了App应用程序
<font  color="#ff000"  >Context是Activity的上下文。Application是ActivityThread的上下文 </font>

#### ApplicationThread
是ActivityThread的一个内部类，是一个Binder对象，说明它的作用就是用于进程间通讯的Binder对象。

```java
public final class ActivityThread {//没有继承或者实现其他类。

    final ApplicationThread mAppThread = new ApplicationThread();

    public ApplicationThread getApplicationThread()
    {
        return mAppThread;
    }

    //ActivityThread的内部类ApplicationThread
    private class ApplicationThread extends ApplicationThreadNative {
    ......
    }
}
```
ApplicationThreadNative就是相当于AIDL通信中的Stub，也就是服务端。

ApplicationThreadProxy 就是相当于AIDL通信中的Proxy，也就是客户端。

所以`ApplicationThread是通信的具体实现类`

<font color="#ff000" >Activity的启动实际上是多次进程通信的成果，ActivityThread通过内部类ApplicationThread来进行进程间通信</font>
```java
public abstract class ApplicationThreadNative extends Binder implements IApplicationThread {

    static public IApplicationThread asInterface(IBinder obj) {...}

    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
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

#### H
H 类是ActivityThread的内部类，相当于ActivityThread和ApplicationThread的中介人，ActivityThread通过ApplicationThread与AMS通讯，
ApplicationThread通过H与ActivityThread通讯，处理Activity事务

H类存在的意义：
* 便于集中管理，方便打印Log 日志等，H就是这其中的管家
* ActivityThread通过ApplicationThread和AMS进行进程间通信，AMS以进程通讯的方式完成ActivityThread的请求后调用ApplicationThread的Binder方法，然后ApplicationThread会向H发送消息，H收到消息后会将ApplicationThread中的逻辑切换到ActivityThread中执行，即切换到主线程中执行，这个过程就是主线程的消息循环模型

这个ActivityThread并不是一个线程Thread，它是final类并且无继承或者实现其它类，它的作用就是在main方法内消息循环，处理主线程事务。（还需了解Looper及消息机制）

### ContextImpl
ContextImpl是一个很重要的数据结构，它是Context的具体实现，Context中的大部分逻辑都是由ContextImpl来完成的。

ContextImpl是通过Activity的attach()方法来和Activity建立关联的.

除此之外，在attach()方法中Activity还会完成Window的创建并建立自己和Window的关联，这样当Window接收到外部输入事件后就可以将事件传递给Activity。

### Instrumentation
管理Activity的一个工具类

包括创建和启动Activity，Activity的生命周期方法都是由Instrumentation这个仪器来控制

一个进程中只用一个Instrumentation实例

### ActivityRecord  
用来记录一个Activity的所有信息，归system_server进程使用，在ActivityStackSupervisor 中创建。

在AMS中，将用ActivityRecord来作为Activity的记录者,每次启动一个Actvity会有一个对应的ActivityRecord对象，表示Activity的一个记录
### ActivityClientRecord
用来记录一个Activity的所有信息，归App进程使用。在ApplicationThread中创建，并添加到了ActivityThread的mActivities中
```java
final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<IBinder, ActivityClientRecord>();
```
这个IBinder 就是一个Token，在scheduleLaunchActivity()的时候，通过ActivityRecord.apptoken传递过来的，而这个appToken，是在ActivityRecord的构造函数中创建的
```java
//ActivityRecord
ActivityRecord(ActivityManagerService _service ...) {
		service = _service;
		appToken = new Token(this);
    ...
}
```
也就是每一个ActivityRecord都会有一个appToken ，而ActivityRecord 创建是在AMS中startActivityLocked()中，这个是在我们startActivity()的时候会调用到的，这个后面会讲到

总结一下：
我们startActivity的时候，会调用到AMS的startActivityLocked()，然后创建一个ActivityRecord 供ActivityStackSupervisor使用，这里面会有一个appToken ,然后在 ApplicationThread的scheduleLaunchActivity()时候，把ActivityRecord转换成ActivityClientRecord，然后放到集合mActivities中。key 就是appToken.

### TaskRecord
Activity栈，内部维护一个`ArrayList<ActivityRecord>`。

App启动一个Activity，会不会新建一个TaskRecord取决于launchMode，launchMode 详情查看 [Activity 的生命周期和启动模式](../../../../2018/03/17/Activity-lifecycle-task)。  默认的standard模式不会创建新的TaskRecord
### ActivityStack
并不是一个Activity栈，真正意义上的Activity栈是TaskRecord。

这个类是负责管理各个Activity栈，内部维护一个`ArrayList<TaskRecord>`
### ActivityStackSupervisor  
管理整个手机任务栈。

内部持有一个ActivityStack，而ActivityStack内部也持有ActivityStackSupervisor，相当于ActivityStack的辅助管理类。

**总结：**
* 每一个ActivityRecord都会有一个Activity与之对应，
* 一个Activity可能会有多个ActivityRecord，因为Activity可以被多次实例化，取决于其launchmode。
* 一系列相关的ActivityRecord组成了一个TaskRecord，
* TaskRecord是存在于ActivityStack中，
* ActivityStackSupervisor是用来管理这些ActivityStack的。
* 整个系统中只有一个AMS，在AMS的构造函数中创建了 ActivityStackSupervisor，所以ActivityStackSupervisor的 也只有一个。
* 一个应用程序中只有一个 Instrumentation 对象
* ActivityStack中会保存有要启动的Activity的信息，即ActivityRecord，

![Alt text](../../../../images/activityrecord.png)

## Activity 工作流程
#### Activity # startActivity()
这一次不贴代码了，只看图。
先来一张流程图：
![Alt text](../../../../images/startActivity.png)

我们需要记住以下几点即可
1. startActivity()有好几种重载，最终调用到了startActivityForResult()。然后执行到了 Instrumentation.execStartActivity()。
2. 启动Activity的真正实现是由ActivityManagerNative.getDefault()的startActivity()方法来完成的。这个ActivityManagerNative.getDefault() 就是AMS。<font color="#ff000" > 第一次跨进程。</font>
3. AMS.startActivity() 执行到 最终会执行到 ActivityStackSupervisor的 realStartActivityLocked()
4. realStartActivityLocked()会通过 ProcessRecord.thread.scheduleLaunchActivity()。ProcessRecord.thread 就是 ApplicationThread。<font color="#ff000" > 第二次跨进程调用。</font>
5. Activity的生命周期几个方法，可以使用这个套路啊,虽然不全是但是也差不离的，后面会具体说到，是不是这样。自己往里套。
```java
ApplicationThread.schedule***Activity()
->ActivityThread.handle***Activity()->ActivityThread.perform***Activity()
->Instrument.callActivityOn***()
->Activity.perform***()->Activity.on***()
```
6. Activity A 中启动Activity B ,流程是 : A  onPause()-> B  onCreat()-> B onStart() -> B onResume() -> A onStop()

#### ApplicationThread # schedulePauseActivity()
流程图走起
![Alt text](../../../../images/schedulePauseActivity.png)

用箭头表示就是如下：
```java
ApplicationThread # schedulePauseActivity()
->通过H发送消息
->ActivityThread.handlePauseActivity()->ActivityThread.performPauseActivity()
      1.->Instrumentation.callActivityOnSaveInstanceState()->Activity#preformOnSaveInstanceState()->Activity#onOnSaveInstanceState()
      2.->Instrumentation.callActivityOnPause()->Activity#preformPause()->Activity#onPause()
```
注意是先执行 onOnSaveInstanceState(),在执行 onPause()
#### ApplicationThread # scheduleLaunchActivity()
流程图如下
![Alt text](../../../../images/scheduleLaunchActivity.png)
用箭头表示就是如下：

```java
ApplicationThread # scheduleLaunchActivity()
->通过H发送消息
->ActivityThread.handleLaunchActivity()
    1.->ActivityThread.performLaunchActivity()
    2.->ActivityThread.handleResumeActivity() ->ActivityThread.performResumeActivity()
```

performLaunchActivity() 和 performResumeActivity()后面会详细说

接下来我们先看performLaunchActivity()的流程
#### ActivityThread # performLaunchActivity()
![Alt text](../../../../images/performLunchActivity.png)
可以猜出来，这里面会执行通过 Instrument 执行 Activity的onCreat(),onStart()方法

但是我们没想到的是

这里面还如果Application对象 不存在的话，创建了Application 对象，并且执行了Application的onCreat(),

以及通过Instrument 创建Activity对象，创建ContextImpl对象等等

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

#### ActivityThread # handleResumeActivity()
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
2. 执行 ApplicationThread 的scheduleStopActivity(),这里面我们就不详说了，肯定发送消息通知ActivityThread，然后执行Activity的perforStopActivity(),最后通过Instrument 执行上一个Activity的onStop()方法，不过这也符合整个生命周期跳转流程

### AMS # startProcessLocked()
这里面还有一个没说到，就是如果是首次启动某个应用，那么这个进程都不存在，又进行了什么操作呢？
将会通过AMS.startProcessLocked()创建一个进程，里面的操作就是就是通过socket发送消息给zygote,zygote将派生出一个子进程，
子进程将通过反射调用ActivityThread的main()。具体的流程图如下
![Alt text](../../../../images/startProcessLocked.png)

通过反射调用了ActivityThread.main()方法，main()方法开始执行，
* 创建Looper对象，
* 创建ActivityThread
* 调用attch()绑定到AMS上去,在这里会得到AMS对象，然后执行AMS的attachApplicationLocked()方法。
* 开启Looper循环消息。
我们熟悉的 Looper就不说了，只说 ActivityThread创建成功后。调用attch()得到AMS之后，执行的 AMS的attachApplicationLocked()

### AMS # attachApplicationLocked()
流程图如下
![Alt text](../../../../images/attachApplicationLocked.png)
注意两点即可
1. 通过ApplicationThread的bindApplication()方法 绑定Application ，还是通过发送handler消息，执行到ActivityThread中的handleApplication(),这个才是重点
2. 创建Application之后，这样Application就启动了，就可以启动Activity，执行了ActivityStackSupervisor的attachApplicationLocked(app)，它会调用realStartActivityLocked(),这就到了上面那个流程图里面了

### ActivityThread # handleApplication()
![Alt text](../../../../images/handleBindApplication.png)
重点：
1. 使用类加载器创建Instrument对象，handleApplication()只在进程创建的时候调用一次，一个应用中也只有一个Instrument对象。
2. ContentProvider 的onCreate()比Application 的onCreat()执行的更早

### 启发
通过上面分析，我忽然灵光乍现了一下。以前一直为起方法名称头疼，这不有现成的例子吗，照猫画虎呗。以后遇到这种发消息的情况，就这么办啊。
* 发送消息的方法就叫 scheduleXXX
* handleMessage()中接收消息的方法名就叫 handleXXX
* 真正处理消息的方法名称就叫： performXXX  (perform 执行)
Google大神就是这样干的，绝对没错啊。
而且好像Android 源码中有好多这样开头的方法名，例如View绘制的时候的 scheduleTraversals()->doTraversals()->performTraversals。


---
搬运地址：   
Android 开发艺术探索
