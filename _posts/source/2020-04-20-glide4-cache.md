---
layout: post
title: 源码分析 - Glide4 之 缓存原理
category: 源码分析
tags: glide4
---
<!-- * content -->
<!-- {:toc} -->
# 概况
Glide 缓存分两个模块： 内存缓存和硬盘缓存

## 内存缓存   
* 作用：防止应用重复的把图片读取到内存中。
* 分为两层
1. 弱引用缓存：正在使用的图片使用弱引用缓存，这样的目的保护正在使用的资源不会被 LruCache 算法回收。
2. LruCache缓存：不再使用的图片使用 LruCache 缓存,这是一个强引用缓存

## 硬盘缓存
* 作用：防止应用重复的从网络或者其他地方下载和读取数据。
* 分为两层
1. 已 decode 过的数据，可以直接供 View 去使用
2. 未 decode 的原始数据，从 DataFetcher 拉取的数据,不能直接供 View 使用

## 缓存策略
默认情况下。 Glide 会在开始一个新的 图片请求之前，会进行一系列的缓存检查
1. 活动资源 ActiveResources ,是否有一个 View 正在展示这张图片，也就是弱引用缓存
2. 内存缓存 MemoryCache ,该图片是否最近被加载过并仍存在于内存中，即 LruCache 缓存，以强引用的方式存储外界的缓存对象    
3. 资源类型 ResourceCache ,被解码，转换的资源是否写入过磁盘缓存，这部分资源基于原始数据的，并且可以直接供 View 使用
4. 数据来源 DataCache ,构建这个图片的资源是否之前曾写入过文件缓存，这部分是源数据。并未解码，不能直接供 View 使用

前两步检测图片是否在内存缓存中，如果是，则直接返回。后两步检查图片是否在磁盘上，以便快速但异步的返回图片。如果这四步都没找到图片资源，则 Glide 会返回原始图片的资源以获取数据（原始文件， URL ，URI）.

注：
1. 之前版本是先从 MemoryCache 中查询，再从 ActiveResources 中查询，即先执行步骤 2 ，再执行步骤 1 ，但是在Glide 4.9版本上，是先执行 1 后 2 。
2. LruCache 算法原理就是把最近使用的对象用强引用存储在 LinkedHashMap（双向循环列表）中，并且把最近最少使用的对象在缓存值达到预设定值之前从内存中移除。淘汰最长时间未使用的对象

## 为啥设计成两层内存缓存
ActiveResources 中存放的是所有未被 clear 的 Request 请求到的 Resource ，这部分 Resource 会存放至 ActiveResources 缓存中；   
当 Request 被 clear 的时候，会把这部分在 ActiveResources 缓存中的 Resource 移动至 MemoryCache 中去。   
如果 MemoryCache 中能够命中，这部分 resource 又会从 MemoryCache 移至 ActiveResources 缓存中去。

这样做能够有效的提高访问速度，避免过多的操作 MemoryCache 。因为我们知道， MemoryCache 中存放的缓存可能很多，这样的话，直接在上面做一层 ActiveResources 缓存显得就很有必要了。  

# 源码分析
在 Glide 创建的时候，也就是GlideBuilder.build()中

```java
@NonNull
Glide build(@NonNull Context context) {
  ...
  if (diskCacheExecutor == null) {
    diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
  }
  if (memoryCache == null) {
    memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
  }
  if (diskCacheFactory == null) {
    diskCacheFactory = new InternalCacheDiskCacheFactory(context);
  }
  ...
}
```
* LruResourceCache 赋值到了 memoryCache 这个对象, 这个就是 MemoryCache 的实现。 Glide 实现内存缓存所使用的 LruCache 对象
* InternalCacheDiskCacheFactory是磁盘缓存（内部存储）所使用的工厂对象。同时在其中初始化了磁盘缓存的大小 (250M) 和文件的路径(/data/data/<application package>/cache/image_manager_disk_cache 目录下面)
* MemorySizeCalculator  缓存大小的计算器，用来根据当前设备的环境计算可用的缓存空间，主要针对的是内存缓存。
```java
MemorySizeCalculator(MemorySizeCalculator.Builder builder) {
    this.context = builder.context;
    arrayPoolSize = isLowMemoryDevice(builder.activityManager)
            ? builder.arrayPoolSizeBytes / LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR
            : builder.arrayPoolSizeBytes;
    // 计算 APP 可申请最大使用内存，再乘以乘数因子，内存过低时乘以0.33，一般情况乘以0.4        
    int maxSize =getMaxSize(builder.activityManager, builder.maxSizeMultiplier, builder.lowMemoryMaxSizeMultiplier);
    int widthPixels = builder.screenDimensions.getWidthPixels();
    int heightPixels = builder.screenDimensions.getHeightPixels();
    // 计算屏幕这么大尺寸的图片占用内存大小， config 是 ARGB_8888 ,每个像素占用 4 个字节内存
    int screenSize = widthPixels * heightPixels * BYTES_PER_ARGB_8888_PIXEL;
    // 计算目标位图池内存大小
    int targetBitmapPoolSize = Math.round(screenSize * builder.bitmapPoolScreens);
    // 计算目标 Lrucache 内存大小，也就是屏幕尺寸图片大小乘以2
    int targetMemoryCacheSize = Math.round(screenSize * builder.memoryCacheScreens);
    // 最终 APP 可用内存大小
    int availableSize = maxSize - arrayPoolSize;

    if (targetMemoryCacheSize + targetBitmapPoolSize <= availableSize) {
      // 如果目标位图内存大小+目标 Lurcache 内存大小小于 APP 可用内存大小，则OK
      memoryCacheSize = targetMemoryCacheSize;
      bitmapPoolSize = targetBitmapPoolSize;
    } else {
      // 否则用 APP 可用内存大小等比分别赋值
      float part = availableSize / (builder.bitmapPoolScreens + builder.memoryCacheScreens);
      memoryCacheSize = Math.round(part * builder.memoryCacheScreens);
      bitmapPoolSize = Math.round(part * builder.bitmapPoolScreens);
    }
  }
```

