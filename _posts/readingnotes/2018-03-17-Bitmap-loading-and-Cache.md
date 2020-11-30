---
layout: post
title: Bitmap 的加载和 Cache 处理
category: 读书笔记
tags: Bitmap  Android开发艺术探索 LruCache LinkedHashMap
description: Bitmap 的加载和 Cache 处理
---

<!-- * content -->
<!-- {:toc} -->

# Bitmap 高效加载
核心思想：采用BitmapFactory.Options 来加载所需要的尺寸的图片，一般都是缩放图片，即用到了 inSmapleSize 参数
1. 将BitmapFactory.Options的inJustDecodeBounds=true，并加载图片
2. 从BitmapFactory.Options中取出图片的原始宽高信息，
3. 采用采样率的规则并结合目标 View 的所需要大小计算出采样率inSampleSize
4. BitmapFactory.Options的inJustDecodeBounds=false，重新加载图片

**inJustDecodeBounds 设置为 true 的时候， BitmapFactory 值会解析图片的原始宽高信息，并不会真正的加载图片。 所以这个操作是轻量级的。 BitmapFactory 获取宽信息和图片的位置以及程序运行的设备有关，比如同一张图片放在不同的 drawable 目录下或者程序运行在不同屏幕设备上，这都可能导致 BitmapFactory 获取到不同的结果，这和 Android 的资源加载机制有关**
# Android 中的缓存策略
一般来说，缓存策略主要包括缓存的添加，获取和删除这三类操作。目前常用的缓存算法就是LRU
* LRU : 全称 Least Recently Used ,即最近最少使用，一种非常常用的置换算法，即淘汰最长时间未使用的对象.    
  他的核心思想是当缓存满时，优先淘汰那些近期最少使用的缓存对象    

采用 LRU 算法缓存有两种，LruCache（实现内存缓存）和DiskLruCache（存储设备缓存）,二者结合，就可以很方便的实现一个完美的ImageLoader
## LruCache
1. android 3.1 提供的缓存类， v4 包中也有，兼容之前的版本，使用 v4 包
2. 属于一个泛型类，内部采用 LinkedHashMap ，以强引用的方式存储外界的缓存对象。
  * 强引用：直接的对象引用
  * 软引用：系统内存不足时被 gc 回收
  * 弱引用：会随时被 gc 回收

3. get(key)获取一个缓存对象
4. put(key,bitmap) 添加一个缓存对象
5. remove(key) 删除一个指定的缓存对象

### 实现原理
**核心：** 存在一种数据结构能够基于访问顺序保存访问对象，这种数据结构就是 LinkedHashMap ,双向循环列表，在构造函数中，通过 boolean 值来指定 LinkedHashMap 的保存方式。
```java
/*
* 初始化LinkedHashMap
* 第一个参数：initialCapacity，初始大小
* 第二个参数：loadFactor，负载因子=0.75f
* 第三个参数：accessOrder=true，基于访问顺序；accessOrder=false，基于插入顺序
*/
public LinkedHashMap(int initialCapacity, float loadFactor , boolean accessOrder) {
  super(initialCapacity, loadFactor);
  init();
  this.accessOrder = accessOrder;
}
```
显然在 LruCache 中， accessOrder 值为 true ，每当我们更新（调用put()）或者访问（get()）map中的结点时， LinkedHashMap 会将这个结点移动的链表的尾部。

因此<span style="border-bottom:1px solid red;"> LinkedHashMap 中尾部则是最近刚刚使用的结点，头部则是最近很少使用的结点</span>。当内存不足的时候，把 LinkedHashMap 头部的结点删除，直到有剩余空间放置新的结点

LinkedHashMap 完成了 LRUCache 的核心功能，而 LruCache 要做的就是
* 定义缓存空间，
* 保存当前数据已使用的容量
* 对外提供 get() ，put()

### 源码解析

#### 关键字段
 LinkedHashMap ，总容量，已使用容量
