---
layout: post
title: 扫盲系列 - Android 类加载器 ClassLoader
category: 扫盲系列
tags:  Android ClassLoader
---

* content
{:toc}


## java 中的classloader
![Alt text](../../../../images/java_classloader.png)

类加载流程
![Alt text](../../../../images/class_Loader_loding.png)

## Android 中的classloader

主要包括
* BootClassLoader
* URLClassLoader
* BaseDexClassLoader
* PathClassLoader
* DexClassLoader
* InMemoryDexClassLoader
* DelegateLastClassLoader


这些最终都继承自 java.lang.ClasssLoader

关系图如下：
![Android 平台的ClassLoader](../../../../images/android_classloader.png)

app 中至少需要 BootClassLoader 和 PathClassLoader 才能运行


###  BootCLassLoader  
* 和 JVM 不同的是， BootClassLoader 是 ClassLoader 的内部类，由 Java 代码实现而不是C++
* 是 Android 平台所有 ClassLoader 的最终 Parent 。
* 这个类是包内可见，所以我们无法使用。
* 加载 Android FW 层的字节码文件

### URLClassLoader   
只能用于 JAR 文件，但是由于 Dalvik 不能识别 JAR ，所以在 Android 平台上无法使用这个加载器

###  BaseDexClassLoader  
DexClassLoader 和 PathClassLoader 都继承 BaseDexClassLoader ，其中主要逻辑都是在 BaseDexClassLoader 完成的。是一个基础的加载器，

先看 BaseDexClassLoader 的构造函数
```java
/**
 * @param dexPath         目标类所在的 APK 或 JAR 文件路径，
 * @param optimizedDirectory   优化目录
 * @param libraryPath       目标所使用的c/c++库存放的路径
 * @param parent           该加载器的父加载器，
 */
public BaseDexClassLoader(String dexPath, File optimizedDirectory
              , String libraryPath , ClassLoadeparent) {
    super(parent);
    this.pathList = new DexPathList(this, dexPath , libraryPath , optimizedDirectory);
}
```
四个参数的解释：
* dexPath    类加载器将从该路径中寻找指定的目标类，
  * 该类必须是 APK 或者 JAR 的全路径如果要包括多个路径，
  * 路径之间必须使用指定的分隔符，可以使用System.getProperty("path.separtor"),
  * DexClassLoader 中所谓的支持加载 APK ， JAR ， dex ,也可以从 SD 卡中加载，指的就是这个路径，
  * 最终是将 dexPath 路径上的文件 ODEX 优化到 optimizedDirectoyr ,然后进行加载
* optimizedDirectory   
  * 由于 dex 文件被包含在 APK 或者 JAR 中，因此在装载目标类之前需要从 APK 或者 JAR 文件中解压 dex 文件，该参数就是定制解压出来的 dex 文件存放的路径的，这也是对 apk 的 dex 根据平台进行 ODE 优化的过程，
  * 其实 APK 是一个压缩包，里面包括 dex 文件， ODEX 文件的优化就是把包里面的执行程序提取出来，就变成 ODEX 文件，因为你提取出来了，系统第一次启动的时候就不用去解压程序压缩包里面的程序，少了一个解压过程，这样系统启动就加快，
  * 为什么说是第一次呢？因为 DEX 版本只会有在第一次会解压执行程序到 /data/dalvik-cache(针对PathClassLoder)或者 optimizedDirectory (针对DexClassLoader)目录，之后也就直接读取 dex 文件，所以第二次启动就和正常启动差不多了，当然这只是简单理解，实际生成 ODEX 还有一定的优化作用，
  * ClassLoader只能加载内部存储路径中的 dex 文件，所以这个路径必须是内部路径
* libraryPath   指的是目标所使用的c/c++库存放的路径
* parent    一般为当前执行类的加载器，例如 Android 中以context.getClassLoader()作为父加载器

### PathClassLoader
用来加载已经安装到系统中的 APk 中 class 文件

继承 BaseDexClassLoader ,构造函数如下
```java
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null , null , parent);
    }
  public PathClassLoader(String dexPath, String libraryPath , ClassLoader parent) {
        super(dexPath, null , libraryPath , parent);
    }
}
```
1. PathClassLoader把 optimizedDirectory 设置为 null ，也就是没有设置优化后存放的路径，<span style="border-bottom:1px solid red;">其实 optimizedDirectory 为 null 的默认路径就是/data/dalivk-cache目录</span>
2. 在 Dalivk 虚拟机上只能加载已安装的 APK 的 dex 。在 ART 虚拟机上，可以加载未安装的 APK 的dex
3. 主要用于系统和 app 的类加载器
4. 在应用启动的时候创建，从data/app/安装目录下加载 apk 文件
5. 在 Android 中， app 安装到手机中， apk 的 dex 文件中的 class 文件都是通过 PathClassLoader 来加载的

