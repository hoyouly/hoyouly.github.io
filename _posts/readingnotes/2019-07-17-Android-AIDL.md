---
layout: post
title: Android AIDL 总结
category: 读书笔记
tags: Android开发艺术探索 AIDL
description: Android AIDL
---

* content
{:toc}


## 名词解释
### IInterface
自定义AIDL接口需要继承了 IInterface 接口，只有一个asBinder(),用来返回AIDL中Stub类的对象。如果是同进程的本地接口，则返回this，否则，返回BinderProxy代理对象。

### IBinder
* IBinder是一个接口，它代表了一种跨进程传输的能力；只要实现了这个接口，就能将这个对象进行跨进程传递；这是驱动底层支持的；在跨进程数据流经驱动的时候，驱动会识别IBinder类型的数据，从而自动完成不同进程Binder本地对象以及Binder代理对象的转换。
* IBinder负责数据传输，那么client与server端的调用契约（这里不用接口避免混淆）呢？这里的IInterface代表的就是远程server对象具有什么能力。具体来说，就是aidl里面的接口。

### Java层的Binder类
Java层的Binder类，代表的其实就是Binder本地对象。BinderProxy类是Binder类的一个内部类，它代表远程进程的Binder对象的本地代理；这两个类都继承自IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder驱动会自动完成这两个对象的转换。

在使用AIDL的时候，编译工具会给我们生成一个Stub的静态内部类；这个类继承了Binder, 说明它是一个Binder本地对象。它实现了IInterface接口，表明它具有远程Server承诺给Client的能力；Stub是一个抽象类，具体的IInterface的相关实现需要我们手动完成，这里使用了策略模式。

## AIDL
1. Android 接口定义语言。Android Interface Definition Language
2. 方便系统为我们生成代码从而实现跨进程通信。说白了就是一个快速跨进程的工具。

### 支持的数据类型
* 基本数量类型（int long，char，boolean，double等）
* String和CharSeuence
* List，只支持Arraylist，里面每个元素都必须能够被AIDL支持，这个List是抽象的List,而List只是一个接口，虽然可以在服务端返回，CopyOnWriteArrayList，但是在Binder中会按照List的规范去访问数据并最终形成一个新的ArrayList传递给客户端。
* Map，只支持HashMap,里面的每个元素都必须被AIDL支持，包括key和value
* Parcelable，所有实现Parcelable 接口的对象，
* AIDL，所有的AIDL接口本身也可以在AIDL文件使用

### 注意事项
* 自定义的Parcelable对象和AIDL对象必须要显示的import进来，不管是否和当前的AIDL文件位于同一个包内，
* 如果用到Parcelable对象，那么必须新建一个和它同名的AIDL文件，并且在其中生命为Parcelable类型，
* 除了基本数据类型，其他的类型参数必须标上方向：in,out或者inout
	* in  输入型参数，客户端数据流入服务端，并且服务端对该数据的修改不会影响客户端
	* out 输出型参数，数据对象有服务端流向客户端，（客户端传递的数据对象时服务端收到的对象内容为空，服务端可以对该数据对象修改，并传给客户端）
	* inout 输入输出型参数，即两种数据流向的结合体。但是不建议使用，因为会增加开销
* AIDL接口中只支持方法，不支持声明静态常量。
* 建议把所有和AIDL相关的类和文件全部都放入同一个包中，好处就是当客户端是另外一个应用时，可以直接把整个包复制到客户端工程中。
* AIDL文件中不能存在同方法名不同参数的方法
* AIDL实体类中必须要有指定的TAG （in out inout）

### CopyOnWriteArrayList
支持并发读/写。

AIDL方法在服务端的Binder线程池中执行，因此当多个客户端同时连接的时候，会存在多个线程同时访问的情形，所以我们要在AIDL方法中处理线程同步问题，而使用CopyOnWriteArrayList 可以进行自动的线程同步

### 权限验证
1. 在onBind()中进行，验证不通过就直接返回null，这样验证失败的客户端直接无法连接服务，验证方式： 可以使用permission 验证。
2. 在服务端的onTransace() 方法中进行权限验证，如果验证失败就返回false，这样服务端就不会终止执行AIDL的方法从而达到保护服务端的效果，验证方式：可以通过使用Permission 验证，还可以通过Uid和Pid来验证，

