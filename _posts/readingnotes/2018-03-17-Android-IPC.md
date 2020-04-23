---
layout: post
title: Android IPC 简介
category: 读书笔记
tags: Android开发艺术探索 IPC
description: Android IPC 简介
---

<!-- * content -->
<!-- {:toc} -->

## IPC
IPC ：Inter_Process Communication的缩写，含义进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。
* 线程：CPU调度的最小单位，
* 进程：一般只一个执行单元，在 PC 和移动设备中指一个程序或者应用，一个进程可以有多个线程，
Android 中主线程也叫 UI 线程， UI 线程中不能执行耗时操作，不然会引起 ANR ，耗时操作需要使用线程

**Android中最特色的进程间通信方式就是 Binder ，通过 Binder 可以实现两个进程通信，除了 Binder ， Android 还使用 Socket ，通过 Socket 也可以实现任意两个终端之间的通信，**

多进程情况：
1. 一个应用由于某些原因自己需要采用多进程的方式来实现。
2. 当前应用需要向其他应用获取数据

## Android中的多进程模式
### 开启多进程模式
**Android中只有一种方式：通过给四大组件（Activity， Service ， Receiver ，ContentProvider）在 AndroidManifest 中指定android:process 属性，就可以轻易的开启多进程模式**

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
* 当 SecondActivity 启动的时候，系统创建单独进程，进程名为com.hoyouly.android_art.chap2`:`remote
* 当 ThridActivity 启动的时候，系统创建单独进程，进程名为com.hoyouly.android_art`.`remote
* 由于 MainActivity 没有指定 process 属性，那么她就运行在默认进程中，`默认进程名就是包名，即com.hoyouly.android_art`
查看进程信息的两种方式
1. 在 DDMS 试图中查看
![Alt text](../../../../images/ddms_proces.png)
2. adb 命令：**adb shell ps 或者adb shell ps | grep com.hoyouly**
![Alt text](../../../../images/adb_shell_process.png)

**注意：** 必须是 SecondActivity 或者 ThridActivity 界面启动，才能把进程启动起来，如果只是单纯的在 AndroidManifest 文件中注册，是开启不了多进程的

**`:`remote  和com.hoyouly.android_art.chap2`.`remote  区别**

|区别      |`:`remote|   com.hoyouly.android_art.chap2`.`remote  |
| :--------: | :--------| :------ |
| 区别一    | 指当前进行名前加上附加上当前包名，即com.hoyouly.android_art.chap2`:`remote，这是一种简单写法 |  完整命名方式，不附件包名信息， |
| 区别二    |   属于当前应用的私有进程，其他应用的组件不可以和他泡在同一个进程中 | 全局进程，其他应用通过 ShareUID 方式可以和他跑在同一个进程中 |

### UID
Android 系统会为每个应用分配一个唯一的 UID ，具有相同的 UID 的应用才能共享数据，
两个应用通过 ShareUID 跑在同一个进程中是有要求：这两个应用有相同的 ShareUID 并且签名相同才行。
## 多进程模式的运行机制
Android 为每个应用分配了一个独立的虚拟机，或者说每一个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，**一个进程中修改某个值，只会影响当前进程，对其他进程不会造成影响。**

### 多进程造成的问题
1. 静态成员和单利模式失效
2. 线程同步机制完全失效
3. SharedPreferences的可靠性下降，因为不支持两个进程同时执行写操作，可能会导致一定几率的数据丢失，这是因为 SharedPreferences 底层是通过读写 XML 文件来实现的，并发写显然可能出问题
4. Application多次创建。一个组件跑在一个新的进程的时候，由于系统要创建新的进程同时分配独立的虚拟机，所以这一个过程其实就是启动一个应用的过程，因此开启一个进程相当于重新启动一遍应用。既然重新启动， Application 自然会新建一个。**运行在同一个进程中的组件属于同一个虚拟机和 Application ，同理，运行在不同的进程中的组件是属于不同的虚虚拟机和 Application 的**

### 跨进程通信的方式
* 通过 Intent 来传递数据
* 共享文件
* SharedPreferences
* 基于 Binder 的Messager
* AIDL
* Socket

## IPC基础概念介绍
主要包括三方面， Serializable 接口， Parcelable 接口以及 Binder ， Serializable 接口和 Parcelable 接口可以完成对象的序列化过程，当我们需要通过 Intent 和 Binder 传输数据的时候，就需要使用 Parcelable 或者 Serializable 。
### Serializable接口
java 提供的序列化接口，是一个空接口，为对象提供标准的序列化和反序列化操作，不声明这个 serialVersionUID 也可以实现序列化，但是会对反序列化过程产生影响。
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
反序列化和序列化的过程，恢复后的 newUser 对象和 user 的内容完全一样，但是二者并不是同一个对象
#### serialVersionUID
serialVersionUID 是用来辅助序列化和反序列化过程的，原则上序列化后的数据中的 serialVersionUID 只有和当前类的 serialVersionUID 相同才能够正常被反序列化，

