---
layout: post
title: Glide4 源码分析
category: 源码分析
tags: glide4
---
* content
{:toc}


## APT 技术
Annotation Processing Tool 可以在代码编译期间对注解进行处理，并且生成Java 文件，减少手动的代码输入。
注解分为
* 编译时注解  例如Dagger2，ButterKnife，EventBus3 ,
* 运行时注解  例如Retortfit ,通过动态代理生成网络请求。

## GlideModule 注解

```java
/**
 *在编译时，为AppGlideModules和LibraryGlideModules提供注入。
 *替换掉  AndroidManifest.xml 中value="GlideModule" 的 <meta-data /> 。
 *这里需要注意后续需要用到<meta-data />这个标签，先记住此处
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)
public @interface GlideModule {
  /**
   *此处返回的name就是你在使用时的class name。
   *eg:将GlideApp改为GlideAppX
   *那么通过注解生成的类就是GlideAppX
   *那么你使用时候就会是 GlideAppX.with(this)
   */
  String glideName() default "GlideApp";
}
```
## GlideApp


## 继承AppGlideModule
主要是为了重写applyOptions()方法
在Glide被创建之前，对GlideBuilder进行设置

isMannifestParsingEnabled() 用途，是否检测AndroidManifest里面的GlideModule,默认返回true。注解和继承AppGlideModule生成自己的module时，官方要求我们实现这个方法，返回并且false，这样避免AndroidManifest加载两次。

GlideApp.with()其实还是调用了Glide.with()
```Java
/**
 * @see Glide#with(Activity)
 */
@NonNull
public static GlideRequests with(@NonNull Activity activity) {
  return (GlideRequests) Glide.with(activity);
}
```
## Glide # with()
一组静态方法，有好几个重载方法，
例如
```
with(android.app.Activity)
with(android.app.Fragment)
with(android.support.v4.app.Fragment)
with(android.support.v4.app.FragmentActivity)
with(android.view)
```
这些方法最后都调用了`getRetriever(activity).get(activity)`,返回的是一个RequestManager对象

## RequestManagerRetriever # getRetriever(activity)
```Java
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
   // Context could be null for other reasons (ie the user passes in null), but in practice it will
   // only occur due to errors with the Fragment lifecycle.
   Preconditions.checkNotNull(
       context,
       "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
           + "returns null (which usually occurs when getActivity() is called before the Fragment "
           + "is attached or after the Fragment is destroyed).");
   return Glide.get(context).getRequestManagerRetriever();
 }
```
然后到了 Glide.get()
### Glide # get()

```java
@NonNull
 public static Glide get(@NonNull Context context) {
   if (glide == null) {
     synchronized (Glide.class) {
       if (glide == null) {
         checkAndInitializeGlide(context);
       }
     }
   }
   return glide;
 }
```
这个就简单了，单例，双重校验锁

```java
private static void checkAndInitializeGlide(@NonNull Context context) {
    if (isInitializing) {
      throw new IllegalStateException("You cannot call Glide.get() in registerComponents(),"
          + " use the provided Glide instance instead");
    }
    isInitializing = true;
    initializeGlide(context);
    isInitializing = false;
  }
```

```Java
private static void initializeGlide(@NonNull Context context) {
    initializeGlide(context, new GlideBuilder());
  }

  @SuppressWarnings("deprecation")
  private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
    Context applicationContext = context.getApplicationContext();
    GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
      ...
    Glide glide = builder.build(applicationContext);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
      module.registerComponents(applicationContext, glide, glide.registry);
    }
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
    }
    applicationContext.registerComponentCallbacks(glide);
    Glide.glide = glide;
  }

```
1. 使用APT技术动态加载GeneratedAppGlideModule对象，然后配置Glide
2. 使用构造者模式GlideBuilder创建Glide对象
3. 使用applicationContext.registerComponentCallbacks()，Glide 实现了ComponentCallbacks2接口，接口中void onTrimMemory(@TrimMemoryLevel int level);的方法会在操作系统内存不足的时候会调用

