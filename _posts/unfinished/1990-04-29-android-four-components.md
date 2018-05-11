---
layout: post
title:  扫盲系列之---Android 四大组件
category:  待整理
tags:  Android Activity Service BroadCast ContentProvider
---
* content
{:toc}

Android四大组件中每个组件的作用是什么？它们都可以开启多进程吗？

## 四大组件的运行状态
### 共性
1. 注册方式上。除了BroadcastReceiver，其他三种组件必须在Android-Manifest中注册，BroadcastReceiver，即可以在Android-Manifest中注册，也可以通过代码注册
2. 在调用方式上。Activity，Service和BroadCastReceiver需要借助Intent，而ContentProvider无需借助Intent
3. 用户感知上，除了Activity，其他三个组件对用户来说都不是可感知的
4. Activity A 跳转到Activity B ，如果 Activity 是一个

### Activity 展示型组件
用于向用户直接展示一个界面，并且可以接受用户的输入信息从而进行交互，
启动由Intent出发，分为两种
* 显示Intent 明确指向一个Activity组件
* 隐式Intent 指向一个或者多个目标Activity组件

一个Activity组件具有特定的启动模式，
通过Activity的finish方法结束一个Activity组件运行。
主要作用是展示一个界面并和用户交互，扮演一种前台界面的角色，

### Service 计算性组件
用于后台执行一系列计算任务，尽管在后台执行，但是本身是运行在主线程中 的，因此**耗时的后台计算任然需要在单独的线程中完成** 用户无法直接感知到它的存在，有两种状态。`都可以进行后台计算，并且可以共存`
#### startService() 启动状态
* 不能和外界有直接交互
* 停止使用stopService()，如果没有调用 stopService()，Service会一直在后台运行
* 多次调用startService()，该Service只能创建一次，onCreat()只会被调用一次
* 每次调用startService()，onStartCommand()方法都会被调用
* 生命周期：**startService() -> onCreat()->  onStartCommand()->服务启动->stopService() ->onDestory()**

#### bindService() 绑定状态
 * 外界可以很方便的和Service组件进行通信，
 * 停止使用unBindService()，或者调用中Context不存在了（例如Activity的finish了）,**绑定即两者共存亡**，Service就会调用onUnbind()->onDestory()，
 * 第一次执行bindService()时，onCreate()和onBind()方法会被调用，
 * 多次执行bindService()时，onCreate()和onBind()方法并不会被多次调用
 * 生命周期： **bindService() -> onCreat()-> onBind()-> 服务启动 -> onUnbind() -> onDestory()**

#### 即使用bindService()又使用startService()
如果即使用了bindService()又使用startService()，该Service会一直在后台运行
1. onCreat()始终只会调用一次
2. 停止服务需要 unbindService()和stopService()同时调用才行，不论先后

### BroadcastReceiver  消息型组件
用于在不同组件乃至不同应用直接传递消息，广播注册有两种方式，
* 静态注册  在Android-Manifest中注册广播，应用安装的时候被系统解析，不需要应用启动就可以收到相应的广播。
* 动态注册  通过代码注册，Context.registerRecevier()来实现，并且在不需要的时候通过Context.unregisterReceiver()来解除广播，应用必须启动才能注册接受广播，
发送和接收过程的匹配是通过广播接收者的`<intent-filter>`来描述
可以用来实现低耦合的观察者模式，不适合执行耗时操作
一般不需要停止，

### ContentProvider  数据共享性组件
用于向其他组件乃至其他应用共享数据。用户无法直接感知。需要实现CRUD四种操作，在它内部维持着一份数据集合，这个数据集合即可以通过数据库来实现，也可以采用其他类型来实现，比如List和Map，
ContentProvider 内部的CURD需要处理号线程同步，因为这几个方法在Binder线程迟被调用，不需要手动停止

## Activity 的工作过程

### AMS
ActivityManagerService（简称AMS）继承自`ActivityManagerNative（简称AMN）`，而AMN继承自Binder并实现了IActivityManager这个Binder接口，因此AMS也是一个Binder，它是IActivityManager的具体实现，由于**ActivityManagerNative.getDefault()其实是一个IActivityManager类型的Binder对象，因此它的具体实现是AMS。**

```java
public final class ActivityManagerService extends ActivityManagerNative {...}

public abstract class ActivityManagerNative extends Binder implements IActivityManager{...}

public interface IActivityManager extends IInterface {...}
```
在AMN中，AMS这个Binder对象采用的是单例模式对外提供的，Singleton是一个单例的封装类，第一次调用它的get方法会通过creat方法初始化AMS这个Binder对象，在后续的调用中则直接返回之前创建的对象

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
		//从ServiceManager中获取AMS中Binder的引用对象，然后将它转换成ActivityManagerProxy对象（简称AMP），AMP就是AMS的代理对象。
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
在run（）方法中会执行startBootstrapServices（）方法，启动引导服务
在引导服务中，会创建AMS对象并且启动
```java
private void startBootstrapServices() {
		Installer installer = mSystemServiceManager.startService(Installer.class);

		// Activity manager runs the show.
		mActivityManagerService = mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class).getService();
		mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
		mActivityManagerService.setInstaller(installer);
		```
	}
```
SystemServiceManager的startService（）方法中会通过类加载器的方式，创建一个SystemService,并且调用开启该服务
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
		Slog.i(TAG, "Starting " + name);

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
2. 把该SystemService子类对象添加到mServices中，也就是注册到mServices
`private final ArrayList<SystemService> mServices = new ArrayList<SystemService>();`
3. 执行onStart()方法

##### AMS.Lifecycle
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
这个类很简单，
1. 在构造函数中创建了一个AMS对象，所以上面在通过类加载器创建ActivityManagerService.Lifecycle对象的时候，AMS对象也就相应的创建了，在AMS中会创建ActivityStackSupervisor对象
```java
 public ActivityManagerService(Context systemContext) {
		。。。
		mStackSupervisor = new ActivityStackSupervisor(this);
		。。。
 }