真正加载图片，是在Engin.load()这个方法中的。
## Engine # load()
```java
public synchronized <R> LoadStatus load(...) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
    //创建了一个 EngineKey 对象，这个对象就是我们说的缓存 key ，加载资源的唯一标识,
    EngineKey key = keyFactory.buildKey(model, signature , width , height ,
 transformations , resourceClass , transcodeClass , options);
    //调用 loadFromActiveResources 方法，通过 key 获取缓存资源，此时的缓存也是内存缓存。
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {//获取到的话也直接进行回调。
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      return null;
    }
    //如果 ActiveResources 中没有找到缓存，通过 key 查找 MemoryCache 缓存，
    //此时的缓存依旧为内存缓存，如果获取的到就直接调用cb.onResourceReady()方法进行回调。
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      return null;
    }

    //到这里，就说明内存缓存中没有进行缓存。要么是被移除了，要么是还未加载过，那么就需要去硬盘缓存中查看

    //是否可能该缓存任务正在处理，还没有完成缓存，所以根据 key 判断缓存的 job 中是否有 current ，
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      //如果有，就不用新创建任务了，而是给其添加回调，等待完成后获取。
      current.addCallback(cb, callbackExecutor);
      return new LoadStatus(cb, current);
    }
    //需要创建新的加载任务。并把当前任务存放在 jobs 这个 map 中。同时要开启线程来加载新的图片了。
    EngineJob<R> engineJob = engineJobFactory.build(...);
    DecodeJob<R> decodeJob = decodeJobFactory.build(...);
    //把创建的 加载任务和 key 放到 joes 中，这样避免加载多次
    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);
    return new LoadStatus(cb, engineJob);
  }
```
这里面主要干了一下几件事
1. 根据各种条件，生成一个 EngineKey ,也就是缓存 key ,这个 key 是加载资源的唯一标识。不管是从内存还是从硬盘中读取缓存，都要依靠这个 key 。
2. 执行 loadFromActiveResources() 方法，通过 key 从 ActiveResources 中找是否有缓存资源，如果有则直接执行cb.onResourceReady()返回，否则执行步骤 3 。
3. 执行 loadFromCache() 方法，通过 key 从 MemoryCache 中查找是否有缓存，如果有则执行cb.onResourceReady()返回，否则执行步骤 4 。
4. 判断该缓存任务是否正在处理中，如果是，则不用创建子任务，而是等待完成后获取，否则执行步骤 5 。
5. 创建子任务，从磁盘缓存中获取。


接下来一条条看

## 生成缓存key
```java
EngineKey key = keyFactory.buildKey(model, signature , width , height , transformations ,
 resourceClass , transcodeClass , options);
//生成 EngineKey 很严格，
EngineKey buildKey(Object model, Key signature , int width , int height ,
     Map<Class<?>, Transformation<?>> transformations, Class<?> resourceClass,
     Class<?> transcodeClass, Options options) {
   return new EngineKey(model, signature , width , height , transformations , resourceClass ,
 transcodeClass , options);
 }        
```
看到没， EngineKey 和很多因素有关，即使你用 override() 方法改变了一下图片的 width 或者 height ，也会生成一个完全不同的缓存 Key , key 是在内存缓存中查询内存的关键， key 一变化，那么之前的缓存就白忙乎了。

## 内存缓存读取

