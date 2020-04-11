---
layout: post
title: 源码分析 - Glide4 之 into()
category: 源码分析
tags: glide4
---
* content
{:toc}

承接上文 [ 源码分析 - Glide4 之 with() 和 load() ](../../../../2019/11/01/glide4-source_analysis/)

```java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);
    BaseRequestOptions<?> requestOptions = this;
    //在没有调用 transform 方法，并且变换可用的情况下，如果设置了 scaletype ，就会根据你的设置对图片进行处理
    if (!requestOptions.isTransformationSet()&& requestOptions.isTransformationAllowed()&& view.getScaleType() != null) {
      switch (view.getScaleType()) {
        //用 clone 方法来保证不会影响之后的视图显示效果
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }

    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
 requestOptions ,
        Executors.mainThreadExecutor());
  }
```
如图，当我们调用into(target)的时候，我们可知
1. 必须在主线程中
2. targetListener 是 null ,是用来加载图片的监听器
3. 根据 View 的 getScaleType 来判断是设置requestOptions
4. 根据 View 和 transcodeClass 来构建一个ViewTarget

先看看 buildImageViewTarget() 是干嘛的吧

### GlideContext buildImageViewTarget()
```java
GlideContext
public <X> ViewTarget<ImageView, X> buildImageViewTarget(
      @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
  }
ImageViewTargetFactory
  public <Z> ViewTarget<ImageView, Z> buildTarget(@NonNull ImageView view,
       @NonNull Class<Z> clazz) {
     if (Bitmap.class.equals(clazz)) {
       return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
     } else if (Drawable.class.isAssignableFrom(clazz)) {
       return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
     } else {
       throw new IllegalArgumentException(
           "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
     }
   }
```
根据 clazz 构建不同的 ViewTarget ，这个 clazz 是在创建 RequestBuilder() 的时候传递过来的，如果加载图片的时候使用 asBtimap() ,那么就会创建 BitmapImageViewTarget ,如果调用 asDrawable() ,或者 asGif() ,那么就创建一个 DrawableImageViewTarget ，否则抛出异常。

接下来看真正的into()

四个参数我们也大概能猜出来到底干嘛的
target 要加载的载体，比如ImageView
targetListener 加载时候的监听
options 加载时候的配置，比如是否圆角，圆形图片等
callbackExecutor 因为加载图片可能耗时，所以需要在子线程中进行，那么就需要线程池进行管理


```java
private <Y extends Target<TranscodeType>> Y into(@NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    //若 load 方法没有调用会抛出异常
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    Request request = buildRequest(target, targetListener , options , callbackExecutor);

    Request previous = target.getRequest();
    //若上一个请求和新建请求一致并且使用了内存缓存或者上一个请求没有完成则将新建请求回收，并保证上一个请求的运行
    if (request.isEquivalentTo(previous)&&!isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      request.recycle();
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        previous.begin();
      }
      return target;
    }
    //清除 target 之前的请求并设置新的请求
    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);
    return target;
  }

```
通过 buildRequest() 创建一个 Request ，用于发起一个加载请求，通过后面的代码也可以知道，真正返回的是 SingleRequest

### buildRequest()

```java
private Request buildRequest(
      Target<TranscodeType> target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {
    return buildRequestRecursive(
 target ,
 targetListener ,
        /*parentCoordinator=*/ null,
 transitionOptions ,
        requestOptions.getPriority(),
        requestOptions.getOverrideWidth(),
        requestOptions.getOverrideHeight(),
 requestOptions ,
        callbackExecutor);
  }

  private Request buildRequestRecursive(...) {

    // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
    ErrorRequestCoordinator errorRequestCoordinator = null;
    if (errorBuilder != null) {
      errorRequestCoordinator = new ErrorRequestCoordinator(parentCoordinator);
      parentCoordinator = errorRequestCoordinator;
    }
    Request mainRequest = buildThumbnailRequestRecursive(...);
    if (errorRequestCoordinator == null) {
      return mainRequest;
    }
    int errorOverrideWidth = errorBuilder.getOverrideWidth();
    int errorOverrideHeight = errorBuilder.getOverrideHeight();
    if (Util.isValidDimensions(overrideWidth, overrideHeight) && !errorBuilder.isValidOverride()) {
      errorOverrideWidth = requestOptions.getOverrideWidth();
      errorOverrideHeight = requestOptions.getOverrideHeight();
    }

    Request errorRequest = errorBuilder.buildRequestRecursive(...);
    errorRequestCoordinator.setRequests(mainRequest, errorRequest);
    return errorRequestCoordinator;
  }

```
* 如果我们调用了 error() 方法， errorBuilder 就不会为 null ，
* errorRequest 则是处理错误显示的请求，
* mainRequest 则是包含了缩略图和目标图的请求，