```java
//核心数据结构
private final LinkedHashMap<K, V> map;
// 当前缓存数据所占的大小
private int size;
//缓存空间总容量
private int maxSize;
```
**注意：** size字段，由于 map 中的数据类型是不定的，这些数据测量的大小方式也是不定的，比如 Bitmap 类型的数据和 String 类型的数据计算方式肯定不同，因此需要复写 sizeOf 方法，自己定义数据的测量方式，所以就有了经常看到的创建 LruCache 对象的方式
```java
private static final int CACHE_SIZE = 4 * 1024 * 1024;//4Mib
    LruCache<String,Bitmap> bitmapCache = new LruCache<String,Bitmap>(CACHE_SIZE){
        @Override
        protected int sizeOf(String key, Bitmap value) {
            return value.getByteCount();//自定义 Bitmap 数据大小的计算方式
        }
    };
```
#### 构造方法
```java
public LruCache(int maxSize) {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }
    this.maxSize = maxSize;
    this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
}
```
做了以下几点事情，
1. 通过构造函数传递过来，给定缓存空间的总大小，maxSize
2. 初始化核心数据结构 LinkedHashMap ，并设置accessOrder=true，基于访问顺序排序

#### sizeOf()和safeSizeOf()
safeSizeOf() 是 sizeOf() 的进一步封装，其实就是计算 sizeOf() 的大小，而由于数据结构不定， sizeOf() 必须由使用者自己去定义，所以我们在创建 LruCache 对象的时候复写 sizeOf() ,
```java
private int safeSizeOf(K key, V value) {
    int result = sizeOf(key, value);
    if (result < 0) {
        throw new IllegalStateException("Negative size: " + key + "=" + value);
    }
    return result;
}
```
#### put() 缓存数据
根据对应的 key 缓存 Value ，并且将该 value 移动到链表的尾部，返回的是该 key 对应之前的前一个value
```java
public final V put(K key, V value) {
    if (key == null || value == null) {
        throw new NullPointerException("key == null || value == null");
    }

    V previous;
    synchronized (this) {
      // 记录 put 的次数
      putCount++;
      // 通过键值对，计算出要保存对象 value 的大小，并更新当前缓存大小
      size += safeSizeOf(key, value);
      // 如果 之前存在 key ，用新的 value 覆盖原来的数据， 并返回 之前 key 的 value ，用 previous 保存
      previous = map.put(key, value);
      // 如果之前存在 key ，并且之前的 value 不为null
      if (previous != null) {
          // 计算出 之前 value 的大小，因为前面 size 已经加上了新的 value 数据的大小
          //此时，需要再次更新 size ，减去原来 value 的大小
          size -= safeSizeOf(key, previous);
      }
    }

    // 如果之前存在 key ，并且之前的 value 不为null
    if (previous != null) {
        //此次添加的 value 已经作为 key 的 新值， 告诉 自定义 的 entryRemoved 方法， previous 值被剔除了
         entryRemoved(false, key , previous , value);
    }
    //裁剪缓存容量（在当前缓存数据大小超过了总容量 maxSize 时，才会真正去执行LRU）
    trimToSize(maxSize);
    return previous;
}
```
主要进行了以下几步：
1. 判断 key 和 Value 是否为 null ，如果一个为 null ，直接抛出空指针异常，这也就是说明了 LruCache 中不允许 key 或者 value 为null
2. 通过 safeSizeOf() 方法获得要保存数据的大小，并更新当前缓存数据的大小（增加）
3. 将当前数据放到缓存中，即调用 LinkHashMap 的 put() ，如果返回的值不为 null ，说明该 key 之前已经保存，那么替换原来的 value 值，并返回原来的 value ，得到之前 value 的大小，再次更新当前缓存数据的大小（减小）
4. 清理缓存空间

#### trimToSize() 清理缓存空间