### 从 ActiveResources 中读取缓存
也就是 loadFromActiveResources() 方法
```java
private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
    //isMemoryCacheable 内存缓存是否被开启， Glide 默认为true
    if (!isMemoryCacheable) {
      return null;
    }
    //根据缓存 key 查找是否存在 activeResources ，
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
      //存在，引用值加1
      active.acquire();
    }
    return active;
  }
```
isMemoryCacheable 默认是 true 的，可以通过 skipMemoryCache() 设置为 false ，表示禁用内存缓存。

关键方法也就是  activeResources.get(key)
```java
final class ActiveResources {
  final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
  private final ReferenceQueue<EngineResource<?>> resourceReferenceQueue = new ReferenceQueue<>();

    @Nullable
    synchronized EngineResource<?> get(Key key) {
      ResourceWeakReference activeRef = activeEngineResources.get(key);
      if (activeRef == null) {
        return null;
      }
      EngineResource<?> active = activeRef.get();
      if (active == null) {
        cleanupActiveReference(activeRef);
      }
      return active;
    }
  }

//ResourceWeakReference 是一个 WeakReference 的子类,也就是一个弱引用类型，存放的对象就是 EngineResource
static final class ResourceWeakReference extends WeakReference<EngineResource<?>> {
}
```
主要做的就是通过缓存 key 查询是否有对应的值。
* 如果没有，说明没有 ActiveResources 缓存。
* 如果存在，然后判断这个被引用类型是否还存在，因为有可能中间发生了 GC ，导致该被引用对象已经回收了。
  * 如果这个被引用类型存在，直接返回。
  * 如果这个被引用类型不存在了，则调用 cleanupActiveReference() 把这个引用释放掉。他们就没比要待在 resourceReferenceQueue 这个队列中了。
```java
void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
    synchronized (listener) {
      synchronized (this) {
        // activeResources 是一个 Map ,移除键值对。
        activeEngineResources.remove(ref.key);
        //没有设置缓存或者缓存为空，则直接返回
        if (!ref.isCacheable || ref.resource == null) {
          return;
        }
        //虽然 activeRef.get() 为空，但是因为 activeRef 中保存了 resource ，所以 ref.resource不为空
        EngineResource<?> newResource = new EngineResource<>(ref.resource, /*isCacheable=*/ true, /*isRecyclable=*/ false);
        newResource.setResourceListener(ref.key, listener);
        //listener 就是 Engine
        listener.onResourceReleased(ref.key, newResource);
      }
    }
  }
//Engine.java
public synchronized void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
  activeResources.deactivate(cacheKey);
  if (resource.isCacheable()) {
    cache.put(cacheKey, resource);
  } else {
    resourceRecycler.recycle(resource);
  }
}
```

如果 这个 Resource 设置了内存缓存，则把从 ActiveResources 中移除的 Resource ，添加到 MemoryCache 中。这里，也就是往 MemoryCache 中添加缓存的地方。
isCacheable() 通过源码可最终到这个字段最初传入的位置是在 RequestOptions 中，也就是说这个字段是针对一次请求的，并不是一个全局设置。我们可以通过构建 Glide 请求的时候执行 apply() 来设置这个值
```java
Glide.with(this)
  .load(url)
  // 不忽略内存缓存，即启用
  .apply(RequestOptions.skipMemoryCacheOf(false))
  .into(view);
```

### 从 MemoryCache 中读取缓存
```java   
//内存缓存 (Memory cache) - 该图片是否最近被加载过并仍存在于内存中？
private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
  //内存缓存是否被开启，默认是开启
  if (!isMemoryCacheable) {
    return null;
  }
  //使用缓存 Key 来从 cache 当中取值
  EngineResource<?> cached = getEngineResourceFromCache(key);
  if (cached != null) {//
    //调用 acquire() 方法会让变量 acquire 加 1 ，用来记录图片被引用的次数，
    cached.acquire();
    //将这个缓存图片存储到 activeResources 当中, activeResources 就是一个弱引用的 HashMap ，用来缓存正在使用中的图片
    activeResources.activate(key, cached);
  }
  return cached;
}
```
1. acquire() 就是 Lru 算法的关键。 acquire 值越大，越不容易被 LRU 算法清理出去。   
acquire() 让 acquire ++   
release() 刚好相反，让acquire --   
2. 把从 MemoryCache 中得到的缓存 Resource 再添加到 ActiveResources 中，因为已经命中，该 Resource 又会成为被 View 所使用。
3. 关键的是 getEngineResourceFromCache() 这个方法， 根据缓存 key 从 MemoryCache 中取得缓存。

```java
private final MemoryCache cache;

private EngineResource<?> getEngineResourceFromCache(Key key) {
  Resource<?> cached = cache.remove(key);
  final EngineResource<?> result;
  if (cached == null) {
    result = null;
  } else if (cached instanceof EngineResource) {
    result = (EngineResource<?>) cached;
  } else {
    result = new EngineResource<>(cached, true /*isMemoryCacheable*/, true /*isRecyclable*/);
  }
  return result;
}
```
cache 在 GlidBuild 中已经定义了，就是 LruResourceCache 这个。它  extends LruCache<Key, Resource<?>>  并且 implements MemoryCache

