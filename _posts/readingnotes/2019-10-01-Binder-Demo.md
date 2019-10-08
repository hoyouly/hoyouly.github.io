---
layout: post
title:  小例子再理解Binder过程
category: 读书笔记
tags:  Binder
---

* content
{:toc}

之前已经写了一篇关于 [Android Binder 总结](http://hoyouly.fun/2018/03/17/Binder/)的文章，但是总感觉还是不太明白，于是就又想了一个感觉还不错的例子再来理解Binder原理。

我们来看个例子。

小明同学要打电话的向教育局反馈老师打人这件事。

然后结合 [Android AIDL 总结](http://hoyouly.fun/2019/07/17/Android-AIDL/) 中的例子，一起说。

## 前提条件：
1. 教育局 能处理老师打人这件事。                    
  * IBookManager 中 有 getBookList()
2. 教育局中有具体人负责处理打人这事，比如是老王同志。   
  * 创建一个Stub,实现 IBookManager接口，
3. 教育局的电话是可以在电话本里面查找到。             
  * 这个没想明白属于哪一步操作
4. 老王处理老师打人这事已经上报给了客户服务中心。
  * onBind()中返回该Stub对象。
5. 小明知道 教育局能处理老师打人这事        
  * 客户端 知道能通过IBookManager的 getBookList()得到书本列表

## 整个流程
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

1. 电话接通 对方会说，你好，这里是***教育局客服服务中心，有什么可以帮到你的吗？   
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

七. 客户服务中心知道这需要老王处理，于是告诉让老王处理老师打人这件事。老王经过核实，是老师不对，然后把老师停职。于是把结果告诉客服。这期间，小明和客服都处于等待中。   
  * 服务端开始处理，然后通过onTransact()把结果给代理对象。
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

八. 客服得到结果，然后把结果告诉给小明同学。老师被停职了。
  * 客户端得到数据，进行相应的操作。例如遍历mBookList
  
九. 小明愉快的挂断了电话，
* 断开连接
```java  
getContext().unbindService(mConnection);
```

这里面小明就是客户端，教育局就是服务端，电话本就是ServiceManager,客户服务中心就是Binder驱动。
而老王呢，则是Stub类，客服就是Proxy.

这样理解是不是更清楚一些呢！！！


---
搬运地址：
Android 开发艺术探索

[Android Binder之应用层总结与分析](http://blog.csdn.net/qian520ao/article/details/78089877)

[Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)

[Android面试一天一题（Day 35：神秘的Binder机制）](https://www.jianshu.com/p/c7bcb4c96b38)