我们先看看

```java
private Request buildThumbnailRequestRecursive(...) {
    if (thumbnailBuilder != null) {
      // Recursive case: contains a potentially recursive thumbnail request builder.
      if (isThumbnailBuilt) {
        throw new IllegalStateException("You cannot use a request as both the main request and a "
            + "thumbnail, consider using clone() on the request(s) passed to thumbnail()");
      }

      TransitionOptions<?, ? super TranscodeType> thumbTransitionOptions =
          thumbnailBuilder.transitionOptions;

      // Apply our transition by default to thumbnail requests but avoid overriding custom options
      // that may have been applied on the thumbnail request explicitly.
      if (thumbnailBuilder.isDefaultTransitionOptionsSet) {
        thumbTransitionOptions = transitionOptions;
      }

      Priority thumbPriority = thumbnailBuilder.isPrioritySet()
          ? thumbnailBuilder.getPriority() : getThumbnailPriority(priority);

      int thumbOverrideWidth = thumbnailBuilder.getOverrideWidth();
      int thumbOverrideHeight = thumbnailBuilder.getOverrideHeight();
      if (Util.isValidDimensions(overrideWidth, overrideHeight)&& !thumbnailBuilder.isValidOverride()) {
        thumbOverrideWidth = requestOptions.getOverrideWidth();
        thumbOverrideHeight = requestOptions.getOverrideHeight();
      }

      ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
      Request fullRequest =obtainRequest(...);
      isThumbnailBuilt = true;
      // Recursively generate thumbnail requests.
      Request thumbRequest =thumbnailBuilder.buildRequestRecursive(...);
      isThumbnailBuilt = false;
      coordinator.setRequests(fullRequest, thumbRequest);
      return coordinator;
    } else if (thumbSizeMultiplier != null) {
       //若调用了thumbnail(float sizeMultiplier)走此处
      // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
      ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
      Request fullRequest =obtainRequest(...);
      BaseRequestOptions<?> thumbnailOptions =requestOptions.clone().sizeMultiplier(thumbSizeMultiplier);
      Request thumbnailRequest =obtainRequest(...);
      coordinator.setRequests(fullRequest, thumbnailRequest);
      return coordinator;
    } else {
      // Base case: no thumbnail.
      return obtainRequest(...);
    }
  }
```
我们发现大部分都会执行到 obtainRequest() ,这个 obtainRequest 可以获取到一个 Request 对象。

```java
private Request obtainRequest(...) {
    return SingleRequest.obtain(...);
  }

public static <R> SingleRequest<R> obtain(...) {
  @SuppressWarnings("unchecked") SingleRequest<R> request =(SingleRequest<R>) request.acquire();
  if (request == null) {
    request = new SingleRequest<>();
  }
  request.init(...);
  return request;
}
```
先从对象池中得到对象，如果不存在，则创建，然后 init ,使用的享元模式，处于对内存的节省，先从对象池中取，有则共享，减少 new 对象的成本，
创建完成 Request 后，然后就是  requestManager # track()

### RequestManager # track()