```
所以一个系统只启动一个AMS，那么也就只会创建一个ActivityStackSupervisor对象

2. 然后在onStart()中调用了AMS的start()方法，所以在上面执行onStart（）的时候，AMS的start()方法也就相应执行了
3. mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class) 得到的是一个ActivityManagerService.Lifecycle，而这个对象的getService（）返回的是一个AMS对象，所以最后得到的就是一个AMS对象即mActivityManagerService，


### ActivityThread
 是主线程，也是UI线程，它是在APP启动时创建的，它代表了App应用程序
`Context是Activity的上下文
Application是ActivityThread的上下文`

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
ApplicationThreadProxy即AIDL通信中的Proxy，也就是客户端。所以`ApplicationThread是通信的具体实现类`

`Activity的启动实际上是多次进程通信的成果，ActivityThread通过内部类ApplicationThread来进行进程间通信`
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
ContextImpl是一个很重要的数据结构，它是Context的具体实现，Context中的大部分逻辑都是由ContextImpl来完成的。ContextImpl来完成的。ContextImpl是通过Activity的attach方法来和Activity建立关联的，除此之外，在attach方法中Activity还会完成Window的创建并建立自己和Window的关联，这样当Window接收到外部输入事件后就可以将事件传递给Activity。

### Instrumentation
Instrumentation  程序中是管理activity的一个工具类，包括创建和启动Activity，activity的生命周期方法都是由Instrumentation这个仪器来控制，一个进程中只用一个Instrumentation实

### ActivityRecord  
用来记录一个Activity的所有信息，在AMS中，将用ActivityRecord来作为Activity的记录者,每次启动一个Actvity会有一个对应的ActivityRecord对象，表示Activity的一个记录
### TaskRecord
Activity栈，内部维护一个`ArrayList<ActivityRecord>`。
App启动一个Activity，会不会新建一个TaskRecord取决于launchMode，默认的standard模式不会创建新的TaskRecord
### ActivityStack
并不是一个Activity栈，真正意义上的Activity栈是TaskRecord，这个类是负责管理各个Activity栈，内部维护一个`ArrayList<TaskRecord>`
### ActivityStackSupervisor  
管理整个手机任务栈。内部持有一个ActivityStack，而ActivityStack内部也持有ActivityStackSupervisor，相当于ActivityStack的辅助管理类。

**总结：**
* 每一个ActivityRecord都会有一个Activity与之对应，
* 一个Activity可能会有多个ActivityRecord，因为Activity可以被多次实例化，取决于其launchmode。
* 一系列相关的ActivityRecord组成了一个TaskRecord，
* TaskRecord是存在于ActivityStack中，
* ActivityStackSupervisor是用来管理这些ActivityStack的。

![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCEd8a8531068680659aee22271dde24ae8/2561)


## Brodcast
### 静态注册广播
在应用安装的时候，由系统自动完成组成，具体来说是由PMS（PackageManagerService）来完成整个注册过程的，除了广播以外，其他三大组件也都是**在应用安装时由PMS解析并注册的**


广播分为三类：
* 普通广播
* 有序广播
* 粘性广播

虽然有序广播和粘性广播比普通广播具有不同的特性，但是发送接受流程是类似的

从Android 3.1 开始，广播默认是不会发送给已经停止的应用的，因为在broadcastIntentLocked（）中有
`intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);`
* Intent.FLAG_EXCLUDE_STOPPED_PACKAGES  表示不包含已经停止的应用，这个时候广播不会发送给已经停止的应用
* Intent.FLAG_INCLUDE_STOPPED_PACKAGES  包含已经停止的应用

两种状态共存的时候，以FLAG_INCLUDE_STOPPED_PACKAGES 为准
这样做是防止广播无意间火灾不必要的时候调用已经停止运行的应用，如果需要调用未启动的应用，那么**只需要为广播的Intent添加FLAG_INCLUDE_STOPPED_PACKAGES标记即可**

一个应用处于停止状态分为两种情形：
* 应用安装后未运行
* 应用被手动或者其他应用强停了

## ContentProvider

`ContentProvider的onCreate方法要优先于Application的onCreate()方法`

一般来说，ContentProvider都是单例的，android:multiprocess属性决定是不是真的单例，
* 为false的时候，为单例，这也是默认值，
* 为true的时候，为多例，每个调用者进程都存在一个ContentProvider对象

**Context.getContentResolver()得到的实际上是ContextImpl.ApplicationContentResolver**
```java
private static final class ApplicationContentResolver extends ContentResolver {
```
ApplicationContentResolver 继承ContentResolver并且实现了ContentResolver的抽方法，当ContentProvider所在的进程未启动的时候，第一次访问会触发ContentProvider的创建，也就伴随这所在进程的启动，通过ContentProvider的四个方法（CRUD）都可以触发启动过程

通过AMS拿到的ContentProvider并不是原始的ContentProvider，而是ContentProvider的Binder类型的对象IContentProvider，IContentProvider的具体实现是ContentProviderNative,
1. 其他应用通过AMS得到ContentProvider的Binder对象IContentProvider
2. IContentProvider的实际者是ContentProvider.Transport，因此其他应用IContentProvider的CRUD方法最终会以进程间通信的方式调用ContentProvider.Transport的CURD方法，ContentProvider.Transport的CRUD最终又调用了ContentProvider的CRUD方法，最后的结果再通过Binder返回给调用者

---
搬运地址：   
