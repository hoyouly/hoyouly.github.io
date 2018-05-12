---
layout: post
title: Android IPC 简介
category: 读书笔记
tags: Android开发艺术探索 IPC
description: Android IPC 简介
---

* content
{:toc}

## IPC
IPC ：Inter_Process Communication的缩写，含义进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。
* 线程：CPU调度的最小单位，
* 进程：一般只一个执行单元，在PC和移动设备中指一个程序或者应用，一个进程可以有多个线程，
 Android中主线程也叫UI线程，UI线程中不能执行耗时操作，不然会引起ANR，耗时操作需要使用线程

**Android中最特色的进程间通信方式就是Binder，通过呢Binder可以起送的实现两个进程通信，除了Binder，Android还执行Socket，通过Socket也可以实现任意两个终端之间的通信，**

多进程情况：
1. 一个应用由于某些原因自己需要采用多进程的方式来实现。
2. 当前应用需要向其他应用获取数据
# Android中的多进程模式
**就可以轻易的开启多进程模式**
## 开启多进程模式
**Android中只有一种方式：**通过给四大组件（Activity，Service，Receiver，ContentProvider）在AndroidManifest中指定android:process 属性，

```java
<application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>

                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <activity
            android:name=".chap2.SecondActivity"
            android:configChanges="screenLayout"
            android:label="@string/app_name"
            android:process=":remote"/>
        <activity
            android:name=".chap2.ThridActivity"
            android:configChanges="screenLayout"
            android:label="@string/app_name"
            android:process="com.hoyouly.android_art.remote"/>

    </application>
```
以上例子一共有三个进程
* 当SecondActivity启动的时候，系统创建单独进程，进程名为com.hoyouly.android_art.chap2`:`remote
* 当ThridActivity 启动的时候，系统创建单独进程，进程名为com.hoyouly.android_art`.`remote
* 由于MainActivity没有指定process属性，那么她就运行在默认进程中，`默认进程名就是包名，即com.hoyouly.android_art`
查看进程信息的两种方式
1. 在DDMS试图中查看
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE024e37b589066e35b0a49d9b111a7884/3061)

2. adb 命令：**adb shell ps 或者adb shell ps | grep com.hoyouly**
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE2432a29f3df94024259cfcaf3f9bfd46/3063)

**注意：** 必须是SecondActivity 或者ThridActivity界面启动，才能把进程启动起来，如果只是单纯的在AndroidManifest文件中注册，是开启不了多进程的

**`:`remote  和com.hoyouly.android_art.chap2`.`remote  区别**

|区别      |`:`remote|   com.hoyouly.android_art.chap2`.`remote  |
| :--------: | :--------| :------ |
| 区别一    | 指当前进行名前加上附加上当前包名，即com.hoyouly.android_art.chap2`:`remote，这是一种简单写法 |  完整命名方式，不附件包名信息， |
| 区别二    |   属于当前应用的私有进程，其他应用的组件不可以和他泡在同一个进程中 | 全局进程，其他应用通过ShareUID方式可以和他跑在同一个进程中 |

### UID
Android系统会为每个应用分配一个唯一的UID，具有相同的UID的应用才能共享数据，
两个应用通过ShareUID 泡在同一个进程中是有要求：这两个应用有相同的ShareUID并且签名相同才行。
## 多进程模式的运行机制
Android为每个应用分配了一个独立的虚拟机，或者说每一个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，**一个进程中修改某个值，只会影响当前进程，对其他进程不会造成影响。**

### 多进程造成的问题
1. 静态成员和单利模式失效
2. 线程同步机制完全失效
3. SharedPreferences的可靠性下降，因为不支持两个进程同事执行写操作，否则导致一定几率的数据丢失，这是因为SP底层是通过读写XML文件来实现的，并发写显然可能出问题
4. Application多次创建。一个组件跑在一个新的进程的时候，由于系统要创建新的进程同时分配独立的虚拟机，所以这一个过程其实就是启动一个应用的过程，因此开启一个进行相当于重新启动一遍应用。既然重新启动，Application自然会新建一个。**运行在同一个进程中的组件属于同一个虚拟机和Application，同理，运行在不同的进程中的组件是属于不同的虚虚拟机和Application的**