### 创建AIDL文件
1. AndroidStudio的aidl文件默认放在src/main/aidl目录下，aidl目录和java目录同级别。
2. 在java目录上右键，创建一个aidl文件，此文件会默认生成到aidl目录下。
3. 同时必须要指明包名，包名必须和java目录下的包名一致。
4. 如果aidl需要使用Model类，Model类必须要实现Parcelable接口！必须要import进来，不然会找不到
5. 然后Make一下，就会自动生成Java文件。

## AIDL 的流程
从客户端发起请求到服务器响应的工作流程。可以看出整体的核心就是Binder
![Alt text](../../../../images/aidl_binder.jpeg)

AIDL文件生成的类。如下

![Alt text](../../../../images/IBookManager_method.png)
由上图结构可以看出，
1. IBookManager中有两个方法，也就是我们在AIDL中定义的那两个方法，addBook(),getBookList()
2. 有一个内部类Stub，extends Binder , 这个就是Binder类，同时实现了IBookManager接口
3. 内部类Stub还有一个内部类Proxy，也实现了IBookManager 接口
```java
public static abstract class Stub extends android.os.Binder
                    implements com.hoyouly.android_art.IBookManager {

      private static class Proxy
                    implements com.hoyouly.android_art.IBookManager{

                    }
}
```

## Stub类
### DESCRIPTOR
Binder的唯一标示，一般用当前Binder的类名标示，比如
```java
private static final java.lang.String DESCRIPTOR = "com.hoyouly.android_art.IBookManager";
```

### asInterface()
用于将服务端的Binder对象转换成客户端所需要的AIDL接口类型的对象，这种转换就是区分进程的。   
如果客户端和服务端唯一同一个进程，那么此方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.Proxy对象
```java
public static com.hoyouly.android_art.IBookManager asInterface(android.os.IBinder obj) {
  //这个obj对象，就是onServiceConnected()中的IBinder对象iBinder，然后在Proxy的构造函数中保存到mRemote中
    if ((obj == null)) {
        return null;
    }
    //通过DESCRIPTOR标识判断
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof com.hoyouly.android_art.IBookManager))) {
      //相同进程
      return ((com.hoyouly.android_art.IBookManager) iin);
    }
    //不同进程，返回Stub的代理内部类Proxy。
    return new com.hoyouly.android_art.IBookManager.Stub.Proxy(obj);
}
```

也就是asInterface()方法返回的是一个远程接口具备的能力（有什么方法可以调用）

而我们知道，在客户端绑定服务bindService()的时候，onServiceConnected()中会调用该方法。点击 [Android 四大组件之 Service](../../../../2019/03/25/Android-Service-Core/) 了解更多详情。

```java
private ServiceConnection mConnection = new ServiceConnection() {
		@Override
		public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
      //iBinder 是在ActivityThread中创建的.  因为Binder实现了IBinder，所以这个iBinder就是实际上的Binder对象
			mService = IBookManager.Stub.asInterface(iBinder);
		}

		@Override
		public void onServiceDisconnected(ComponentName componentName) {
			mService = null;
		}
	};

getContext().bindService(commonIntent, mConnection, Context.BIND_AUTO_CREATE);
```

### asBinder()
是 IInterface 定义的方法，返回当前Binder对象
```Java
//Stub.java
@Override
public android.os.IBinder asBinder() {
    return this;
}

//Stub.Proxy.java
@Override
public android.os.IBinder asBinder() {
    //就是onServiceConnected()中的IBinder对象iBinder，
    return mRemote;
}
```
这个Binder对象具有跨进程能力，在Stub类里面（也就是本进程）直接就是Binder本地对象，在Proxy类里面返回的是远程代理对象(Binder代理对象)。
![Alt text](../../../../images/asbinder.png)

### onTransact()
在`服务端的Binder线程池`中，当客户端发起跨进程请求的时，远程请求会通过系统底层封装后交由此方法来处理，如果此方法返回false,那么客户端的请求就会失败。我们可以利用这个特性来做权限验证，