###  两者的区别与联系
1. ActiveResources 缓存是基于弱引用缓存，会在内存不足的时候清理掉。而 MemoryCache 是基于 LruCache 的强引用缓存，因此不会因为内存不足被清理掉。 MemoryCache 只有当缓存达到数据后，才会将最近最少使用的缓存清理掉
2. 两个缓存都是基于 Hash 表，只是 LruCache 除了了具有 Hash 表的数据结构之外，还维护了一个链表。而弱引用类型的缓存的 key 与 LruCache 一致，但是值却是弱引用类型的
3. 除了内存不够的时候 ActiveResources 会被释放，还在在 Engine 的资源被释放的时候清理掉
4. 基于弱引用的缓存一致都存在，用户无法禁用，但是用户可以关闭 LruCache 缓存
5. 本质上基于弱引用的缓存与基于 LruCache 缓存针对不同的应用场景，弱引用的缓存算是缓存的一种类型，只是这种缓存受内存的影响比 LruCache 更大



## 磁盘缓存读取
接口为 DiskCache ， Glide 使用 DiskLruCacheWrapper 作为默认的磁盘缓存， DiskLruCacheWrapper 是一个使用 LRU 算法的固定大小的磁盘缓存。默认磁盘大小为 250MB ，位置是在应用的缓存文件夹下中，即 /data/data/<application package>/cache/image_manager_disk_cache 目录下面

磁盘缓存可以根据实际情况，设置不同的策略。设置方式是调用 diskCacheStrategy() 方法，如下，设置了禁用磁盘缓存

```java
GlideApp.with(fragment)
  .load(url)
  //禁用磁盘缓存
  .diskCacheStrategy(DiskCacheStrategy.NONE)
  .into(view);
```

### 磁盘缓存策略
* DiskCacheStrategy.ALL 表示既缓存原始图片，也缓存转换过后的图片。对于远程图片，缓存 DATA 和 RESOURCE 。对于本地图片，只缓存 RESOURCE 。
* DiskCacheStrategy.AUTOMATIC 它会尝试对本地和远程图片使用最佳的策略。当你加载远程数据（比如，从 URL 下载）时， AUTOMATIC 策略仅会存储未被你的加载过程修改过(比如，变换，裁剪–译者注)的原始数据（DATA），因为下载远程数据相比调整磁盘上已经存在的数据要昂贵得多。对于本地数据， AUTOMATIC 策略则会仅存储变换过的缩略图（RESOURCE），因为即使你需要再次生成另一个尺寸或类型的图片，取回原始数据也很容易。
* DiskCacheStrategy.RESOURCE  表示只缓存转换过后的图片。（也就是经过 decode ，转化裁剪的图片）
* DiskCacheStrategy.DATA  表示只缓存未被处理的文件。我的理解就是我们获得的 stream 。它是不会被展示出来的，需要经过 decode ，对图片进行压缩和转换等操作，得到最终的图片才能被展示。
* DiskCacheStrategy.NONE   表示不缓存任何内容


从前面的Engine.load()方法中，我们知道，从磁盘缓存中读取关键代码是 engineJob.start(decodeJob)，那么我们就从这里开始分析

```java
//EngineJob.java
public synchronized void start(DecodeJob<R> decodeJob) {
   this.decodeJob = decodeJob;
   GlideExecutor executor = decodeJob.willDecodeFromCache() ? diskCacheExecutor: getActiveSourceExecutor();
   executor.execute(decodeJob);
 }
```
关键代码也就是 DecodeJob 的 willDecodeFromCache()
```java
//DecodeJob.java
boolean willDecodeFromCache() {
   Stage firstStage = getNextStage(Stage.INITIALIZE);
   return firstStage == Stage.RESOURCE_CACHE || firstStage == Stage.DATA_CACHE;
 }
 ```