```java
requestManager.clear(target);
target.setRequest(request);
requestManager.track(target, request);

synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
  //保存 target 的记录，方便清除请求
   targetTracker.track(target);
   requestTracker.runRequest(request);
 }

public void runRequest(@NonNull Request request) {
  requests.add(request);
  //判断 Glide 当前是不是处理暂停状态
  if (!isPaused) {
    //调用 Request 的 begin() 方法来执行Request
    request.begin();
  } else {
    //Request添加到待执行队列里面，等暂停状态解除了之后再执行
    request.clear();
    pendingRequests.add(request);
  }
}
```
注释都写上去了，就不多解释了。因为我们创建的是 SingleRequest ,所以进入到了 SingleRequest 的 begin 中,执行 request.
### SingleRequest # begin()
```java
public synchronized void begin() {
    assertNotCallingCallbacks();
    stateVerifier.throwIfRecycled();
    startTime = LogTime.getLogTime();
    if (model == null) {
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        width = overrideWidth;
        height = overrideHeight;
      }
      int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
      onLoadFailed(new GlideException("Received null model"), logLevel);
      return;
    }

    if (status == Status.RUNNING) {
      throw new IllegalArgumentException("Cannot restart a running request");
    }

    if (status == Status.COMPLETE) {
      onResourceReady(resource, DataSource.MEMORY_CACHE);
      return;
    }
    status = Status.WAITING_FOR_SIZE;
    //是否使用了 override() API 为图片指定了一个固定的宽高
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      onSizeReady(overrideWidth, overrideHeight);
    } else {
      //没有的话，会根据 ImageView 的 layout_width 和 layout_height 值做一系列的计算，来算出图片应该的宽高。在计算完之后，它也会调用 onSizeReady() 方法
      target.getSize(this);
    }

    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE) && canNotifyStatusChanged()) {
      target.onLoadStarted(getPlaceholderDrawable());
    }
  }

```
这个主要是根据 status 进行判断
* 如果 COMPLETE 已完成，则直接回调 onResourceReady()
* 如果是 WAITING_FOR_SIZE ，则会执行 onSizeReady() ，最后通过 Engine 的 load() 方法，然后同样执行 onResourceReady() ,


* 如果 model 为 null ，则调用 onLoadFailed()

### onLoadFailed()

```java
private synchronized void onLoadFailed(GlideException e, int maxLogLevel) {
    stateVerifier.throwIfRecycled();
    e.setOrigin(requestOrigin);
    int logLevel = glideContext.getLogLevel();

    loadStatus = null;
    status = Status.FAILED;

    isCallingCallbacks = true;
    try {
      boolean anyListenerHandledUpdatingTarget = false;
      if (requestListeners != null) {
        for (RequestListener<R> listener : requestListeners) {
          anyListenerHandledUpdatingTarget |=listener.onLoadFailed(e, model , target , isFirstReadyResource());
        }
      }
      anyListenerHandledUpdatingTarget |= targetListener != null && targetListener.onLoadFailed(e, model , target , isFirstReadyResource());

      if (!anyListenerHandledUpdatingTarget) {
        setErrorPlaceholder();
      }
    } finally {
      isCallingCallbacks = false;
    }

    notifyLoadFailed();
  }

  private synchronized void setErrorPlaceholder() {
    if (!canNotifyStatusChanged()) {
      return;
    }

    Drawable error = null;
    if (model == null) {
      error = getFallbackDrawable();
    }
    // Either the model isn't null, or there was no fallback drawable set.
    if (error == null) {
      error = getErrorDrawable();
    }
    // The model isn't null, no fallback drawable was set or no error drawable was set.
    if (error == null) {
      error = getPlaceholderDrawable();
    }
    target.onLoadFailed(error);
  }
```
里面调用了 setErrorPlaceholder() ，会先去获取一个 fallback 图，若获取不到 fallback 图则获取 error 的占位图，如果获取不到的话会再去获取一个 loading 占位图。


target.onLoadStarted()方法，这个方法传入了一个 loading 占位图，在也就说，在图片请求开始之前，会先使用这张占位图代替最终的图片显示。

### onSizeReady()
```java
public synchronized void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    if (status != Status.WAITING_FOR_SIZE) {
      return;
    }
    status = Status.RUNNING;
    //获取的是缩略图的缩小倍数
    float sizeMultiplier = requestOptions.getSizeMultiplier();
    //得到显示的宽高，
    this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
    this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

    loadStatus =  engine.load(...);

    if (status != Status.RUNNING) {
      loadStatus = null;
    }
  }
```