**serialVersionUID工作机制：** 序列化的时候系统会把当前类的 serialVersionUID 写入序列化的文件中。当反序列化的时候系统会去检测文件中的 serialVersionUID ，看它是否和当前类的 serialVersionUID 一致，如果一致就说明序列化的类的版本和当前类的版本相同，这个时候可以成功反序列化。否则说明当前类和序列化的类发生某些变化，比如成员变量的数量，类型等等，这个时候回无法正常反序列化，程序 crash ，如下图所示
![Alt text](../../../../images/serialVersionUID_error.png)

一般来说需要手动指定 serialVersionUID 的值，如果不手动指定 serialVersionUID 的值，反序列化时当前类有所改变，那么系统会重新接收当前类的 hash 值并赋值给 serialVersionUID 。这个时候当前类的 serialVersionUID 就和序列化的数据 serialVersionUID 不一致，**手动指定 serialVersionUID ，可以避免反序列化过程的失败**

注意：
1. 静态成员变量属于类但是不属于对象，不会参与序列化过程
2. 用 transient 关键字标记的成员变量不参与序列化过程
3. 默认的序列化过程是可以改变的，只需要重写系统默认的序列化和反序列化方法就可以了
```java
private void writeObject(ObjectOutputStream out)throws IOException{
  //write this to out
}
private void readObject(ObjectInputStream in)throws IOException,ClassNotFoundException{
  //populate the fields of this from the data in 'in'
}
```

### Parcelable 接口
也是一个接口，只要实现这个接口，一个类的对象就可以实现序列化并可以通过 Intent 和 Binder 传递,如下图就是一个实现 Parcelable 接口的例子
![](../../../../images/Parcelable_1.png)
![](../../../../images/Parcelable_2.png)

#### Parcal
Parcel 内部包装了可序列化的数据，可以在 Binder 中自由传输，
* 序列化过程是由 writeToParcel() 方法完成。最终是通过 Parcel 中的一系列 write 方法来完成的，
* 反序列化功能由 CREATOR 来完成，内部表明了如何创建序列化对象和数组，并通过 Parcel 的一系列 read 方法来完成反序列化过程，
* 内容表示功能由 describeContents() 方法来完成，*几乎在所有情况下这个方法都返回 0 ，仅当当前对象中存在文件描述符时，此方法返回1*
* 由于 book 是另外一个可序列化对象，所以他的反序列化过程需要当前线程的上下文类加载器，

#### Parcelable的方法说明

| 方法      |     功能 |   标记位   |
| :-------- | :--------| :------ |
| createFromParcel(Parcel in)    |   从序列化后的洗创建原始对象 |    |
| newArray(int size)    |   创建指定长度的原始对象数组 |    |
| User2(Parcel in)    |   从序列化后的对象中创建原始对象 |  field3  |
| writeToParcel(Parcel out, int flags)    |   将当前对象写入序列化结构中，其中 flag 表示有两种值：0或者 1 ，为 1 时标识当前对象需要作为返回值返回，不能立即释放资源，几乎所有情况为0 |  PARCELABLE_WRITE_RETURN_VALUE  |
| describeContents()    |  返回当前对象的内容描述，如果含有文件描述，返回 1 ，否则返回 0 ，几乎所有情况都返回0 |  CONTENTS_FILE_DESCRIPTOR  |


系统已经为我们提供了许多实现 Parcelable 接口的类，他们都可以直接序列化，比如 Intent ， Bundle ， Bitmap ，等，同时 List 和 Map 也可以实现序列化，前提是他们里面的子元素都是可序列化的
#### Parcelable和 Serializable 的区别
1. Serializable是 java 中 序列化接口，使用简单但是开销很大，序列化和反序列化过程需要大量I/O操作，
2. Parcelable 是 Android 中的序列化方式，使用起来稍微麻烦，但是效率很高，推荐使用parcelable
3. 使用 Parcelable 将对象序列化到存储设备中或者将对象序列化后通过网络传输，过程复杂，建议在这种情况下使用Serializable

### Binder
详情点击  [Android Binder 总结](../../../../2018/03/17/Android-Binder/)  和   [从一个小例子理解 Binder 整个流程](../../../../2019/10/01/Binder-Demo/)


## Android中的 IPC 方式
### Bundle
&#8195;&#8195;四大组件中的三个（Activity， Service ，Receiver）都支持在 Intent 中传递 Bundle 数据，因为 Bundle 实现了 Parcelable 接口，所以可以方便在不同进程之间传输.