### GlideBuilder # build()
得到Glide

```Java
Glide build(@NonNull Context context) {
    ...
    if (engine == null) {
      engine =
          new Engine(
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),
              GlideExecutor.newAnimationExecutor(),
              isActiveResourceRetentionAllowed);
    }
    ...
    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);

    return new Glide(...);
  }
```
* 创建了一个RequestManagerRetriever对象，这个就是 Glide.get(context).getRequestManagerRetriever()得到的那个这个类的 职责就是创建RequestManager，
* 还有创建了一个Engine对象，这个后期会用到

然后看Glide的构造函数干了啥事情
```java
Glide(...) {
    ...
    registry
        //将byteBuffer转换为Flile
        .append(ByteBuffer.class, new ByteBufferEncoder())
        //将inputStream转换为File
        .append(InputStream.class, new StreamEncoder(arrayPool))
        /* Bitmaps */
        .append(Registry.BUCKET_BITMAP, ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder)
        .append(Registry.BUCKET_BITMAP, InputStream.class, Bitmap.class, streamBitmapDecoder)
        .append(
            Registry.BUCKET_BITMAP,
            ParcelFileDescriptor.class,
            Bitmap.class,
            parcelFileDescriptorVideoDecoder)
        .append(
            Registry.BUCKET_BITMAP,
            AssetFileDescriptor.class,
            Bitmap.class,
            VideoDecoder.asset(bitmapPool))
        .append(Bitmap.class, Bitmap.class, UnitModelLoader.Factory.<Bitmap>getInstance())
        .append(
            Registry.BUCKET_BITMAP, Bitmap.class, Bitmap.class, new UnitBitmapDecoder())
        .append(Bitmap.class, bitmapEncoder)
        /* BitmapDrawables */
        //ByteBuffer解码成Bitmap :decoderRegistry
        .append(
            Registry.BUCKET_BITMAP_DRAWABLE,
            ByteBuffer.class,
            BitmapDrawable.class,
            new BitmapDrawableDecoder<>(resources, byteBufferBitmapDecoder))
        ...
    glideContext =new GlideContext(...);
  }

```
注册了ModuleLoader,Encoder,Decoder,Transcoder这四个组件，使其具备转换模型数据。解析String，url等源，解析加载Bitmap，Drawable，file用的Decoder，对资源进行转码的Transcoder,将资源转File的Encoder


#### RequestManagerRetriever # get()
```Java
@NonNull
 public RequestManager get(@NonNull FragmentActivity activity) {
   if (Util.isOnBackgroundThread()) {
     return get(activity.getApplicationContext());
   } else {
     assertNotDestroyed(activity);
     FragmentManager fm = activity.getSupportFragmentManager();
     return supportFragmentGet(
         activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
   }
 }

 @NonNull
  public RequestManager get(@NonNull Context context) {
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }

    return getApplicationManager(context);
  }

```
将参数分为两种
1. 当传入的是 Application对象的时候，就会调用getApplicationManager(context)得到RequestManager对象
2. 传入的不是Application对象的时候，就会执行相应的方法，但是这三个方法都最终执行到了`fragmentGet(activity, fm,  null, isActivityVisible(activity));或者 supportFragmentGet(activity, fm,  null, isActivityVisible(activity))`
3. 如果是在子线程中，默认就当Application处理

![](../../../../images/glide_requestmanagerretriever_get.png)