### DecodeJob # getNextStage()
```java
private Stage getNextStage(Stage current) {
  switch (current) {
    case INITIALIZE:
    //如果我们指定磁盘缓存策略为 ALL 或 RESOURCE 或由 AUTOMATIC 对远程图片使用磁盘缓存时，此时返回 true ,返回Stage.RESOURCE_CACHE。
    //如果为 false ，递归调用，判断是否为diskCacheStrategy.decodeCachedData()，也就是指定磁盘缓存策略为 ALL 或DATA
      return diskCacheStrategy.decodeCachedResource()? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
    case RESOURCE_CACHE:
    //是否解码缓存的原始数据，就是指缓存的未做过变换的数据
      return diskCacheStrategy.decodeCachedData() ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
    case DATA_CACHE:
      //判断 onlyRetrieveFromCache ，这个值是初始化 DecodeJob 中传进来的，它代表是否仅从缓存加载图片，通过onlyRetrieveFromCache(true)制定,默认为 false
      //如果为 true ，它意味着要从内存或磁盘读取，如果内存或磁盘不存在该资源，则加载直接失败。一般情况下我们不会制定，为 false ，也就是会返回 Stage.SOURCE。代表不使用磁盘缓存，也就是之前文章分析的，直接从服务器下载
      return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
    case SOURCE:
    case FINISHED:
      return Stage.FINISHED;
    default:
      throw new IllegalArgumentException("Unrecognized stage: " + current);
  }
}
```
当我们设置的缓存测量是 ALL 或 RESOURCE 或 AUTOMATIC 或者 DATA 的时候，返回 Stage.RESOURCE_CACHE 或者 Stage.DATA_CACHE 这种的时候， willDecodeFromCache() 就是 true ，就由 diskCacheExecutor 来执行， diskCacheExecutor 在 GlideBuilde 的是时候也初始化了，最终经过层层调用，还是执行到了 DecodeJob 的 run() 方法，又到了 runWrapped() 中

```java
//DecodeJob.java
private void runWrapped() {
  switch (runReason) {
    case INITIALIZE:
      // 初始化 获取下一个阶段状态
      stage = getNextStage(Stage.INITIALIZE);
      currentGenerator = getNextGenerator();
      // 运行
      runGenerators();
      break;
    case SWITCH_TO_SOURCE_SERVICE:
      runGenerators();
      break;
    case DECODE_DATA:
      decodeFromRetrievedData();
      break;
    default:
      throw new IllegalStateException("Unrecognized run reason: " + runReason);
  }
}
```
又执行到了 getNextStage()
```java
private Stage getNextStage(Stage current) {
  switch (current) {
    case INITIALIZE:
    //是否解码缓存的转换图片，就是只做过变换之后的缓存数据
    return diskCacheStrategy.decodeCachedResource()? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
    ...
  }
}      
```
current 是 INITIALIZE ，所以通过 diskCacheStrategy.decodeCachedResource() 判断返回 Stage 类型。

这也验证了上面的流程，先从 ResourceCache 中查找缓存，如果这里面没有的话，就从 DataCache 中查找缓存。

### 从 ResourceCache 中读取缓存

根据上面的缓存测量可知，如果我们指定磁盘缓存策略为 ALL 或 RESOURCE ， AUTOMATIC 此时为 true ,返回Stage.RESOURCE_CACHE。
```java
private DataFetcherGenerator getNextGenerator() {
  switch (stage) {
    case RESOURCE_CACHE:
      return new ResourceCacheGenerator(decodeHelper, this);
    case DATA_CACHE:
      return new DataCacheGenerator(decodeHelper, this);
    case SOURCE:
      return new SourceGenerator(decodeHelper, this);
    case FINISHED:
      return null;
    default:
      throw new IllegalStateException("Unrecognized stage: " + stage);
  }
}
```
* ResourceCacheGenerator     加载磁盘上调整后大小资源，解码后的数据
* DataCacheGenerator  加载磁盘上原大小资源，原始数据
* SourceGenerator  网络获取资源，

