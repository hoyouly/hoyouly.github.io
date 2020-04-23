---
layout: post
title: 从一个小例子理解 Binder 整个流程
category: 读书笔记
tags:  Binder
---

<!-- * content -->
<!-- {:toc} -->

之前已经写了一篇关于 [Android Binder 总结](../../../../2018/03/17/Android-Binder/)的文章，但是总感觉还是不太明白，于是就又想了一个感觉还不错的例子再来理解 Binder 流程。

我们来看个例子。

小明同学要打电话的向公安局揭发有人聚众赌博这件事。

然后结合 [Android AIDL 总结](../../../../2019/07/17/Android-AIDL/) 中的例子，一起说。

## 前提条件：
一. 公安局 能处理聚众赌博这件事。                    
  * IBookManager 中 有 getBookList()

二. 公安局中有具体人负责聚众赌博这事，比如王队长，专门处理聚众赌博。   
  * 创建一个 Stub ,实现 IBookManager 接口，

三. 公安局的电话是可以在电话本里面查找到。             
  * Service在AndroidMinfest.xml文件注册  

```Java
<service android:name="com.hoyouly.android_art.BookManagerService"
         android:enabled="true"
         android:exported="true"
         android:process=":remote">
    <intent-filter>
        <action android:name="com.hoyouly.android_art.service" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</service>
```

四. 王队长处理聚众赌博这事已经上报给了报警中心。
  * onBind()中返回该 Stub 对象。

五. 小明知道 公安局能处理聚众赌博这事        
  * 客户端 知道能通过 IBookManager 的 getBookList() 得到书本列表

## 整个流程
一. 小明同学通过电话本找到公安局的电话。例如 110 ，
  * 其实很多时候，服务端会告诉你怎么连接服务，比如通过某个 Action 即可连接上去。
```Java
Intent commonIntent = new Intent();
commonIntent.setAction("com.hoyouly.android_art.service");
commonIntent.setPackage(getContext().getPackageName());
```

二. 拿起电话，开始拨号 110     
  * 知道 Action 后，开始连接服务器。
```java
getContext().bindService(commonIntent, mConnection , Context.BIND_AUTO_CREATE);
```

三. 这时会有两种可能，电话接通，或者电话未接通。

1. 电话接通 对方会说，你好，这里是***公安局，有什么可以帮到你的吗？   
  * 执行到 onServiceConnected() ,这里面会有一个 iBinder 对象    
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

四. 小明说我要找能处理聚众赌博这件事的人。
  * 得到 mService 对象。
```java
mService = IBookManager.Stub.asInterface(iBinder);
```

五. 转接成功。小明同学以为是接的电话的人就能处理聚众赌博这事。于是开始说聚众赌博这件事（某某地方有一群人。天天聚众赌博....）

可是实际上接电话的只是一个客服。  
  *  执行 mService.getBookList()，其实 mService 只是一个代理对象。

六. 客服自己知道没有处理聚众赌博这件事的能力，可是他会把这件事记录下来，发送给报警中心。但是会说，请稍等。正在处理。
  * mService 把数据封装起来，然后执行 transact() 通过 Binder 驱动发送给服务端，报警中心就相当于 Binder 驱动
```java
public java.util.List<com.hoyouly.android_art.Book> getBookList() throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    java.util.List<com.hoyouly.android_art.Book> _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        mRemote.transact(Stub.TRANSACTION_getBookList, _data , _reply , 0);
        _reply.readException();
        _result = _reply.createTypedArrayList(com.hoyouly.android_art.Book.CREATOR);
    } finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```

七. 报警中心知道这需要王队长处理，于是告诉让王队长处理聚众赌博这件事。王队长亲自带队出警，抓聚众赌博。于是把结果告诉客服。这期间，小明和客服都处于等待中。   
  * 服务端开始处理，然后通过 onTransact() 把结果给代理对象。
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
    return super.onTransact(code, data , reply , flags);
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

八. 客服得到结果，然后把结果告诉给小明同学。已出警。
  * 客户端得到数据，进行相应的操作。例如遍历mBookList

九. 小明愉快的挂断了电话，
* 断开连接
```java  
getContext().unbindService(mConnection);
```

这里面
* 小明 就是客户端
* 公安局 就是服务端，
* 电话本 就是 ServiceManager ,
* 报警中心 就是 Binder 驱动。
* 王队长  就是是 Stub 类，
* 客服 就是Proxy.  
* 聚众赌博 就是 getBookList() 方法

这样理解是不是更清楚一些呢！！！


---
搬运地址：    

Android 开发艺术探索

[Android Binder之应用层总结与分析](http://blog.csdn.net/qian520ao/article/details/78089877)

[Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)

[Android面试一天一题（Day 35：神秘的 Binder 机制）](https://www.jianshu.com/p/c7bcb4c96b38)