跨进程通信的方式：
* 通过Intent来传递数据
* 共享文件
* SharedPreferences
* 基于Binder的Messager
* AIDL
* Socket

# IPC基础概念介绍
主要包括三方面，Serializable接口，Parcelable接口以及Binder，Serializable接口和Parcelable接口可以完成对象的序列化过程，当我们需要通过Intent和Binder传输数据的时候，就需要使用Parcelable或者Serializable。
## Serializable接口
java提供的序列化接口，是一个空接口，为对象提供标准的序列化和反序列化操作，不声明这个serialVersionUID也可以实现序列化，但是会对反序列化过程产生影响。
```java
 //序列化过程
        User user=new User(1,"jack",true);
        try {
            ObjectOutputStream oos=new ObjectOutputStream(new FileOutputStream("cache.txt"));
            oos.writeObject(user);
            oos.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

        //反序列化过程
        try {
            ObjectInputStream ois=new ObjectInputStream(new FileInputStream("cache.txt"));
            User newUser= (User) ois.readObject();
            ois.close();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
```
反序列化和序列化的过程，回复后的newUser对象和user的内容完全一样，但是二者并不是同一个对象
### serialVersionUID 是个毛线
serialVersionUID 是用来辅助序列化和反序列化过程的，原则上序列化后的数据中的serialVersionUID只有和当前类的serialVersionUID相同才能够正常被反序列化，
**serialVersionUID工作机制：** 序列化的时候系统会吧当前类的serialVersionUID写入序列化的文件中。当反序列化的时候系统会去检测文件中的serialVersionUID，看它是否和当前类的serialVersionUID 一致，如果一致就说明序列化的类的版本和当前类的版本相同，这个时候可以成功反序列化。否则说明当前类和序列化的类发生某些变化，比如成员变量的数量，类型等等，这个时候回无法正常反序列化，会报错
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE62880fc65c8c7b1719b44c344a158b66/3065)
一般来说需要手动指定serialVersionUID的值，如果不手动指定serialVersionUID的值，反序列化是当前类有所改变，那么系统会重新接受当前类的hash值并赋值给serialVersionUID。这个时候当前类的serialVersionUID就和序列化的数据serialVersionUID不一致，反序列化是吧，程序crash，**手动指定serialVersionUID，可以避免反序列化过程的失败**

注意：
1. 静态成员变量属于类但是不属于对象，不会参与序列化过程
2. 用transient关键字标记的成员变量不参与序列化过程
3. 默认的序列化过程是可以改变的，只需要重写系统默认的序列化和反序列化方法就可以了
```java
private void writeObject(ObjectOutputStream out)throws IOException{
	//write this to out
}
private void readObject(ObjectInputStream in)throws IOException,ClassNotFoundException{
	//populate the fields of this from the data in 'in'
}
```
##Parcelable 接口
也是一个接口，只要实现这个接口，一个类的对象就可以实现序列化并可以通过Intent和Binder传递,
![](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCEc2172767018a3c67604bf04628f44ecd/3071)
![](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE4017b2539462092c42fb79f8a0f7992b/3073)
Pacel 内部包装了可序列化的数据，可以在Binder中自由传输，
* 序列化过程是有writeToParcel方法完成。最终是通过Pacel中的一系列write方法来完成的，
* 反序列化功能有CREATOR来完成，内部表明了如何创建序列化对象和数组，并通过Parcel的一系列read方法来完成反序列化过程，
* 内容表示功能有describeContents方法来完成，*几乎在所有情况下这个方法都返回0，仅当当前对象中存在文件描述符时，此方法返回1*
* 由于book是另外一个可序列化对象，所以他的反序列化过程需要当前线程的上下文类加载器，

Parcelable的方法说明

| 方法      |     功能 |   标记位   |
| :-------- | :--------| :------ |
| createFromParcel(Parcel in)    |   从序列化后的洗创建原始对象 |    |
| newArray(int size)    |   创建指定长度的原始对象数组 |    |
| User2(Parcel in)    |   从序列化后的对象中创建原始对象 |  field3  |
| writeToParcel(Parcel out, int flags)    |   将当前对象写入序列化结构中，其中flag表示有两种值：0或者1，为1 时标识当前对象需要作为返回值返回，不能立即释放资源，几乎所有情况为0 |  PARCELABLE_WRITE_RETURN_VALUE  |
| describeContents()    |  返回当前对象的内容描述，如果含有文件描述，返回1，否则返回0，几乎所有情况都返回0 |  CONTENTS_FILE_DESCRIPTOR  |