当我们添加一条数据的时候，为了保证当前数据缓存大小没有超过我们指定的总大小，通过调用 trimToSize() 来进行缓存空间进行管理，
```java
public void trimToSize(int maxSize) {
    // 循环进行 LRU ，直到当前所占容量大小没有超过指定的总容量大小
    while (true) {
        K key;
        V value;
        synchronized (this) {
            // 一些异常情况的处理
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(getClass().getName()
                           + ".sizeOf() is reporting inconsistent results!");
            }
            // 首先判断当前缓存数据大小是否超过了指定的缓存空间总大小。
            // 如果没有超过，即缓存中还可以存入数据，直接跳出循环，清理完毕
            if (size <= maxSize || map.isEmpty()) {
                break;
            }
            // 执行到这，表示当前缓存数据已超过了总容量，
            // 需要执行 LRU ，即将最近最少使用的数据清除掉，直到数据所占缓存空间没有超标;
            // 根据前面的原理分析知道，在链表中，链表的头结点是最近最少使用的数据，
            //因此，最先清除掉链表前面的结点
            Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
            key = toEvict.getKey();
            value = toEvict.getValue();
            map.remove(key);
            // 移除掉后，更新当前数据缓存的大小
            size -= safeSizeOf(key, value);
            // 更新移除的结点数量
            evictionCount++;
        }
        // 通知某个结点被移除，类似于回调
        entryRemoved(true, key , value , null);
    }
}
```
**作用：** 保证当前缓存数据大小不能超过我们指定的缓存总大小，如果超过，则移除最近最少使用的对象，即链表头部的，直至 size 小于缓存总大小，在 put() 方法中一定会调用，但是 get() 方法中不一定(重写了 create() 方法后，在 get 方法会执行调用到)
#### get()获取缓存数据
*  根据 key 查询缓存，如果该 key 对应的 value 存在于缓存，直接返回value；
*  访问到这个结点时， LinkHashMap 会将它移动到双向循环链表的的尾部。
*  如果没有缓存的值，则返回 null 。（如果开发者重写了 create() 的话，返回创建的value）

```java
public final V get(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }
    V mapValue;
    synchronized (this) {
        // LinkHashMap 如果设置按照访问顺序的话，这里每次 get 都会重整数据顺序
        mapValue = map.get(key);
        // 计算 命中次数
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        // 计算 丢失次数
        missCount++;
    }

    /*
     * 官方解释：
     * 尝试创建一个值，这可能需要很长时间，并且 Map 可能在 create() 返回的值时有所不同。
     * 如果在 create() 执行的时候，用这个 key 执行了 put 方法，那么此时就发生了冲突，
     * 我们在 Map 中删除这个创建的值，释放被创建的值，保留 put 进去的值。
     */
    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }

    /***************************
     * 不覆写 create 方法走不到下面 *
     ***************************/
    /*
     * 正常情况走不到这里
     * 走到这里的话 说明 实现了自定义的 create(K key) 逻辑
     * 因为默认的 create(K key) 逻辑为null
     */
    synchronized (this) {
        // 记录 create 的次数
        createCount++;
        // 将自定义 create 创建的值，放入 LinkedHashMap 中，如果 key 已经存在，会返回 之前相同 key 的值
        mapValue = map.put(key, createdValue);
        // 如果之前存在相同 key 的 value ，即有冲突。
        if (mapValue != null) {
            // 有冲突,所以 撤销 刚才的 操作, 将 之前相同 key 的值 重新放回去
            map.put(key, mapValue);
        } else {
            // 拿到键值对，计算出在容量中的相对长度，然后加上
            size += safeSizeOf(key, createdValue);
        }
    }
    // 如果上面 判断出了 将要放入的值发生冲突
    if (mapValue != null) {
        // 刚才 create() 的值被删除了，原来的 之前相同 key 的值被重新添加回去了, 告诉 自定义 的 entryRemoved 方法
        entryRemoved(false, key , createdValue , mapValue);
        return mapValue;
    } else {
        // 上面 进行了 size += 操作 所以这里要重整长度
        trimToSize(maxSize);
        return createdValue;
    }
}
```
主要进行了功能
1. 先尝试从 map 中根据 key 取得对应的 value ，如果得到说明有缓存对象，直接返回，
2. 如果value==null，需要看是否重写了 create() 方法，因为 create() 方法默认是返回 null ，如果重写了 create() 方法，并且不为 null ，那么就会在没有缓存的时候就自己创建一个，然后继续往下走，这个时候还需要解决冲突问题
3. 通过 LinkHashMap 进行 get() 或者 put 操作的结点会被调整到链表尾部