当客户端和服务端都位于同一个进程的时候，该方法调用不会走跨进程的transact()方法，而当两者位于不同的进程时，该方法调用执行transact()方法，而这个逻辑由Stub的内部代理类Proxy来完成，所以，**这个接口的核心实现就是内部类Stub和Stub的内部代理类Proxy**：
```java
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
```
这几个参数的意义
* code : 确定客户端请求的目标方法是什么。（IBookManager 里面的方法）
* data : 如果目标方法有参数的话，就从data取出目标方法所需的参数。
* reply : 当目标方法执行完毕后，如果目标方法有返回值，就向reply中写入返回值。
* flag : Additional operation flags. Either 0 for a normal RPC, or FLAG_ONEWAY for a one-way RPC.（暂时还没有发现用处，先标记上英文注释）   

总结就是：<span style="border-bottom:1px solid red;">服务端通过code可以确定客户端所请求的目标方法，接着从data中取出目标方法所需要的参数（如果有的话），然后执行目标方法，当执行完毕，就向reply中写入返回值（如果有返回值）。</span>

```java
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply
									, int flags) throws android.os.RemoteException {
   switch (code) {
		 ...
		 case TRANSACTION_getBookList: {
			 	data.enforceInterface(DESCRIPTOR);
			 	//执行目标方法 getBookList(),得到返回结果
			 	java.util.List<com.hoyouly.android_art.Book> _result = this.getBookList();
			 	reply.writeNoException();
			 	reply.writeTypedList(_result);//把结果写入reply中。
			 	return true;
	 	}
	 	case TRANSACTION_addBook: {
		    	data.enforceInterface(DESCRIPTOR);
		    	com.hoyouly.android_art.Book _arg0;
		    	if ((0 != data.readInt())) {//读取所需要的参数
		        	_arg0 = com.hoyouly.android_art.Book.CREATOR.createFromParcel(data);
		    	} else {
		        	_arg0 = null;
		    	}
		    	this.addBook(_arg0);
		    	reply.writeNoException();
		    	return true;
				}
				...			 
  	}
  return super.onTransact(code, data, reply, flags);
}
```

## Stub.Proxy类
主要是用做客户端跨进去调用的。
有一个mRemote对象，这个就是在onServiceConnected()中的IBinder对象iBinder，传递给了Stub的asInterface()，然后在asInterface()中创建Proxy对象，又赋值给了mRemote
```java
private android.os.IBinder mRemote;

Proxy(android.os.IBinder remote) {
    mRemote = remote;
}
```

### Proxy # getBookList()
在客户端运行，,源码如下
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
当客户端远程调用此方法是，它的内部实现：
1. 首先创建该方法所需要的输入型Parcel对象_data,输出型Parcel对象_reply,和返回值对象list，**在跨进程通讯中Parcel是通讯的基本单元，传递载体。**
2. 把该方法的参数信息写入_data，
3. 调用transact()方法来发起RPC（远程过程调用）请求，同时当前线程挂起，然后服务器的onTransact()方法会被调用，直到RPC过程返回，当前线程继续执行，transact()是一个本地方法。在native层实现的。
4. 从_reply中读取RPC过程的返回结果
5. 返回_reply中的数据

## Stub和Proxy 比较
相同点： 他们均实现了所有的IInterface函数。

不同点:
1. 在创建的时候，虽然都接收一个IBinder对象，但是Stub采用的是继承（is 关系），Proxy采用的是组合（has 关系）
2. Stub 使用策略模式调用的是虚函数（待子类实现），而Proxy则使用组合模式，
    * Stub本身是一个IBinder（Binder），也就是一个能跨越进程边界传输的对象，所以它得继承IBinder实现transact()这个函数从而得到跨越进程的能力（这个能力由驱动赋予）
    * Proxy类使用组合，是因为他不关心自己是什么，它也不需要跨越进程传输，它只需要拥有这个能力即可，要拥有这个能力，只需要保留一个对IBinder的引用
3. Proxy中 接口的方法只是把参数包装然后交给驱动

就算是到这里，还是没解释清楚，本地服务怎么和Stub关联起来啊。

## 本地服务和Stub关联起来
### 服务端创建Stub
在我们创建本地服务的时候，需要实现onBind()方法，而这个onBind()返回的是一个Binder对象，而这个Binder对象，我们一般都是创建出来的，通常是
```java
public class RecognitionRemoteService extends Service {
  private Binder mBinder = new IBookManager.Stub(){
    ...
  }

  public IBinder onBind(Intent intent) {
		return mBinder;
	}
}
```