系统已经为我们提供了许多实现Parcelable接口的类，他们都可以直接序列化，比如Intent，Bundle，Bitmap，等，同事List和Map也可以实现序列化，前提是他们里面的子元素都是可序列化的
### Parcelable和Serializable的区别
1. Serializable是java中 序列化接口，使用简单但是开销很大，序列化和反序列化过程需要大量I/O操作，
2. Parcelable 是Android中的序列化方式，使用起来稍微麻烦，但是效率很高，推荐使用parcelable
3. 使用Parcelable 将对象序列化到存储设备中或者将对象序列化后通过网络传输，过程复杂，建议在这种情况下使用Serializable

## Binder
* 是Android的一个类，实现了IBinder接口
* 从IPC角度说，是Android的一种跨进程通信方式，在Linux中没有
* 还可以理解为一种虚拟的物理设备，设备驱动在/dev/binder,
* 从Android Framework角度来说，是ServiceManager连接各种Manager（ActivityManager，WindowManager等）和形影的ManagerService的桥梁
* 从Android应用层来说，是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务器提供的服务或者数据了，这里的服务包括普通的服务和基于AIDL的服务

Android开发中，Binder主要用于在Service中，包括AIDL和Messager，其中普通服务中的Binder不涉及到进程通信，而Messager底层是AIDL技术。

### Android Studio 中创建AIDL文件
1. AndroidStudio的aidl文件默认放在src/main/aidl目录下，aidl目录和java目录同级别。
2. 在java目录上右键，创建一个aidl文件，此文件会默认生成到aidl目录下。
3. 同时必须要指明包名，包名必须和java目录下的包名一致。
4. 如果aidl需要使用Model类，Model类必须要实现Parcelable接口！必须要import进来，不然会找不到
5. 然后Make一下，就会自动生成Java文件。

生成的AIDL 文件中 的内部类Stub，就是Binder类，

![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCEd0a4586622f56cc169fb7426dc19acbd/3067)
由上图结构可以看出，IBookManager中有两个方法，也就是我们在AIDL中定义的那两个方法，addBook(),getBookList(),还有一个内部类Stub，
```java
    public static abstract class Stub extends android.os.Binder implements com.hoyouly.android_art.IBookManager {

```
由于这个内部类是继承Binder，那么她就是Binder类，内部类中还有一个内部类Proxy,
```java
private static class Proxy implements com.hoyouly.android_art.IBookManager
```
内部类Proxy实现我们定义的AIDL类
当客户端和服务端都位于同一个进程的时候，方法调用不会走跨进程的transact方法，而当两者位于不同的进程时，方法调用执行transact方法，而这个逻辑由Stub的内部代理类Proxy来完成，所以，**这个接口的核心实现就是内部类stub和Stub的内部代理类Proxy**
### Stub类和Stub的内部代理类Proxy详细介绍
#### DESCRIPTOR
Binder的唯一标示，一般用当前Binder的类名标示，比如
```java
        private static final java.lang.String DESCRIPTOR = "com.hoyouly.android_art.IBookManager";
```
#### asInterface(android.os.IBinder obj)
用于将服务端的Binder对象转换成客户端所需要的AIDL接口类型的对象，这种转换就是区分进程的，如果客户端和服务端唯一同一个进程，那么次方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.proxy对象
#### asBinder
返回当前Binder对象
#### onTransact
在`服务端的Binder线程池`中，当客户端发起跨进程请求的时，远程请求会通过呢系统底层封装后交由次方法来处理，方法全名是：
```java
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
```
服务端通过code可以确定客户端所请求的目标方法，接着从data中取出目标方法所需要的参数（如果有的话），然后执行目标方法，当执行完毕，就向reply中写入返回值（如果有返回值）。如果次方法返回false，那么客户端的请求就会失败，我们可以利用这个特性来做权限验证，