如图，get()最终都会执行到fragmentGet（）和supportFragmentGet（）这两个方法中。
那么就看看这两个方法的区别啊
```java
private RequestManager fragmentGet(@NonNull Context context,
     @NonNull android.app.FragmentManager fm,
     @Nullable android.app.Fragment parentHint,
     boolean isParentVisible) {
   RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
   RequestManager requestManager = current.getRequestManager();
   if (requestManager == null) {
     // TODO(b/27524013): Factor out this Glide.get() call.
     Glide glide = Glide.get(context);
     requestManager =
         factory.build(
             glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
     current.setRequestManager(requestManager);
   }
   return requestManager;
 }

 private RequestManager supportFragmentGet(
     @NonNull Context context,
     @NonNull FragmentManager fm,
     @Nullable Fragment parentHint,
     boolean isParentVisible) {
   SupportRequestManagerFragment current =
       getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
   RequestManager requestManager = current.getRequestManager();
   if (requestManager == null) {
     // TODO(b/27524013): Factor out this Glide.get() call.
     Glide glide = Glide.get(context);
     requestManager =
         factory.build(
             glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
     current.setRequestManager(requestManager);
   }
   return requestManager;
 }

```
创建一个 SupportRequestManagerFragment 或者RequestManagerFragment ,这就是为啥我们在得到Fragment的数量的时候，会多出来一个Fragment的原因

添加这个隐藏的Fragment，主要是为了监听当前Activity的生命周期，如果Activity已经销毁，那么对应的Fragment也就没有了，可以在这个隐藏的Fragment的销毁的时候，停止加载图片。

创建一个RequestManager

## RequestBuilder # into()
load()返回的是GlideRequest 对象，但是在GlideRequest中并没有查看到into方法，所以需要到GlideRequest 父类RequestBuilder中查找

```java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);
    BaseRequestOptions<?> requestOptions = this;
    //在没有调用transform方法，并且变换可用的情况下，如果设置了scaletype，就会根据你的设置对图片进行处理
    if (!requestOptions.isTransformationSet()&& requestOptions.isTransformationAllowed()&& view.getScaleType() != null) {
      switch (view.getScaleType()) {
        //用clone方法来保证不会影响之后的视图显示效果
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
        requestOptions,
        Executors.mainThreadExecutor());
  }
```
如图，当我们调用into(target)的时候，我们可知
1. 必须在主线程中
2. targetListener 是null,是用来加载图片的监听器
3. 根据View的getScaleType来判断是设置requestOptions
4. 根据View 和transcodeClass来构建一个ViewTarget
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
根据clazz构建不同的ViewTarget，这个clazz是在创建RequestBuilder()的时候传递过来的，如果加载图片的时候使用asBtimap(),那么就会创建BitmapImageViewTarget,如果调用asDrawable(),或者asGif(),那么就创建一个DrawableImageViewTarget，否则抛出异常。

接下来看真正的into()

```java
private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    //若load方法没有调用会抛出异常
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    Request request = buildRequest(target, targetListener, options, callbackExecutor);

    Request previous = target.getRequest();
    //若上一个请求和新建请求一致并且使用了内存缓存或者上一个请求没有完成则将新建请求回收，并保证上一个请求的运行
    if (request.isEquivalentTo(previous)&&!isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      request.recycle();
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        previous.begin();
      }
      return target;
    }
    //清除target之前的请求并设置新的请求
    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);
    return target;
  }

```
### buildRequest()