&#8195;&#8195;A进程需要把计算的结果传递给 B 进程中的一个组件，可是这个结果不支持 Bundle ，可以使用曲线救国方式。

&#8195;&#8195;通过 Intent 启动进程 B 点一个 Service 组件，然后在这个组件中计算，计算后再启动 B 进行真正要启动的目标组件，由于 Service 也在 B 进程中，所以目标组件可以直接获取计算结果。

&#8195;&#8195;这种方式的核心思想：将原本属于 A 进程的计算任务转移到 B 进程的后台 Service 中执行。这样就成功避免进程通信的问题，而且代价很小
### 文件共享
共享数据是对文件格式没有具体要求的，只要是读写双方约定的数据格式即可。文件共享局限性：比如并发读写问题，适合在对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读写问题
不建议在进程通信中使用 SharedPreference ，因为面对高并发的读写， SharedPreference 有很大的几率会丢失数据。
### Messager
信使。通过它可以在不同进程中传递 Message 对象，在 Message 中存放我们需要传递的数据，这样就可以轻松实现进程传递过程。轻量级的 IPC 方案，底层实现是 AIDL ，由于它一次处理一个请求，不需要考虑线程同步问题。
实现步骤：
1. 服务端进程
  1. 在服务端创建 Service 来处理客户端的连接请求，
  2. 创建 Handler 并通过它来创建一个 Messager 对象，
  3. 在 onBind 中返回这个 Messager 底层的 Binder 即可
 2. 客户端进程
   1. 绑定服务端 service ，绑定成功后用服务端的 IBinder 对象创建一个Messager
   2. 通过这个 Messager 就可以向服务端发送消息。消息类型为 Message 对象

在 Messenger 中进行数据传递必须将数据放入到 Message 中，而 Messenger 和 Message 都实现了 Parcelable 接口，因此可以跨进程传输，
Messenger 是以串行方式处理客户端发来的消息，如果大量的消息同事发送到服务器，服务器任然只能一个个处理，这样就不适合大量并发请求。主要作用是为了传递消息，不能夸进程调用服务端的方法，
### AIDL
如何使用AIDL
1.  创建一个 Service 和一个 AIDL 接口
2.  创建一个类继承 AIDL 接口中的 Stub 类并实现 Stub 中的抽象方法
3.  在 Service 的 onBind 方法中返回这个类对象
4.  客户端绑定服务端 Service 。
5.  建立连接后就可以访问远程服务端的方法了

详情点击  [Android AIDL 总结](../../../../2019/07/17/Android-AIDL/)

### ContentProvider
Android 中专门用于不同应用程序间进行数据共享的方式，天生适合进程通信，底层实现 Binder 。

继承 ContentProvider ,并实现六个抽象方法， onCreate() , query() , update() , insert() , delete() 和getType()
* onCreate() 代表 ContentProvider 的创建，一般是需要我们做一些初始化工作
* getType()用来返回一个 Uri 请求对应的 MIME 类型，比如图片，视频等，如果应用程序不关心这个选项，可以返回 null 或者`*/*`,
* 剩下的四个对应着 CRUD 的操作

<span style="border-bottom:1px solid red;"> 除了 onCreate() 由系统调用并运行在主线程中，其他五个方法都是由外界调用，并运行在 Binder 线程池中</span>

虽然 ContentProvider 底层数据看起来像是一个 SQLite 数据库，但是 ContentProvider 对底层的数据存储方式没有任何要求。我们即可以使用 SQLite 数据库，也可以使用普通文件，甚至可以采用内存中的一个对象进行数据的存储。

注册 ContentProvider ，其中<font color="#ff000" > android:authorities是 Contentprovider 的唯一标识，因此android:authorities必须唯一</font>，建议命名的时候加上包名前缀.

ContentProvider 通过 Uri 来区分外界要访问的数据集合，为了知道外界要访问的是哪个表，我们需要为他们定义单独的 Uri 和 Uri_code ,并将 Uri 和 Uri_Code 想关联，可以使用 UriMatcher 的 addURI 方法将 Uri 和 Uri_code 关联到一起，

### Socket
* TCP协议 面向连接的协议，提供稳定的双向通信功能，建立连接需要经过`三次握手`才能完成,
* UDP 协议，无连接，提供不稳定的单项通行功能，效率更好，但是不能保证数据一定能够正确传输。

## Binder连接池

Binder 连接池的主要作用：将每个业务模块的 Binder 请求统一转发到远程 Service 中执行。从而避免了重复创建 Service 的过程。
## 选用合适的 IPC 方式
![Alt text](../../../../images/ipc_method.png)

---
搬运地址：    
 
Android 开发艺术探索