getNextGenerator() 中返回的就是 ResourceCacheGenerator ，那么 currentGenerator 就是 ResourceCacheGenerator 对象。 接着继续执行 runGenerators()
```java
private void runGenerators() {
  currentThread = Thread.currentThread();
  startFetchTime = LogTime.getLogTime();
  boolean isStarted = false;
  while (!isCancelled && currentGenerator != null && !(isStarted = )) {
    stage = getNextStage(stage);
    currentGenerator = getNextGenerator();

    if (stage == Stage.SOURCE) {
      reschedule();
      return;
    }
  }
  if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
    notifyFailed();
  }
}
```
currentGenerator.startNext() 就执行到了 ResourceCacheGenerator 中 的 startNext() ，它用来从缓存中得到变换之后数据
```java
//ResourceCacheGenerator.java
public boolean startNext() {
      ...
      // 1 构建获取缓存信息的键
      currentKey = new ResourceCacheKey(...);
      // 2 从缓存中获取缓存信息
      cacheFile = helper.getDiskCache().get(currentKey);
      if (cacheFile != null) {
        sourceKey = sourceId;
        modelLoaders = helper.getModelLoaders(cacheFile);
        modelLoaderIndex = 0;
      }
    }

    loadData = null;
    boolean started = false;
    // 查找ModelLoader
    while (!started && hasNextModelLoader()) {
      ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
      //根据 key 读取缓存文件 cacheFile ，传入 File ，
      loadData = modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
        started = true;
        // 通过 FileLoader 继续加载数据
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }
    return started;
  }
```
helper.getDiskCache() 获取的是 DiskCache 对象，一路追踪这个对象，就会找到了 DiskLruCacheWrapper ，他内部保证了LruCache.最终从磁盘加载数据，是使用 DiskLruCache 来实现的。对于最终使用 DiskLruCache 获取数据的逻辑我们不进行说明了，它的逻辑并不复杂，都是单纯的文件读写，只是设计了一套缓存的规则。
![添加图片](../../../../images/glide_fetcher.png)
fetcher 有很多实现类，根据你传递过来的 modle ，进行匹配。
因为 modelLoader 泛型参数是 File 类型，所以就执行到了 FileFetcher ,这里就把文件 转换成了Resource
```java
//FileFetcher.java
public void loadData(@NonNull Priority priority, @NonNull DataCallback<? super Data> callback) {
  try {
    data = opener.open(file);
  } catch (FileNotFoundException e) {
    callback.onLoadFailed(e);
    return;
  }
  //返回结果,
  callback.onDataReady(data);
}
```
callback 就是对应的 Generator ,这里指的就是 ResourceCacheGenerator ,然后经过层层回调，最终执行 DecodeJob 的 onDataFetcherReady() 中，
```java
//DecodeJob.java
public void onDataFetcherReady(Key sourceKey, Object data , DataFetcher<?> fetcher, DataSource dataSource , Key attemptedKey) {
  ...
  if (Thread.currentThread() != currentThread) {
    runReason = RunReason.DECODE_DATA;
    callback.reschedule(this);
  } else {
    GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
    try {
      decodeFromRetrievedData();
    } finally {
      GlideTrace.endSection();
    }
  }
}

```
中间涉及到线程切换的，就不多说了。这里面又执行到了 decodeFromRetrievedData() 方法，我们就直接看这个方法吧。

```java
private void decodeFromRetrievedData() {
  Resource<R> resource = null;
  try {
    //将数据解码成 Resource 对象
    resource = decodeFromData(currentFetcher, currentData , currentDataSource);
  } catch (GlideException e) {
    e.setLoggingDetails(currentAttemptingKey, currentDataSource);
    throwables.add(e);
  }
  if (resource != null) {
    //回调 UI 线程显示出来。
    notifyEncodeAndRelease(resource, currentDataSource);
  } else {
    //如果 ResourceCache 中没有，则进行递归调用，生成下一个 Generator ,继续查找
    runGenerators();
  }
}
```
通过 decodeFromData 方法将数据解码成 Resource 对象后返回即可。如果 Resource 对象不为 null ，则通过 notifyEncodeAndRelease() 回调 UI 线程显示出来。
否则 进行递归调用，执行 runGenerators() ,生成下一个 Generator ,继续查找,这个时候就到了 DataCacheGenerator 中了，这个就和 diskCacheStrategy.decodeCachedResource() 就返回 false 类似了

### 从 DataCache 中读取缓存
```java
private Stage getNextStage(Stage current) {
  switch (current) {
    case INITIALIZE:
      return diskCacheStrategy.decodeCachedResource()? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
    case RESOURCE_CACHE:
      return diskCacheStrategy.decodeCachedData() ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
      ...
  }
}
```
current 就变成了 RESOURCE_CACHE ，那么通过 diskCacheStrategy.decodeCachedData()进行判断。

```java
public static final DiskCacheStrategy ALL = new DiskCacheStrategy() {
     public boolean decodeCachedData() {
         return true;
     }
 };
 public static final DiskCacheStrategy DATA = new DiskCacheStrategy() {
       public boolean decodeCachedData() {
           return true;
       }
   };
  public static final DiskCacheStrategy AUTOMATIC = new DiskCacheStrategy() {
     public boolean decodeCachedData() {
         return true;
     }
  };
```
如果设置了 ALL ， DATA 或 AUTOMATIC ， diskCacheStrategy.decodeCachedData()就返回 true ， getNextGenerator() 返回的就是 DataCacheGenerator 。

接下来我们看看 DataCacheGenerator.startNext(),
```java
public boolean startNext() {
  while (modelLoaders == null || !hasNextModelLoader()) {
    sourceIdIndex++;
    if (sourceIdIndex >= cacheKeys.size()) {
      return false;
    }

    Key sourceId = cacheKeys.get(sourceIdIndex);
    Key originalKey = new DataCacheKey(sourceId, helper.getSignature());
    cacheFile = helper.getDiskCache().get(originalKey);
    if (cacheFile != null) {
      this.sourceKey = sourceId;
      modelLoaders = helper.getModelLoaders(cacheFile);
      modelLoaderIndex = 0;
    }
  }

  loadData = null;
  boolean started = false;
  while (!started && hasNextModelLoader()) {
    ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
    loadData = modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(),helper.getOptions());
    if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
      started = true;
      loadData.fetcher.loadData(helper.getPriority(), this);
    }
  }
  return started;
}
```
里面的内容和 ResourceCacheGenerator 中的差不多，也是根据 key ,读取缓存文件。然后返回

