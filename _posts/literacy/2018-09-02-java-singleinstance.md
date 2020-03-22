---
layout: post
title: 扫盲系列 - Java 单例模式
category: 扫盲系列
tags:  Java
---

* content
{:toc}

##  恶汉式
```java
public class Singleton {
  private  static Singleton singleton=new Singleton();
  private Singleton(){}
  public static Singleton newInstance(){
    return singleton;
  }
}
```
## 双重校验锁 DCL
```java
public class Singleton {
  private  static volatile Singleton singleton;
  private Singleton(){}
  public static Singleton newInstance(){
    if(singleton==null){
      synchronized (Singleton.class){
        if(singleton==null){
          singleton=new Singleton();
        }
      }
    }
    return singleton;
  }
}
```
volatile的两层含义:
*  可见性。可见性指的是在一个线程中对该变量的修改会马上由工作内存（Work Memory）写回主内存（Main Memory），所以会马上反应在其它线程的读取操作中
*  禁止指令重排优化，但是这个语义直到jdk1.5以后才能正确工作。此前的JDK中即使将变量声明为volatile也无法完全避免重排序所导致的问题。所以，在jdk1.5版本前，双重检查锁形式的单例模式是无法保证线程安全的。不过好在现在使用JDK1.5 之前版本的很少很少了，

##  静态内部类
```java
public class Singleton {
  private static class SingletonHolder {
    private static final Singleton INSTANCE = new Singleton();
  }
  private Singleton() {
  }
  public static final Singleton getInstance() {
    return SingletonHolder.INSTANCE;
  }
}
```
以上所有实现方式都有两个共同缺点：
* 都需要额外的工作(Serializable、transient、readResolve())来实现序列化，否则每次反序列化一个序列化的对象实例时都会创建一个新的实例。
* 可能会有人使用反射强行调用我们的私有构造器（如果要避免这种情况，可以修改构造器，让它在创建第二个实例的时候抛异常）。

## 枚举写法
```java
public enum Singleton {
    INSTANCE;
}
```
* 使用枚举除了线程安全和防止反射强行调用构造器之外，
* 提供了自动序列化机制，防止反序列化的时候创建新的对象。

<span style="border-bottom:1px solid red;">因此，Effective Java推荐尽可能地使用枚举来实现单例。   
尽管枚举写法有点这么多，但是在Android平台上却不被推荐使用。</span>

原因 [Android 官方API指南](https://developer.android.com/topic/performance/memory.html) 明确指出：`Enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android. 翻译过来就是枚举通常需要比静态常量多两倍的内存。 你应该严格避免在Android上使用枚举。`简单来说就是枚举耗内存，而Android最缺的就是内存。


建议
* 如果是在Android应用上，建议使用双重校验锁 DCL  或者静态内部类，这两个效果是一样的。
* 如果是java程序，建议使用 枚举类型写法


下面是我在Android源码中看到的，看着像是懒汉式的二次封装，可以参考一下
##  Android源码中的单例模式
```java
//android.util.Singleton.java
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

//使用方法如下
public abstract class ActivityManagerNative extends Binder implements IActivityManager {
  private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
      //通过应用程序中的0号引用，可以向SMgr获取服务端（Server）的Binder引用。
      IBinder b = ServiceManager.getService("activity");
      IActivityManager am = asInterface(b);
      return am;
    }
  };

static public IActivityManager getDefault() {
    return gDefault.get();
  }
}

ActivityManagerNative.getDefault().finishActivityAffinity(mToken)
```

- - - -
搬运地址：    


 [你真的会写单例模式吗——Java实现](http://www.importnew.com/18872.html)