#### Proxy#getBookList
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
当客户端远程调用次方法是，它的内部实现：
1. 首先创建该方法所需要的输入型Parcel对象_data,输出型Parcel对象_reply,和返回值对象list，
2. 把该方法的参数信息写入_data，
3. 调用transact方法来发起RPC（远程过程调用）请求，同时当前线程挂起，然后服务器的onTransact方法会被调用，直到RPC过程返回，当前线程继续执行，
4. _reply中读取RPC过程的返回结果，
5. 返回_reply中的数据
#### Proxy# addBook
运行在客户端，执行过程和getBookList是一样的，由于没有返回值，不需要从_reply中读取返回值

**总结：**AIDL文件并不是实习Binder的必需品，AIDL文件的本质是系统为我们提供了一种快捷实习Binder的工具，仅此而已
# Android中的IPC方式
## 使用Bundle
四大组件中的三个（Activity，Service，Receiver）都支持在Intent中传递Bundle数据，由于Parcelable接口，可以方便在不同进程之间传输，
A进程需要把计算的结果传递给B进程中的一个组件，可是这个结果不支持Bundle，可以使用曲线救国方式。
通过Intent启动进程B点一个Service组件，然后在这个组件中计算，计算后再启动B进行真正要启动的目标组件，由于Service也在B进程中，所以目标组件可以直接获取计算结果。这种方式的核心思想：将原本属于A进程的计算任务转移到B进程的后台Service中执行。这样就成功避免进程通信的问题，而且代价很小
## 文件共享
共享数据是对文件格式没有具体要求的，只要是读写双方约定的数据格式即可。文件共享局限性：比如并发读写问题，适合在对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读写问题
不建议在进程通信中使用SharedPreference，因为面对高并发的读写，SharedPreference有很大的几率会丢失数据。
## 使用Messager
信使。通过它可以在不同进程中传递Message对象，在Message中存放我们需要传递的数据，这样就可以轻松实现进程传递过程。轻量级的IPC方案，底层实现是AIDL，由于它一次处理一个请求，不需要考虑线程同步问题。
实现步骤：
1. 服务端进程
	1. 在服务端创建Service来处理客户端的连接请求，
	2. 创建Handler并通过它来创建一个Messager对象，
	3. 在onBind中返回这个Messager底层的Binder即可
 2. 客户端进程
	 1. 绑定服务端service，绑定成功后用服务端的IBinder对象创建一个Messager
	 2. 通过这个Messager就可以向服务端发送消息。消息类型为Message对象

在Messenger中进行数据传递必须将数据放入到Message中，而Messenger和Message都实现了Parcelable接口，因此可以跨进程传输，
Messenger是以串行方式处理客户端发来的消息，如果大量的消息同事发送到服务器，服务器任然只能一个个处理，这样就不适合大量并发请求。主要作用是为了传递消息，不能夸进程调用服务端的方法，
## AIDL
Messenger底层是AIDL，AIDL可以实现夸进程的方法调用。
### AIDL支持的数据类型
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
	* in  输入型参数
	* out 输出型参数
	* inout 输入输出型参数
* AIDL接口中只支持方法，不支持声明静态常量。
* 建议把所有和AIDL相关的类和文件全部都放入同一个包中，好处就是当客户端是另外一个应用时，可以直接把整个包复制到客户端工程中。

CopyOnWriteArrayList 支持并发读/写,AIDL方法砸服务端的Binder线程池中执行，因此当多个客户端同时连接的时候，会存在多个线程同时访问的情形，所以我们要在AIDL方法中处理线程同步问题，而使用CopyOnWriteArrayList 可以进行自动的线程同步


ADIL权限验证
1. 在onBind中进行，验证不通过就直接返回null，这样验证失败的客户端直接无法连接服务，验证方式： 可以使用permission 验证。
2. 在服务端的onTransace 方法中进行权限验证，如果验证失败就返回false，这样服务端就不会终止执行AIDL的方法从而达到保护服务端的效果，验证方式：可以通过使用Permission 验证，还可以通过Uid和Pid来验证，

## ContentProvider
Android中专门用于不同应用程序间进行数据共享的方式，天生适合进程通信，底层实现Binder，