如果返回的数据为 null ，或者  diskCacheStrategy.decodeCachedData()返回 false ，就继续递归调用，getNextStage(Stage.DATA_CACHE)

### 禁用磁盘缓存

```java
private Stage getNextStage(Stage current) {
  ...
    case DATA_CACHE:
      return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
    case SOURCE:
    case FINISHED:
      return Stage.FINISHED;
    ...
  }
}
```
 onlyRetrieveFromCache ，这个值是初始化 DecodeJob 中传进来的，它代表是否仅从缓存加载图片，通过onlyRetrieveFromCache(true)制定,默认为 false ，
如果为 true ，它意味着要从内存或磁盘读取，如果内存或磁盘不存在该资源，则加载直接失败。一般情况下我们不会制定.
如果为 false ，也就是会返回 Stage.SOURCE。代表不使用磁盘缓存，那么得到的 Generator 就是 SourceGenerator ，
那么我们就 SourceGenerator.startNext()

```java
public boolean startNext() {
   if (dataToCache != null) {
     Object data = dataToCache;
     dataToCache = null;
     cacheData(data);
   }
   //正在执行 startNext ，则直接返回
   if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
     return true;
   }
   sourceCacheGenerator = null;

   loadData = null;
   boolean started = false;
   while (!started && hasNextModelLoader()) {
     loadData = helper.getLoadData().get(loadDataListIndex++);
     if (loadData != null && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())|| helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
       started = true;
       loadData.fetcher.loadData(helper.getPriority(), this);
     }
   }
   return started;
 }
```
刚开始 dataToCache 为空，所以不走这一步，然后就是通过loadData.fetcher.loadData()加载完毕会返调用 SourceGenerator.onDataReady(result)将结果返回

已经得到数据了，可是什么时候写入磁盘缓存呢？
## 磁盘缓存写入

```java
//SourceGenerator类
@Override
  public void onDataReady(Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      dataToCache = data;
      cb.reschedule();
    } else {
      cb.onDataFetcherReady(loadData.sourceKey, data , loadData.fetcher,loadData.fetcher.getDataSource(), originalKey);
    }
  }
```
此时如果我们的磁盘缓存策略没有禁止，那么 dataToCache = data;同时执行 cb.reschedule();也就是 DecodeJob.reschedule()-> DecodeJob.run()-> DecodeJob.runGenerators()->currentGenerator.startNext(),又执行到了 SourceGenerator.startNext()中。

```java
//SourceGenerator.java
 public boolean startNext() {
   // 这个时候 dataToCache 不为 null ，所以执行 cacheData() ,进行缓存
   if (dataToCache != null) {
     Object data = dataToCache;
     dataToCache = null;
     cacheData(data);
   }
   if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
     return true;
   }
   ...
 }

```
此时 dataToCache 已经不为 null 了。也就可以执行cacheData(data)就是这里了，我们在这里写入数据。
```java
private void cacheData(Object dataToCache) {
  long startTime = LogTime.getLogTime();
  try {
    // 根据不同的数据获取注册的不同Encoder
    Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
    DataCacheWriter<Object> writer = new DataCacheWriter<>(encoder, dataToCache , helper.getOptions());
    originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
    //得到 DiskCache 得实现，并存入磁盘。
    helper.getDiskCache().put(originalKey, writer);
    }
  } finally {
    loadData.fetcher.cleanup();
  }
  sourceCacheGenerator = new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper , this);
}
```
helper.getDiskCache().put(originalKey, writer)，这行代码，就把 Resource 存到磁盘中了


内存之间互相写入我们刚才说了，就是在 ActiveResources 中根据 key 查找不到 Resource 的时候，写入到 MemoryCache 中，从 MemoryCache 中根据 key 查到对应的 Resource 的时候，把该 Resource 再写入到 ActiveResources 中。
可是硬盘缓存的什么时候写入到内存缓存中呢？