**疑问：** 为什么重写了 create() 后，还会有冲突? 换句话说，什么情况下，mapValue = map.put(key, createdValue)中的 mapValue 值会不为 null 呢。如果不为 null ，那么是什么时候存放进去的呢？因为在 get() 方法一开始执行的时候，根据 key 从 map 集合中是得不到 value 的，中间没有对集合进行任何操作啊

**解答：** 在 get 中，使用 create() 尝试创建一个值，这可能需要很长时间，同时在 get 中调用 create() 时还没有进行 synchronized 同步，因此此时也是线程不安全的，可能在 create() 执行的时候，另一个线程用这个 key 执行了 put 方法，那么此时这个 key 中就有对应的值了，当前面调用 get 的线程执行完 create 之后，进行 put() 操作，此时 mapValue 是不为 null 的，也就发生了冲突，我们就把通过 create 创建的这个 createdValue 作为一个“脏数据”丢弃它，即释放被创建的值，保留另一个线程已经 put 进去的值。


#### entryRemoved()
* 当被回收或者删掉时调用。该方法当 value 被回收释放存储空间时被 remove 调用,或者替换条目值时 put 调用，默认实现什么都没做。
* 该方法没用同步调用，如果其他线程访问缓存时，该方法也会执行。
开发者根据需求是否重写改方法处理自己的逻辑，可以进行的一些操作包括
1. 资源的回收
2. <span style="border-bottom:1px solid red;">实现二级缓存。</span>   
   **思路：** 重写 entryRemoved() 方法，把删除掉的 item 再次存入另外一个`LinkedHashMap<String, SoftWeakReference<Bitmap>>`中，这个数据当做二级缓存，每次获得图片的时候，先判断 LruCache 中是否存在缓存，如果没有的话，判断这个二级缓存中是否有，如果都没有在从 SDcard 中获取，如果 SDCard 中也不存在，那么直接从网络中获取，    
   
```java
/**
  * evicted=true：如果该条目被删除空间 （表示 进行了trimToSize or remove）
  * evicted=false， put 冲突后 或 get 里成功 create 后 导致 newValue !=null，那么则被 put() 或 get() 调用。
**/
protected void entryRemoved(boolean evicted, K key , V oldValue , V newValue) {
}
```
在 LruCache 中有四个地方进行了调用：put()、get()、trimToSize()、remove()中进行了调用。

#### 线程安全
由于在put、get、trimToSize、remove的方法中都加入 synchronized 进行同步控制。所以 LruCache 是线程安全的

### 总结
#### 使用注意
1. 在构造函数中需要提供一个总的缓存大小
2. 复写 sizeOf() 方法，对存入 map 的数据自定义测量大小
3. 根据需求，决定是否重写 entryRemoved() 方法
4. 使用 LruCache 中的 put 方法和 get 方法进行数据的缓存

#### 小结
1. LruCache本身没有释放内存，只是 LinkedHashMap 把数据移除了，如果数据在其他地方被引用，还是会引起内存泄露，还需要手动释放内存
2. 重写 entryRemoved() 方法能知道 LruCache 数据是否发送了冲突，也可以手动释放资源

## DiskLruCache
用于实现储存设备缓存，即磁盘缓存，通过将缓存写文件系统从而实现缓存的效果。不属于 Android SDK 的一部分，并没有集成到 Android 源码中

### DiskLruCache实现原理
DiskLruCache 缓存目录中有一个 journal 文件，如果一个 APP 的缓存文件中存在 journal 文件文件，就可以断定使用了 DiskLruCache 缓存策略。<font color="#ff000" > journal文件 是 DiskLruCache 的核心</font>

DiskLruCache 也是使用的 LRU 算法，所以也是用到了 LinkedHashMap 数据结。

但是单单使用 LinkedHashMap 还是不够，因为我们不能直接把 value 存放到 LinkedHashMap 的 value 中，因为数据是缓存到本地磁盘中的， LinkedHashMap 中的 value 保存的只是一些简要的 Entry 信息，包括唯一文件名称，大小，是否可读等信息，