查看Stub类，是一个抽象类，继承Binder，所以我们在服务端中创建Binder对象的时候，可以直接 new IBookManager.Stub()，然后把该对象在onBind()中返回。

又因为实现了它的外部类IBookManager （`这中写法挺怪`）。所以new IBookManager.Stub() 的时候，需要实现里面的抽象方法，即我们定义的那几个方法，所以这个Stub类又具有AIDL接口的能力。
```java
public interface IBookManager extends android.os.IInterface {
public static abstract class Stub extends android.os.Binder implements com.hoyouly.android_art.IBookManager {
		private static final java.lang.String DESCRIPTOR = "com.hoyouly.android_art.IBookManager";
		public Stub() {
			this.attachInterface(this, DESCRIPTOR);
		}
    ...
}
```

#### Binder # attachInterface()

先看看 attachInterface()  Stub 继承Binder，但是并没有重写 attachInterface()方法，所以我们需要去Binder中查找。
```java
//Binder.java
public void attachInterface(IInterface owner, String descriptor) {
  mOwner = owner;
  mDescriptor = descriptor;
}
```
在Binder中，attachInterface() 只是简单的保存传递过来的变量,
* mDescriptor 就是我们传递过来的DESCRIPTOR，即`"com.hoyouly.android_art.IBookManager"`
* mOwner  是 this,即创建的Stub对象。
这样就创建了一个 具有我们定义的接口能力的Binder对象。

### 客户端绑定Service
我们知道，在客户端绑定服务的时候，需要通过IBookManager.Stub.asInterface(iBinder) 得到服务对象

```java
private ServiceConnection mConnection = new ServiceConnection() {
	@Override
	public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
		mService = IBookManager.Stub.asInterface(iBinder);
	}

	@Override
	public void onServiceDisconnected(ComponentName componentName) {
		mService = null;
	}
}
```

#### Stub # asInterface()
继续看asInterface() 方法,接受的参数是onServiceConnected() 传递过来的IBinder对象iBinder，这个iBinder是在ActivityThread中创建的
因为Binder实现了IBinder，所以这个iBinder实际上是Binder对象

```java
IBookManager.Stub
public static com.hoyouly.android_art.IBookManager asInterface(android.os.IBinder obj) {
   //这个obj对象，就是onServiceConnected()中的IBinder对象iBinder，然后在Proxy的构造函数中保存到mRemote中
    if ((obj == null)) {
        return null;
    }
    //通过DESCRIPTOR标识判断
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof com.hoyouly.android_art.IBookManager))) {
      //相同进程
      return ((com.hoyouly.android_art.IBookManager) iin);
    }
    //不同进程，返回Stub的代理内部类Proxy。并保存到Proxy中
    return new com.hoyouly.android_art.IBookManager.Stub.Proxy(obj);
}
```
Stub继承Binder，但是并没有重写 queryLocalInterface()方法，所以我们需要去Binder中查找。
```java
public IInterface queryLocalInterface(String descriptor) {
    if (mDescriptor.equals(descriptor)) {
        return mOwner;
    }
    return null;
}
```
之前在attachInterface() 方法中已经传递过来descriptor了，而这个descriptor是Binder的唯一标识，一般用当前Binder的类名表示。即
```Java
private static final java.lang.String DESCRIPTOR = "com.hoyouly.android_art.IBookManager";
```
queryLocalInterface() 得到的就是我们本地服务中创建的Binder对象，这样就可以直接调用本地服务中实现Binder对象的方法


## 总结
AIDL 通过Stub 来接受并处理数据，Proxy代理类用来发送数据。这两个类只是对Binder的处理和调用而已。
![Alt text](../../../../images/stub_proxy_binder.jpeg)
并且客户端和服务器是相对而言的，服务端不仅可以接收和处理消息，而且可以定时往客户端发送数据，与此同时服务端使用Proxy类跨进程调用，相当于充当了"Client"。

---
搬运地址：   
Android 开发艺术探索

[Android 深入浅出AIDL（一）](https://blog.csdn.net/qian520ao/article/details/78072250)  

[Android 深入浅出AIDL（二）](https://blog.csdn.net/qian520ao/article/details/78074983)  