###  DexClassLoader
继承 BaseDexClassLoader ,构造函数如下
```java
public class DexClassLoader extends BaseDexClassLoader {
  public DexClassLoader(String dexPath, String optimizedDirectory ,
 String libraryPath , ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath , parent);
  }
}
```
1. 支持加载 APK ， JAR ，和 DEX ，也可以从 SD 卡上进行加载
2. 虽然说 dalivk 不能直接识别 JAR ，但却说 DexClassLoader 能加载 JAR 文件，其实这个不矛盾，因为在 BaseDexClassLoader 里面对“.jar”,".apk",".zip",".dex"后缀的文件最后都会生成对应的 dex 文件，所以最终处理的还是 dex 文件，而 URLClassLoader 却没有做这样的处理，所以我们<font color="#ff000" > 一般都是用 DexClassLoader 来作为动态加载的加载器</font>
3. 可以从包含classes.dex的 JAR 或者 APK 中加载类的加载器，用于执行动态加载，但必须是 app 私有可写目录来缓存 ODEX 文件，能够加载系统没按照的 apk 或者 JAR 文件，因此很多插件化方案都是用DexClassLoader

### InMemoryDexClassLoader
在API 26 (Android 8.0 )时新增的一个类加载器

```java
public final class InMemoryDexClassLoader extends BaseDexClassLoader {
    public InMemoryDexClassLoader(ByteBuffer[] dexBuffers, ClassLoader parent) {
        super(dexBuffers, parent);
    }
    public InMemoryDexClassLoader(ByteBuffer dexBuffer, ClassLoader parent) {
        this(new ByteBuffer[] { dexBuffer }, parent);
    }
}
```
由代码可知：
1. 继承 BaseDexClassLoader 。
2. 构造函数调用父类的构造函数， ByteBuffer 数组构造一个 DesPathList ,可用于内存的 DEX 文件

### DelegateLastClassLoader
在API 27 (Android 8.1)新增的类加载器,继承 PathClassLoder.

```java
public final class DelegateLastClassLoader extends PathClassLoader {
    public DelegateLastClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, parent);
    }
    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
      Class<?> cl = findLoadedClass(name);
      if (cl != null) {
          return cl;
      }
      try {
          return Object.class.getClassLoader().loadClass(name);
      } catch (ClassNotFoundException ignored) {
      }
        ClassNotFoundException fromSuper = null;
        try {
            return findClass(name);
        } catch (ClassNotFoundException ex) {
            fromSuper = ex;
        }
        try {
            return getParent().loadClass(name);
        } catch (ClassNotFoundException cnfe) {
            throw fromSuper;
        }
    }
}
```
实行的是最后查找策略。使用 DelegateLastClassLoader 来加载每个类或者资源。使用一下查找顺序
1. 首先判断是否已经加载过该类
2. 然后搜索此类的类加载器是否加载过这个类
3. 最后，搜索与此类加载器的 dexPath 关联的 dex 文件列表，委托给指定的父对象加载


## 双亲委托模型
我们知道，每一个加载器都有一个父加载器，这个双亲委托模型就和这个父加载器有关：如果一个加载器收到了加载类的请求，他首先不会自己尝试去加载这个类，而是把这个请求交给他的父类加载器完成，父类加载器也有父类加载器的，每个加载器都是如此，只有当父加载器在自己搜索范围内找不到指定的类时( ClassNotFoundException),子加载器才会尝试自己去加载。
具体加载过程如下：
1. 源 ClassLoader 判断是否加载过该 Class ，如果加载过，则直接返回该 Class ，如果没有则委托父类加载。
2. 父类加载器判断是否加载过该 Class ，如果加载过，则直接返回该 Class ，如果没有则委托父类的父类，也就是爷爷类加载器
3. 以此类推，直到始祖类加载器（引用类加载器）
4. 始祖类加载器判断是否加载过该 Class ，如果加载过，则直接返回该 Class ，如果没有，则尝试从其对应的类的路径下寻找 class 字节码并载入，如果载入成功，则直接返回 Class ，如果载入失败，则委托给始祖类的子类加下载器
5. 始祖类的子类加载器尝试从其对应的类的路径下寻找 class 字节码并载入，如果载入成功，则直接返回该 Class ，如果载入失败，则委托给始祖类子类的子类，即孙子类加载器
6. 以此类推，直到源ClassLoader
7. 源 ClassLoader 尝试从其对应的类的路径下寻找 class 字节码并载入，如果载入成功，则直接返回该 class ，如果载入失败，源 ClassLoader 不会在委托其他子类加载器，而是抛出异常。

### 委托机制的意义：
`防止内存中出现多份同样的字节码`