```java
private final class Entry {
    private final String key;
    /** Lengths of this entry's files. */
    private final long[] lengths;
    /** True if this entry has ever been published */
    private boolean readable;
    /** The ongoing edit or null if this entry is not being edited. */
    private Editor currentEditor;
    /** The sequence number of the most recently committed edit to this entry. */
    private long sequenceNumber;
    private Entry(String key) {
        this.key = key;
        this.lengths = new long[valueCount];
    }
        ...
}
```
在 LruCache 中数据是直接缓存到内存中的，但是在 DiskLruCache 中，由于数据是保存到本地上的，相当于永久保存的文件，即使程序退出也还存在，因此，<span style="border-bottom:1px solid red;"> 在获取 DiskLruCahce 实例的时候，会读取 journal 这个日志文件，根据这个日志文件中的信息，建立 map 的初始信息，同时会根据 journal 这个日志文件，维护本地缓存文件</span>

```java
 /**
  * @param directory  磁盘缓存在文件系统中的路径，
           可以是 SD 卡缓存目录（sdcard/Android/data/package_name/cache），也可以是 SD 卡的其他目录
  * @param appVersion   版本号，如果版本号发生改变，会清空之前所有的缓存文件
  * @param valueCount 单个节点所对应的数据的个数，一般设置为 1 即可
  * @param maxSize 表示缓存的总大小,当缓存大小超过这个设定值，就会清除一些缓存，
  * @throws IOException if reading or writing the cache directory fails
*/
public static DiskLruCache open(File directory, int appVersion , int valueCount , long maxSize)
    throws IOException {
    ...
    // prefer to pick up where we left off
    DiskLruCache cache = new DiskLruCache(directory, appVersion , valueCount , maxSize);
    if (cache.journalFile.exists()) {
    try {
        cache.readJournal();
        cache.processJournal();
        cache.journalWriter = new BufferedWriter(new FileWriter(cache.journalFile, true),IO_BUFFER_SIZE);
        return cache;
    } catch (IOException journalIsCorrupt) {
        cache.delete();
    }
   }

    // create a new empty cache
    directory.mkdirs();
    cache = new DiskLruCache(directory, appVersion , valueCount , maxSize);
    cache.rebuildJournal();
    return cache;
}
```
其中，   
cache.readJournal();   
cache.processJournal();   
这两个方法正是读取 journal 文件，建立 map 的初始数据，维护缓存文件

### journal文件
一个标准的 journal 文件信息如下
```
libcore.io.DiskLruCache    //第一行，固定内容，声明
1                          //第二行， cache 的版本号，恒为1
1                          //第三行， APP 的版本号
2                          //第四行，一个 key ，可以存放多少条数据valueCount    
                           //第五行，空行分割行
DIRTY 335c4c6028171cfddfbaae1a9c313c52
CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934
REMOVE 335c4c6028171cfddfbaae1a9c313c52
DIRTY 1ab96a171faeeee38496d8b330771a7a
CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
READ 335c4c6028171cfddfbaae1a9c313c52
READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
...
```
前五行成为 journal 的日志文件的头，下面部分的每一行会以四种前缀之一开始：DIRTY、CLEAN、REMOVE、READ。    
以 DIRTY 这个这个前缀开头，意味着这是一条脏数据。每当我们调用一次 DiskLruCache 的 edit() 方法时，都会向 journal 文件中写入一条 DIRTY 记录，表示我们正准备写入一条缓存数据，但不知结果如何。   
然后调用 commit() 方法表示写入缓存成功，这时会向 journal 中写入一条 CLEAN 记录，意味着这条“脏”数据被“洗干净了”   
调用 abort() 方法表示写入缓存失败，这时会向 journal 中写入一条 REMOVE 记录。     
也就是说，<span style="border-bottom:1px solid red;"> 每一行 DIRTY 的 key ，后面都应该有一行对应的 CLEAN 或者 REMOVE 的记录，否则这条数据就是“脏”的，会被自动删除掉。在 CLEAN 前缀和 key 后面还有一个数值，代表的是该条缓存数据的大小。</span>


