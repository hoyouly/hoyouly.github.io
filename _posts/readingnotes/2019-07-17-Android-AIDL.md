---
layout: post
title: 深入了解 Android AIDL
category: 读书笔记
tags: Android开发艺术探索 AIDL
description: Android AIDL
---

* content
{:toc}

### AIDL
可以进行跨进程通信，
#### AIDL支持的数据类型
* 基本数量类型（int long，char，boolean，double等）
* String和CharSeuence
* List，只支持Arraylist，里面每个元素都必须能够被AIDL支持，这个List是抽象的List,而List只是一个接口，虽然可以在服务端返回，CopyOnWriteArrayList，但是在Binder中会按照List的规范去访问数据并最终形成一个新的ArrayList传递给客户端。
* Map，只支持HashMap,里面的每个元素都必须被AIDL支持，包括key和value
* Parcelable，所有实现Parcelable 接口的对象，
* AIDL，所有的AIDL接口本身也可以在AIDL文件使用

注意一下几点：
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

CopyOnWriteArrayList 支持并发读/写,AIDL方法砸服务端的Binder线程池中执行，因此当多个客户端同时连接的时候，会存在多个线程同时访问的情形，所以我们要在AIDL方法中处理线程同步问题，而使用CopyOnWriteArrayList 可以进行自动的线程同步

#### ADIL权限验证
1. 在onBind中进行，验证不通过就直接返回null，这样验证失败的客户端直接无法连接服务，验证方式： 可以使用permission 验证。
2. 在服务端的onTransace 方法中进行权限验证，如果验证失败就返回false，这样服务端就不会终止执行AIDL的方法从而达到保护服务端的效果，验证方式：可以通过使用Permission 验证，还可以通过Uid和Pid来验证，

#### AS中创建AIDL文件
1. AndroidStudio的aidl文件默认放在src/main/aidl目录下，aidl目录和java目录同级别。
2. 在java目录上右键，创建一个aidl文件，此文件会默认生成到aidl目录下。
3. 同时必须要指明包名，包名必须和java目录下的包名一致。
4. 如果aidl需要使用Model类，Model类必须要实现Parcelable接口！必须要import进来，不然会找不到
5. 然后Make一下，就会自动生成Java文件。

#### AIDL 的流程
从客户端发起请求到服务器响应的工作流程。可以看出整体的核心就是Binder
![Alt text](https://github.com/hoyouly/BlogResource/raw/master/imges/aidl_binder.png)

AIDL文件生成的类。如下

![Alt text](https://github.com/hoyouly/BlogResource/raw/master/imges/IBookManager_method.png)
由上图结构可以看出，IBookManager中有两个方法，也就是我们在AIDL中定义的那两个方法，addBook(),getBookList(),还有一个内部类Stub，extends Binder ,这个就是Binder类，
还有一个Stub还有一个内部类Proxy实现我们定义的AIDL类
```java
public static abstract class Stub extends android.os.Binder implements com.hoyouly.android_art.IBookManager {
      private static class Proxy implements com.hoyouly.android_art.IBookManager{}
}
```
### Stub类和代理类Stub.Proxy
#### DESCRIPTOR
Binder的唯一标示，一般用当前Binder的类名标示，比如
```java
private static final java.lang.String DESCRIPTOR = "com.hoyouly.android_art.IBookManager";
```
#### asInterface(android.os.IBinder obj)
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
而我们知道，在客户端bindService(绑定服务)的时候，onServiceConnected()中会调用该方法
```java
private ServiceConnection mConnection = new ServiceConnection() {
		@Override
		public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
			mService = IRecognitionInterface.Stub.asInterface(iBinder);
		}

		@Override
		public void onServiceDisconnected(ComponentName componentName) {
			mService = null;
		}
	};

Intent commonIntent = new Intent();
commonIntent.setAction("cn.okay.recognition.service");
commonIntent.setPackage(getContext().getPackageName());
getContext().bindService(commonIntent, mConnection, Context.BIND_AUTO_CREATE);
```

#### asBinder()
是IInterface定义的方法，IInterface 是Binder接口的基类。
返回当前Binder对象
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
![Alt text](https://github.com/hoyouly/BlogResource/raw/master/imges/asbinder.png)

#### onTransact()
在`服务端的Binder线程池`中，当客户端发起跨进程请求的时，远程请求会通过系统底层封装后交由此方法来处理，如果此方法返回false,那么客户端的请求就会失败。我们可以利用这个特性来做权限验证，
当客户端和服务端都位于同一个进程的时候，该方法调用不会走跨进程的transact方法，而当两者位于不同的进程时，该方法调用执行transact方法，而这个逻辑由Stub的内部代理类Proxy来完成，所以，**这个接口的核心实现就是内部类stub和Stub的内部代理类Proxy**：
```java
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
```
这几个参数的意义
* code : 确定客户端请求的目标方法是什么。（IRecognitionInterface 里面的方法）
* data : 如果目标方法有参数的话，就从data取出目标方法所需的参数。
* reply : 当目标方法执行完毕后，如果目标方法有返回值，就向reply中写入返回值。
* flag : Additional operation flags. Either 0 for a normal RPC, or FLAG_ONEWAY for a one-way RPC.（暂时还没有发现用处，先标记上英文注释）
服务端通过code可以确定客户端所请求的目标方法，接着从data中取出目标方法所需要的参数（如果有的话），然后执行目标方法，当执行完毕，就向reply中写入返回值（如果有返回值）。
```java
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
   switch (code) {
     ...
    case TRANSACTION_getCorrectResult: {
        data.enforceInterface(DESCRIPTOR);
        cn.okay.recognition.CoordinateValues _arg0;
        if ((0 != data.readInt())) {//读取所需要的参数
            _arg0 = cn.okay.recognition.CoordinateValues.CREATOR.createFromParcel(data);
        } else {
            _arg0 = null;
        }
        //执行目标方法 getCorrectResult(),得到返回结果
        java.util.List<cn.okay.recognition.CorrectionResult> _result = this.getCorrectResult(_arg0);
        reply.writeNoException();
        reply.writeTypedList(_result);//把结果写入reply中。
        return true;
    }
  }
  return super.onTransact(code, data, reply, flags);
}
```

### Proxy
主要是用做客户端跨进去调用的。

#### Proxy # getCorrectResult()
在客户端运行，,源码如下
```java
@Override
public java.util.List<cn.okay.recognition.CorrectionResult> getCorrectResult(cn.okay.recognition.CoordinateValues coordinateValues) throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    java.util.List<cn.okay.recognition.CorrectionResult> _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        if ((coordinateValues != null)) {
            _data.writeInt(1);
            coordinateValues.writeToParcel(_data, 0);
        } else {
            _data.writeInt(0);
        }
        mRemote.transact(Stub.TRANSACTION_getCorrectResult, _data, _reply, 0);
        _reply.readException();
        _result = _reply.createTypedArrayList(cn.okay.recognition.CorrectionResult.CREATOR);
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

**总结：AIDL文件并不是Binder的必需品，AIDL文件的本质是系统为我们提供了一种快捷实现Binder的工具，仅此而已**
---
搬运地址：
[Android 深入浅出AIDL（一）](https://blog.csdn.net/qian520ao/article/details/78072250)  

[Android 深入浅出AIDL（二）](https://blog.csdn.net/qian520ao/article/details/78074983)  