例如如果 A 类和 B 类都需要加载一个 System ，如果不用委托机制，而是自己加载自己，那么 A 类会加载一份 System 字节码.B 类又会加载一份 System 字节码,这样内存中就会出现两份 System 字节码。

试想一个场景：黑客自定义一个java.lang.String类，该 String 类具有系统的 String 类一样的功能，只是在某个函数稍作修改。比如 equals 函数，这个函数经常使用，如果在这这个函数中，黑客加入一些“病毒代码”。并且通过自定义类加载器加入到 JVM 中。此时，如果没有双亲委派模型，那么 JVM 就可能误以为黑客自定义的java.lang.String类是系统的 String 类，导致“病毒代码”被执行。

而有了双亲委派模型，黑客自定义的java.lang.String类永远都不会被加载进内存。因为首先是最顶端的类加载器加载系统的java.lang.String类，最终自定义的类加载器无法加载java.lang.String类。

## 双亲委托模型在热修复领域的应用
在用 BaseDexClassLoader 或者 DexClassLoader 去加载某个 dex 或者某个 apk 中的 class 时候，无法调用 findClass() 方法，因为该方法是包访问权限的，需要调用 loadClass() ,该方法其实是 BaseDexClassLoader 的父类 ClassLoader 实现的， findclass() 方法就是双亲委托模型的实际体现。源码如下
```java
public Class<?> loadClass(String className) throws ClassNotFoundException {
    return loadClass(className, false);
}
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
    Class<?> clazz = findLoadedClass(className);//查找是否加载过该 Class 文件

    if (clazz == null) {//没有加载过，
        try {
            //调用父类的 loadClass() 方法去加载该文件
            clazz = parent.loadClass(className, false);
        } catch (ClassNotFoundException e) {
            // Don't want to see this.
        }

        if (clazz == null) {//父类没有加载过或者加载失败，
            clazz = findClass(className);//自己去加载
        }
    }
    //说明加载过，直接返回
    return clazz;
}
```
流程图如下：

![Alt text](../../../../images/loadclass.png)

如图，当前 classloader 是否加载过当前类，如果加载过，就直接返回，如果没有加载过，则看父类有没有加载过，如果加载过，则直接返回，如果都没加载过，则子 classloader 调用 findClass() 真正加载类

findClass() 在 ClassLoader 中是一个空实现，所以需要到具体的子类中去处理，我们知道， PathClassLoader 和 DexClassLoader 都是继承 BaseDexClassloader ,所以可以先在 BaseDexClassloader 中去查找
```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
     List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
     Class c = pathList.findClass(name, suppressedExceptions);
     if (c == null) {
         ClassNotFoundException cnfe = new ClassNotFoundException(
                 "Didn't find class \"" + name + "\" on path: " + pathList);
         for (Throwable t : suppressedExceptions) {
             cnfe.addSuppressed(t);
         }
         throw cnfe;
     }
     return c;
  }
```

这里面关键的就是执行 pathList.findClass() ，所以自定义加载器，主要就是实现 findClass()

## ClassLoader 加载 DEX 的逻辑

DEX 就是能够被 DVM 识别，并且加载执行的文件格式   

一个 ClassLoader 可以有多个 DEX 文件，每个 Dex 文件是一个 Element ,多个 Dex 文件排成一个有序的 dexElements 。   
当加载类的时候，会按照顺序遍历 Dex 文件，然后从当前的 Dex 文件中寻找该类。   
由于双亲委托模型机制，只要找到就会停止并且返回，如果找不到就从下一个 Dex 文件中继续寻找。   
只要先加载修复好的 DEX 文件，那么就不会加载有 bug 的 Dex 文件了。 这就是热修复的核心   

另外，假设 APP 引用的类 A ，再其内部引用的类 B ，如果类 A 和类 B 在同一个 DEX 文件中，那么类 A 上就会被打上 CLASS_ISPREVERIFIED 标记，被标记的这个类不能引用其他 dex 文件中的类，否则会报错。 A 类如果还引用了一个 C 类，而 C 类在其他 dex 中，那么 A 类并不会被打上标记。换句话说，只要在 static 方法，构造方法， private 方法， override 方法中直接引用了其他 dex 中的类，那么这个类就不会被打上 CLASS_ISPREVERIFIED 标记。

解决方案就是 让所有类都引用其他 dex 中的某个类就可以了。


- - -
搬运地址：    

[Android动态加载之 ClassLoader 详解](https://www.jianshu.com/p/a620e368389a)   

[Android类加载器ClassLoader](http://gityuan.com/2017/03/19/android-classloader/)    

[热修复入门：Android 中的 ClassLoader](https://jaeger.itscoder.com/android/2016/08/27/android-classloader.html)  

[Android热补丁动态修复技术（二）：实战！ CLASS_ISPREVERIFIED 问题！](https://blog.csdn.net/u010386612/article/details/51077291)