### DiskLruCache的缓存添加
通过 Editor 完成，表示一个缓存对象的编辑对象。
1. 获取图片的 Url 所对应的 key ,原因是图片的 url 中可能有特殊字符，直接使用可能会有影响，一般采用 url 的 MD5 值作为key
2. 根据 key 就可以通过 edit() 获取 Editor 对象，如果这个缓存正在被剪辑，那么 edit() 会返回 null ，不允许同时编辑同一个缓存对象，

### 工作流程
1. 初始化，通过 open 方法，获得 DiskLruCache 实例，在 open 中会通过 readJournal() 读取 Journal 文件信息从而建立 map 的初始数据，然后在调用 processJournal() 方法，对刚刚建立的 map 集合进行分析，包括
* 计算当前有效缓存（即被 CLEAR 的）大小，
* 清理无用缓存文件
2. 数据缓存与获取缓存
数据缓存的操作步骤： 需要通过 DiskLruCache 的 edit() 方法获取DiskLruCache.Editor，写入完成后调用 commit 方法
简单实例
```java
new Thread(new Runnable() {  
    @Override  
    public void run() {  
        try {  
            String imageUrl = "http://img.my.csdn.net/uploads/201309/01/1378037235_7476.jpg";  
            //MD5对 url 进行加密，这个主要是为了获得统一的 16 位字符
            String key = hashKeyForDisk(imageUrl);  
            //拿到 Editor ，往 journal 日志中写入 DIRTY 记录
            DiskLruCache.Editor editor = mDiskLruCache.edit(key);  
            if (editor != null) {  
                OutputStream outputStream = editor.newOutputStream(0);  
                //downloadUrlToStream方法为下载图片的方法，并且将输出流放到outputStream
                if (downloadUrlToStream(imageUrl, outputStream)) {
                     //完成后记得 commit() ，成功后，再往 journal 日志中写入 CLEAN 记录
                    editor.commit();  
                } else {  
                    //失败后，要 remove 缓存文件，往 journal 文件中写入 REMOVE 记录
                    editor.abort();
                }  
            }  
            //将缓存操作同步到 journal 日志文件，不一定要在这里就调用
            mDiskLruCache.flush();  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
}).start();
```

注意：
每次调用 edit() 方法，会向 journal 文件中写入 DIRTY 为前缀的一条记录；

文件保存成功，调用 commit 方法，会向 journal 文件中写入 CLREA 为前缀的一条记录，

如果保存失败，需要调用 abort() 方法，这个时候会向 Journal 文件中写入 REMOVE 为前缀的一条记录

### 合适的地方进行flush()
文件写入时通过 IO 的 Writer 写入的，要想生效，还需要调用 writer 的 flush() ,而 DiskLruCache 中的 flush() 方法中封装了writer.flush()的操作，由于这是一个消耗 IO 的操作，不必每次往 Journal 写入的时候就 flush ，这样对效率影响很大，只需要在合适的地方执行一次就可以了，可以在 Activity 的 onPause 中调用一些就可以了

### 注意和小结
1. 可以在 UI 线程中检测内存缓存，即可以在主线程中使用LruCache
2. 使用 DiskLruCache 时，由于需要对本地文件进行操作，需要在另外一个线程中执行，在子线程中检测磁盘缓存、保存缓存数据，磁盘操作从来不应该在 UI 线程中实现；
3. LruCache内存缓存的核心是 LinkedHashMap ，而 DiskLruCache 的核心是 LinkedHashMap 和 journal 日志文件，相当于把 journal 看作是一块“内存”， LinkedHashMap 的 value 只保存文件的简要信息，对缓存文件的所有操作都会记录在 journal 日志文件中。

### DiskLruCache可能的优化方案
DiskLruCache 是基于日志文件 journal 的，这就决定了每次对缓存文件的操作都需要进行日志文件的记录，我们可以不用 journal 文件，在第一次构造 DiskLruCache 的时候，直接从程序访问缓存目录下的缓存文件，并将每个缓存文件的访问时间作为初始值记录在 map 的 value 中，每次访问或保存缓存都更新相应 key 对应的缓存文件的访问时间，这样就避免了频繁的 IO 操作，这种情况下就需要使用单例模式对 DiskLruCache 进行构造了，