```java
private Request buildRequest(
      Target<TranscodeType> target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> requestOptions,
      Executor callbackExecutor) {
    return buildRequestRecursive(
        target,
        targetListener,
        /*parentCoordinator=*/ null,
        transitionOptions,
        requestOptions.getPriority(),
        requestOptions.getOverrideWidth(),
        requestOptions.getOverrideHeight(),
        requestOptions,
        callbackExecutor);
  }

  private Request buildRequestRecursive(...) {

    // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
    ErrorRequestCoordinator errorRequestCoordinator = null;
    if (errorBuilder != null) {
      errorRequestCoordinator = new ErrorRequestCoordinator(parentCoordinator);
      parentCoordinator = errorRequestCoordinator;
    }

    Request mainRequest =
        buildThumbnailRequestRecursive(...);

    if (errorRequestCoordinator == null) {
      return mainRequest;
    }

    int errorOverrideWidth = errorBuilder.getOverrideWidth();
    int errorOverrideHeight = errorBuilder.getOverrideHeight();
    if (Util.isValidDimensions(overrideWidth, overrideHeight)
        && !errorBuilder.isValidOverride()) {
      errorOverrideWidth = requestOptions.getOverrideWidth();
      errorOverrideHeight = requestOptions.getOverrideHeight();
    }

    Request errorRequest =
        errorBuilder.buildRequestRecursive(...);
    errorRequestCoordinator.setRequests(mainRequest, errorRequest);
    return errorRequestCoordinator;
  }

```
* 如果我们调用了error()方法，errorBuilder就不会为null，
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
我们发现大部分都会执行到obtainRequest(),这个obtainRequest可以获取到一个Request对象。

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
先从对象池中得到对象，如果不存在，则创建，然后init,使用的享元模式，处于对内存的节省，先从对象池中取，有则共享，减少new对象的成本，
创建完成Request后，然后就是
```java
requestManager.clear(target);
target.setRequest(request);
requestManager.track(target, request);

synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
  //保存target的记录，方便清除请求
   targetTracker.track(target);
   requestTracker.runRequest(request);
 }

 public void runRequest(@NonNull Request request) {
    requests.add(request);
    //判断Glide当前是不是处理暂停状态
    if (!isPaused) {
      //调用Request的begin()方法来执行Request
      request.begin();
    } else {
      //Request添加到待执行队列里面，等暂停状态解除了之后再执行
      request.clear();
      pendingRequests.add(request);
    }
  }
```
因为我们创建的是SingleRequest,所以进入到了SingleRequest的begin中
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
    //是否使用了override() API为图片指定了一个固定的宽高
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      onSizeReady(overrideWidth, overrideHeight);
    } else {
      //没有的话，会根据ImageView的layout_width和layout_height值做一系列的计算，来算出图片应该的宽高。在计算完之后，它也会调用onSizeReady()方法
      target.getSize(this);
    }

    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
        && canNotifyStatusChanged()) {
      target.onLoadStarted(getPlaceholderDrawable());
    }
  }

```
* 如果model 为null，则调用 onLoadFailed()
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
      //TODO: what if this is a thumbnail request?
      boolean anyListenerHandledUpdatingTarget = false;
      if (requestListeners != null) {
        for (RequestListener<R> listener : requestListeners) {
          anyListenerHandledUpdatingTarget |=
              listener.onLoadFailed(e, model, target, isFirstReadyResource());
        }
      }
      anyListenerHandledUpdatingTarget |=
          targetListener != null
          && targetListener.onLoadFailed(e, model, target, isFirstReadyResource());

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
里面调用了setErrorPlaceholder（），会先去获取一个fallback图，若获取不到fallback图则获取error的占位图，如果获取不到的话会再去获取一个loading占位图。


target.onLoadStarted()方法，这个方法传入了一个loading占位图，在也就说，在图片请求开始之前，会先使用这张占位图代替最终的图片显示。

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

    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);
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
    //看EngineJob中是否已经包含当前任务
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

activeResources是一个以弱引用为值的map,他是用于存储使用中的资源，

MemoryCache是一个LruResourceCache，先从activeResources中获取，获取不到则到MemoryCache中获取，获取到了则将资源缓存到activeResources，获取到后均会进行引用计数，获取不到则看EngineJob中是否已经包含当前任务，如果有则替换回调，没有则新建任务，


```Java
public synchronized void start(DecodeJob<R> decodeJob) {
  this.decodeJob = decodeJob;
  GlideExecutor executor = decodeJob.willDecodeFromCache()
      ? diskCacheExecutor
      : getActiveSourceExecutor();
  executor.execute(decodeJob);
}
```
主要根据是否有缓存来决定从磁盘上加载资源或是从网络上加载资源，接下来我们看decodeJob的run方法：
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
INITIALIZE和SWITCH_TO_SOURCE_SERVICE最终都执行到了runGenerators(),如果DECODE_DATA，那么执行到decodeFromRetrievedData()中
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
重点看 currentGenerator.startNext()，而currentGenerator有三种类型，
* SourceGenerator    加载磁盘上原大小资源
* DataCacheGenerator  加载磁盘上调整后大小资源
* ResourceCacheGenerator  网络获取资源

```java
SourceGenerator
@Override
 public boolean startNext() {
   //当dataToCache不为空的时候，将dataToCache放入内存中
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
   //获取能够用的LoadData对象，然后调用它的LoadData（）
   while (!started && hasNextModelLoader()) {
     loadData = helper.getLoadData().get(loadDataListIndex++);
     if (loadData != null
         && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
         || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
       started = true;
       loadData.fetcher.loadData(helper.getPriority(), this);
     }
   }
   return started;
 }