```java
public synchronized <R> LoadStatus load(...) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    EngineKey key = keyFactory.buildKey(model, signature , width , height , transformations ,
 resourceClass , transcodeClass , options);
    //从活跃资源中获取
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      return null;
    }
    //从缓存中获取
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      return null;
    }
    //看 EngineJob 中是否已经包含当前任务
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      return new LoadStatus(cb, current);
    }

    EngineJob<R> engineJob =engineJobFactory.build(...);

    DecodeJob<R> decodeJob =decodeJobFactory.build(...);

    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);
    return new LoadStatus(cb, engineJob);
  }

```

activeResources 是一个以弱引用为值的 map ,他是用于存储使用中的资源，

MemoryCache 是一个 LruResourceCache ，先从 activeResources 中获取，获取不到则到 MemoryCache 中获取，获取到了则将资源缓存到 activeResources ，获取到后均会进行引用计数，获取不到则看 EngineJob 中是否已经包含当前任务，如果有则替换回调，没有则新建任务，


```Java
public synchronized void start(DecodeJob<R> decodeJob) {
  this.decodeJob = decodeJob;
  GlideExecutor executor = decodeJob.willDecodeFromCache()? diskCacheExecutor : getActiveSourceExecutor();
  executor.execute(decodeJob);
}
```
主要根据是否有缓存来决定从磁盘上加载资源或是从网络上加载资源，接下来我们看 decodeJob 的 run 方法：
```Java
public void run() {
  ...
  runWrapped();
  ...
}

private void runWrapped() {
  switch (runReason) {
    case INITIALIZE:
      stage = getNextStage(Stage.INITIALIZE);
      currentGenerator = getNextGenerator();
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
INITIALIZE 和 SWITCH_TO_SOURCE_SERVICE 最终都执行到了 runGenerators() ,如果 DECODE_DATA ，那么执行到 decodeFromRetrievedData() 中
#### runGenerators()
```java
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled && currentGenerator != null && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
    // We've run out of stages and generators, give up.
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }
  }
```
重点看 currentGenerator.startNext()，而 currentGenerator 有三种类型，

* SourceGenerator    加载磁盘上原大小资源
* DataCacheGenerator  加载磁盘上调整后大小资源
* ResourceCacheGenerator  网络获取资源

```java
// SourceGenerator.java
@Override
public boolean startNext() {
   //当 dataToCache 不为空的时候，将 dataToCache 放入内存中
   if (dataToCache != null) {
     Object data = dataToCache;
     dataToCache = null;
     cacheData(data);
   }

   if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
     return true;
   }
   sourceCacheGenerator = null;

   loadData = null;
   boolean started = false;
   //获取能够用的 LoadData 对象，然后调用它的LoadData（）
   while (!started && hasNextModelLoader()) {
     loadData = helper.getLoadData().get(loadDataListIndex++);
     if (loadData != null && (
     helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource()) || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
       started = true;
       loadData.fetcher.loadData(helper.getPriority(), this);
     }
   }
   return started;
}

```
SourceGenerator 实现了 DataCallBack 接口

`class SourceGenerator implements DataFetcherGenerator,
    DataFetcher.DataCallback<Object>,
    DataFetcherGenerator.FetcherReadyCallback `

```java
@Override
  public void onDataReady(Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      dataToCache = data;
      // We might be being called back on someone else's thread. Before doing anything, we should
      // reschedule to get back onto Glide's thread.
      cb.reschedule();
    } else {
      cb.onDataFetcherReady(loadData.sourceKey, data , loadData.fetcher,
          loadData.fetcher.getDataSource(), originalKey);
    }
  }

```
获取到数据可以被缓存的情况下就会将 data 赋值给 dataToCache ，如果不能被赋就回调传入的 FetcherReadyCallback 对象的 onReadyCallback 方法。

```Java
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
      loadData =modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(),helper.getOptions());
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
        started = true;
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }
    return started;
  }