### 创建ContentProvider
继承ContentProvider,并实现六个抽象方法，onCreate,query,update,insert,delete和getType
* onCreate 代表ContentProvider的创建，一般是需要我们做一些初始化工作
* getType用来返回一个Uri请求对应的MIME类型，比如图片，视频等，如果应用程序不关心这个选项，可以返回null或者`*/*`,
* 剩下的四个对应着CRUD的操作

除了onCreate又系统调用并运行在主线程中，其他五个方法都是由外界调用，并运行在Binder线程池中
虽然ContentProvider底层数据看起来像是一个SQLite数据库，但是ContentProvider对底层的数据存储方式没有任何要求。我们即可以使用SQLite数据库，也可以使用普通文件，绳子可以采用内存中的一个对象进行数据的存储。

注册ContentProvider，其中`android:authorities是Contentprovider的唯一标识,因此android:authorities必须唯一，`建议命名的时候加上包名前缀，

ContentProvider 通过Uri来区分外界要访问的数据集合，为了知道外界要访问的是哪个表，我们需要为他们定义单独的Uri和Uri_code,并将Uri和Uri_Code想关联，可以使用UriMatcher的addURI方法将Uri和Uri_code 关联到一起，
## Socket
* TCP协议 面向连接的协议，提供稳定的双向通信功能，建立连接需要经过`三次握手`才能完成,
* UDP 协议，无连接，提供不稳定的单项通行功能，效率更好，但是不能保证数据一定能够正确传输。

#Binder连接池
如何使用AIDL
1.  创建一个Service和一个AIDL接口
2.  创建一个类继承AIDL接口中的Stub类并实现Stub中的抽象方法
3.  在Service的onBind方法中返回这个类对象
4.  客户端绑定服务端Service。
5.  建立连接后就可以访问远程服务端的方法了

Binder连接池的主要作用：将每个业务模块的Binder请求统一转发到远程Service中执行。从而避免了重复创建Service的过程。
# 选用合适的IPC方式
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCEa054370d7dd6be4ab1dcc7f40b7c30a5/3069)



Client中的Binder也可以看作是Server Binder的‘代理’，在本地代表远端Server为Client提供服务。
面向对象思想的引入**将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法**，而其独特之处在于**Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中。**
最诱人的是，这个引用和java里引用一样既可以是强类型，也可以是弱类型，而且可以从一个进程传给其它进程，让大家都能访问同一Server，就象将一个对象或引用赋值给另一个引用一样。Binder模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行于同一个面向对象的程序之中。

Binder框架定义了四个角色：Server，Client，ServiceManager（以后简称SMgr）以及Binder驱动。其中Server，Client，SMgr运行于用户空间，驱动运行于内核空间。这四个角色的关系和互联网类似：Server是服务器，Client是客户终端，SMgr是域名服务器（DNS），驱动是路由器。
尽管名叫‘驱动’，实际上和硬件设备没有任何关系，只是实现方式和设备驱动程序是一样的：

SMgr的作用是**将字符形式的Binder名字转化成Client中对该Binder的引用，使得Client能够通过Binder名字获得对Server中Binder实体的引用。注册了名字的Binder叫实名Binder，**

Server创建了Binder实体，为其取一个字符形式，可读易记的名字，将这个Binder连同名字以数据包的形式通过Binder驱动发送给SMgr，

Binder基于Client-Server通信模式，传输过程只需一次拷贝，为发送发添加UID/PID身份，既支持实名Binder也支持匿名Binder，安全性高。
要想实现Client-Server通信据必须实现以下两点：
1. server必须有确定的访问接入点或者说地址来接受Client的请求，并且Client可以通过某种途径获知Server的地址；
2. 制定Command-Reply协议来传输数据。例如在网络通信中Server的访问接入点就是Server主机的IP地址+端口号，传输协议为TCP协议。

对Server而言，Binder可以看成Server提供的实现某个特定服务的访问接入点， Client通过这个‘地址’向Server发送请求来使用该服务；对Client而言，Binder可以看成是通向Server的管道入口，要想和某个Server通信首先必须建立这个管道并获得管道入口。

Client通过Binder的引用访问Server