```
SourceGenerator实现了DataCallBack接口

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
      cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
          loadData.fetcher.getDataSource(), originalKey);
    }
  }

```
获取到数据可以被缓存的情况下就会将data赋值给dataToCache，如果不能被赋就回调传入的FetcherReadyCallback对象的onReadyCallback方法。

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
2. 如果不为null，则通过DecodeHelper来获得List中的ModelLoader，即helper.getModelLoaders(cacheFile)
3. modelLoader.buildLoadData获取LoadData，最后通过loadData.fetcher.loadData来加载资源

loadData有很多实现，在加载好资源后最终都会回调 callback.onDataReady(result);这个方法会回调onDataFetcherReady方法，
```Java
public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher,
     DataSource dataSource, Key attemptedKey) {
   this.currentSourceKey = sourceKey;
   this.currentData = data;
   this.currentFetcher = fetcher;
   this.currentDataSource = dataSource;
   this.currentAttemptingKey = attemptedKey;
   //如果DecodeJob的线程不是当前线程，则重新加载，
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
      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
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
通过decodeFromData方法来解码资源，如果执行完成则执行notifyEncodeAndRelease（），否则，执行runGenerators()

#### decodeFromData()
```java
private <Data> Resource<R> decodeFromData(DataFetcher<?> fetcher, Data data,
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
    listener.onEngineJobComplete(this, localKey, localResource);

    for (final ResourceCallbackAndExecutor entry : copy) {
      entry.executor.execute(new CallResourceReady(entry.cb));
    }
    decrementPendingCallbacks();
  }
```
CallResourceReady 是EngineJob的一个内部类，然后我就查看这个内部类的run方法
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


 private synchronized void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
     // We must call isFirstReadyResource before setting status.
     boolean isFirstResource = isFirstReadyResource();
     status = Status.COMPLETE;
     this.resource = resource;

     isCallingCallbacks = true;
     try {
       boolean anyListenerHandledUpdatingTarget = false;
       if (requestListeners != null) {
         for (RequestListener<R> listener : requestListeners) {
           anyListenerHandledUpdatingTarget |=listener.onResourceReady(result, model, target, dataSource, isFirstResource);
         }
       }
       anyListenerHandledUpdatingTarget |=
           targetListener != null&& targetListener.onResourceReady(result, model, target, dataSource, isFirstResource);

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
调用了target.onResourceReady(),我们进入ImageViewTarget中
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
setResource()是一个抽象方法，所以需要到子类中去查找
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
[Glide4.0源码全解析（一），GlideAPP和.with()方法背后的故事](https://blog.csdn.net/github_33304260/article/details/77869221)    
[Glide4.0源码全解析（二），load()背后的故事](https://blog.csdn.net/github_33304260/article/details/77992717)    
[Glide源码解析(三)](https://blog.csdn.net/pmx_121212/article/details/79085947)     
[Android图片加载库Glide 知其然知其所以然 开篇](https://juejin.im/post/5cbea88cf265da03555c7f58)    
[Android 图片加载库Glide知其然知其所以然之加载](https://juejin.im/post/5cd51235e51d456e831f69f6)    
[Glide架构设计艺术](https://www.jianshu.com/p/5c8ce241199e)     
[Glide源码导读](https://www.cnblogs.com/angeldevil/p/5841979.html)