```
流程基本一致，
1. 通过helper.getDiskCache().get(originalKey) 得到缓存文件
2. 如果不为 null ，则通过 DecodeHelper 来获得 List 中的 ModelLoader ，即helper.getModelLoaders(cacheFile)
3. modelLoader.buildLoadData获取 LoadData ，最后通过loadData.fetcher.loadData来加载资源

loadData 有很多实现，在加载好资源后最终都会回调 callback.onDataReady(result);这个方法会回调 onDataFetcherReady 方法，
```Java
public void onDataFetcherReady(Key sourceKey, Object data , DataFetcher<?> fetcher,
 DataSource dataSource , Key attemptedKey) {
   this.currentSourceKey = sourceKey;
   this.currentData = data;
   this.currentFetcher = fetcher;
   this.currentDataSource = dataSource;
   this.currentAttemptingKey = attemptedKey;
   //如果 DecodeJob 的线程不是当前线程，则重新加载，
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
#### decodeFromRetrievedData()
```Java
private void decodeFromRetrievedData() {
    Resource<R> resource = null;
    try {
      resource = decodeFromData(currentFetcher, currentData , currentDataSource);
    } catch (GlideException e) {
      e.setLoggingDetails(currentAttemptingKey, currentDataSource);
      throwables.add(e);
    }
    if (resource != null) {
      notifyEncodeAndRelease(resource, currentDataSource);
    } else {
      runGenerators();
    }
  }
```
通过 decodeFromData 方法来解码资源，如果执行完成则执行 notifyEncodeAndRelease() ，否则，执行runGenerators()

#### decodeFromData()
```java
private <Data> Resource<R> decodeFromData(DataFetcher<?> fetcher, Data data ,
     DataSource dataSource) throws GlideException {
   try {
     if (data == null) {
       return null;
     }
     long startTime = LogTime.getLogTime();
     Resource<R> result = decodeFromFetcher(data, dataSource);
     if (Log.isLoggable(TAG, Log.VERBOSE)) {
       logWithTimeAndKey("Decoded result " + result, startTime);
     }
     return result;
   } finally {
     fetcher.cleanup();
   }
 }
```

#### notifyEncodeAndRelease()
```java
private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
    if (resource instanceof Initializable) {
      ((Initializable) resource).initialize();
    }

    Resource<R> result = resource;
    LockedResource<R> lockedResource = null;
    if (deferredEncodeManager.hasResourceToEncode()) {
      lockedResource = LockedResource.obtain(resource);
      result = lockedResource;
    }
    notifyComplete(result, dataSource);
    stage = Stage.ENCODE;
    try {
      if (deferredEncodeManager.hasResourceToEncode()) {
        deferredEncodeManager.encode(diskCacheProvider, options);
      }
    } finally {
      if (lockedResource != null) {
        lockedResource.unlock();
      }
    }
    // Call onEncodeComplete outside the finally block so that it's not called if the encode process
    // throws.
    onEncodeComplete();
  }
```
然后查看notifyComplete()

```java
private void notifyComplete(Resource<R> resource, DataSource dataSource) {
   setNotifiedOrThrow();
   callback.onResourceReady(resource, dataSource);
 }

 void notifyCallbacksOfResult() {
    ResourceCallbacksAndExecutors copy;
    Key localKey;
    EngineResource<?> localResource;
    synchronized (this) {
      ...
    listener.onEngineJobComplete(this, localKey , localResource);

    for (final ResourceCallbackAndExecutor entry : copy) {
      entry.executor.execute(new CallResourceReady(entry.cb));
    }
    decrementPendingCallbacks();
  }
```
CallResourceReady 是 EngineJob 的一个内部类，然后我就查看这个内部类的 run 方法
```java
@Override
   public void run() {
     synchronized (EngineJob.this) {
       if (cbs.contains(cb)) {
         // Acquire for this particular callback.
         engineResource.acquire();
         callCallbackOnResourceReady(cb);
         removeCallback(cb);
       }
       decrementPendingCallbacks();
     }
   }
 }

 @Synthetic
  synchronized void callCallbackOnResourceReady(ResourceCallback cb) {
    try {
      cb.onResourceReady(engineResource, dataSource);
    } catch (Throwable t) {
      throw new CallbackException(t);
    }
  }
```
cb 就是我们传递过来的SingleRequest
```java
public synchronized void onResourceReady(Resource<?> resource, DataSource dataSource) {
   stateVerifier.throwIfRecycled();
   loadStatus = null;
   if (resource == null) {
     GlideException exception = new GlideException(...);
     onLoadFailed(exception);
     return;
   }

   Object received = resource.get();
   if (received == null || !transcodeClass.isAssignableFrom(received.getClass())) {
     releaseResource(resource);
     GlideException exception = new GlideException(...);
     onLoadFailed(exception);
     return;
   }

   if (!canSetResource()) {
     releaseResource(resource);
     // We can't put the status to complete before asking canSetResource().
     status = Status.COMPLETE;
     return;
   }

   onResourceReady((Resource<R>) resource, (R) received, dataSource);
 }


 private synchronized void onResourceReady(Resource<R> resource, R result , DataSource dataSource) {
     // We must call isFirstReadyResource before setting status.
     boolean isFirstResource = isFirstReadyResource();
     status = Status.COMPLETE;
     this.resource = resource;

     isCallingCallbacks = true;
     try {
       boolean anyListenerHandledUpdatingTarget = false;
       if (requestListeners != null) {
         for (RequestListener<R> listener : requestListeners) {
           anyListenerHandledUpdatingTarget |=listener.onResourceReady(result, model , target , dataSource , isFirstResource);
         }
       }
       anyListenerHandledUpdatingTarget |=
 targetListener != null&& targetListener.onResourceReady(result, model , target , dataSource , isFirstResource);

       if (!anyListenerHandledUpdatingTarget) {
         Transition<? super R> animation =animationFactory.build(dataSource, isFirstResource);
         target.onResourceReady(result, animation);
       }
     } finally {
       isCallingCallbacks = false;
     }

     notifyLoadSuccess();
   }
```
调用了target.onResourceReady(),我们进入 ImageViewTarget 中
```java
public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
   if (transition == null || !transition.transition(resource, this)) {
     setResourceInternal(resource);
   } else {
     maybeUpdateAnimatable(resource);
   }
 }

 private void setResourceInternal(@Nullable Z resource) {
   // Order matters here. Set the resource first to make sure that the Drawable has a valid and
   // non-null Callback before starting it.
   setResource(resource);
   maybeUpdateAnimatable(resource);
 }

```
setResource() 是一个抽象方法，所以需要到子类中去查找
```java
BitmapImageViewTarget
protected void setResource(Bitmap resource) {
   view.setImageBitmap(resource);
 }

 DrawableImageViewTarget
 protected void setResource(@Nullable Drawable resource) {
   view.setImageDrawable(resource);
 }

 ThumbnailImageViewTarget
 protected void setResource(@Nullable T resource) {
    ViewGroup.LayoutParams layoutParams = view.getLayoutParams();
    Drawable result = getDrawable(resource);
    if (layoutParams != null && layoutParams.width > 0 && layoutParams.height > 0) {
      result = new FixedSizeDrawable(result, layoutParams.width, layoutParams.height);
    }

    view.setImageDrawable(result);
  }
```
终于到我们熟悉的view.setImageDrawable()或者view.setImageBitmap()中了

全局完。。

---
搬运地址：    

[Glide4.0源码全解析（一）， GlideAPP 和.with()方法背后的故事](https://blog.csdn.net/github_33304260/article/details/77869221)

[Glide4.0源码全解析（二）， load() 背后的故事](https://blog.csdn.net/github_33304260/article/details/77992717)

[Glide源码解析(三)](https://blog.csdn.net/pmx_121212/article/details/79085947)

[Android图片加载库 Glide 知其然知其所以然 开篇](https://juejin.im/post/5cbea88cf265da03555c7f58)

[Android 图片加载库 Glide 知其然知其所以然之加载](https://juejin.im/post/5cd51235e51d456e831f69f6)

[Glide架构设计艺术](https://www.jianshu.com/p/5c8ce241199e)

[Glide源码导读](https://www.cnblogs.com/angeldevil/p/5841979.html)

[Android源码系列-解密Glide](https://juejin.im/post/5e5c5e23f265da570c753621#heading-29)