## 内存缓存写入
不管是 DataCacheGenerator , ResourceCacheGenerator 还是 SourceGenerator ,都会执行到loadData.fetcher.loadData(helper.getPriority(), this)得到的 Resource 经过层层调用，就会执行到 DecodeJob 的 onDataFetcherReady()-> decodeFromRetrievedData()-> notifyEncodeAndRelease()->notifyComplete()-> EngineJob 的 onResourceReady() ,
```java
//EngineJob.java
public void onResourceReady(Resource<R> resource, DataSource dataSource) {
  synchronized (this) {
    this.resource = resource;
    this.dataSource = dataSource;
  }
  notifyCallbacksOfResult();
}

void notifyCallbacksOfResult() {
    ...
    //listener 就是 Engine
    listener.onEngineJobComplete(this, localKey , localResource);
    for (final ResourceCallbackAndExecutor entry : copy) {
      // 在这里，会去加载数据，显示到 UI 上
      entry.executor.execute(new CallResourceReady(entry.cb));
    }
    decrementPendingCallbacks();
  }
```
关键代码 listener.onEngineJobComplete(this, localKey , localResource);

### 写入 ActiveResources
```java
//Engine.java
public synchronized void onEngineJobComplete( EngineJob<?> engineJob, Key key , EngineResource<?> resource) {
  if (resource != null) {
    //将 resource 添加监听
    resource.setResourceListener(key, this);
    //回调过来的 EngineResource 被 put 到了 activeResources 当中，在这里写入到了内存缓存的弱引用缓存。
    //写入到弱引用缓存的原因是这个资源是属于正在被加载展示的资源，也就是正在被使用的资源。
    if (resource.isCacheable()) {
      activeResources.activate(key, resource);
    }
  }
  jobs.removeIfCurrent(key, engineJob);
}
```
如果设置了内存缓存，则就会把 Resource 写入 ActiveResources 中，因为该 Resource 属于被加载的资源，

```java
//ActiveResources.java
synchronized void activate(Key key, EngineResource<?> resource) {
  // 根据 缓存 key , resource 等创建一个 ResourceWeakReference ,
  ResourceWeakReference toPut =
      new ResourceWeakReference(key, resource , resourceReferenceQueue , isActiveResourceRetentionAllowed);
    // 把这个放入到 activeEngineResources map 集合，返回的是之前存放的
  ResourceWeakReference removed = activeEngineResources.put(key, toPut);
  if (removed != null) {
    //如果这个缓存 key 之前存放的有 ResourceWeakReference ,则重置这个 ResourceWeakReference ，
    removed.reset();
  }
}
```

### 写入 MemoryCache
什么时候写入到 MemoryCache 中，需要根据 EngineResource 中的 acquired 的值来判断。   
当 acquired 变量大于 0 的时候，说明图片正在使用中，也就应该放到 activeResources 弱引用缓存当中,如果 acquired 变量等于 0 了，说明图片已经不在使用中了，此时放到 LruCache 来进行缓存。    
acquired 的增加和减少通过 EngineResource 的 acquire() 和 release() 方法。

```java
//EngineResource.java
synchronized void acquire() {
  if (isRecycled) {
    throw new IllegalStateException("Cannot acquire a recycled resource");
  }
  ++acquired;
}

void release() {
  synchronized (listener) {
    synchronized (this) {
      if (acquired <= 0) {
        throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
      }
      if (--acquired == 0) {
        listener.onResourceReleased(key, this);
      }
    }
  }
}
```
当 acquired==0的时候，执行了 onResourceReleased() ,这个 listener 就是 Engine ，

```java
//Engine.java
public synchronized void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
  //当不在使用中时，此时就可以从 activeResources 移除，同时就可以添加到 Lrucache 中了。此处为写入内存缓存的 LruCache 地方。
   activeResources.deactivate(cacheKey);
   if (resource.isCacheable()) {
     cache.put(cacheKey, resource);
   } else {
     resourceRecycler.recycle(resource);
   }
 }
```
cache.put(cacheKey, resource) 这个就把该 Resource 存放到了 MemoryCache 中了。

## 取得数据显示到View

[源码分析 - Glide4 之 into()](../../../../2019/11/01/glide4-source_analysis_2)
---
搬运地址：    

[Glide4.0源码全解析（一）， GlideAPP 和.with()方法背后的故事](https://blog.csdn.net/github_33304260/article/details/77869221)    

[Glide4.0源码全解析（二）， load() 背后的故事](https://blog.csdn.net/github_33304260/article/details/77992717)   

[Glide源码解析(三)](https://blog.csdn.net/pmx_121212/article/details/79085947)   

[Android图片加载库 Glide 知其然知其所以然 开篇](https://juejin.im/post/5cbea88cf265da03555c7f58)    

[Android 图片加载库 Glide 知其然知其所以然之加载](https://juejin.im/post/5cd51235e51d456e831f69f6)    

[Glide架构设计艺术](https://www.jianshu.com/p/5c8ce241199e)   

[Glide源码导读](https://www.cnblogs.com/angeldevil/p/5841979.html)

[【一篇就懂系列】Glide源码分析之缓存处理](https://www.jianshu.com/p/3fc0bbdb3891)

[Glide 系列-3：Glide 缓存的实现原理（4.8.0）](https://www.jianshu.com/p/8f2631094804)
