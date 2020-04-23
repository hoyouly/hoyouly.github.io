---
layout: post
title: 源码分析 - Glide4 之 into()
category: 源码分析
tags: glide4
---
* content
{:toc}

承接上文 [ 源码分析 - Glide4 之 with() 和 load() ](../../../../2019/11/01/glide4-source_analysis/),前面 with() 和 load() 都讲完了，
load() 后返回后的是一个 RequestBuilder ,所以就从 RequestBuilder 的 into() 开始玩吧
现在说是into()

```java
//RequestBuilder.java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    //检查线程
    Util.assertMainThread();
    Preconditions.checkNotNull(view);
    BaseRequestOptions<?> requestOptions = this;
    //在没有调用 transform 方法，并且变换可用的情况下，如果设置了 scaletype ，就会根据你的设置对图片进行处理
    if (!requestOptions.isTransformationSet()&& requestOptions.isTransformationAllowed()&& view.getScaleType() != null) {
      //根据设置的 scaleType ，得到相应的Options
      switch (view.getScaleType()) {
        //用 clone 方法来保证不会影响之后的视图显示效果
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        ...
      }
    }
    return into(glideContext.buildImageViewTarget(view, transcodeClass), null , requestOptions , Executors.mainThreadExecutor());
  }
```
如图，当我们调用into(target)的时候，我们可知
1. 必须在主线程中
2. 通过 buildImageViewTarget() 创建一个 ImageViewTarget ，
3. targetListener 是 null ,是用来加载图片的监听器
4. 根据 View 的 getScaleType 来判断是设置requestOptions
5. 根据 View 和 transcodeClass 来构建一个ViewTarget

先看看 buildImageViewTarget() 吧

#  生成 ImageViewTarget
```java
GlideContext
public <X> ViewTarget<ImageView, X> buildImageViewTarget( @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
  }
//ImageViewTargetFactory.java  
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
根据 clazz 构建不同的 ViewTarget ，这个 clazz 是在创建 RequestBuilder() 的时候传递过来的。    
如果加载图片的时候使用 asBtimap() ,那么就会创建 BitmapImageViewTarget 。    
如果调用 asDrawable() ,或者 asGif() ,那么就创建一个 DrawableImageViewTarget 。   
否则抛出异常。

接下来看真正的into()
# RequestBuilder # into()
函数的定义如下
```java
private <Y extends Target<TranscodeType>> Y into(@NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,Executor callbackExecutor) {}
```

四个参数我们也大概能猜出来到底干嘛的
* target 要加载的载体，比如ImageView
* targetListener 加载时候的监听
* options 加载时候的配置，比如是否圆角，圆形图片等
* callbackExecutor 因为加载图片可能耗时，所以需要在子线程中进行，那么就需要线程池进行管理

```java
//RequestBuilder.java
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
通过 buildRequest() 创建一个 Request ，用于发起一个加载请求，通过后面的代码也可以知道，返回的是 SingleRequest 类型

## RequestBuilder # buildRequest()

```java
//RequestBuilder.java
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
      ...
      Request thumbRequest =thumbnailBuilder.buildRequestRecursive(...);
      isThumbnailBuilt = false;
      coordinator.setRequests(fullRequest, thumbRequest);
      return coordinator;
    } else if (thumbSizeMultiplier != null) {
       //若调用了thumbnail(float sizeMultiplier)走此处
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
先从对象池中得到对象，如果不存在，则创建，然后 init ,使用的享元模式，用于对内存的节省，先从对象池中取，有则共享，减少 new 对象的成本，
创建完成 Request 后，然后就是
```java
//RequestBuilder.java  # into()
requestManager.clear(target);
target.setRequest(request);
requestManager.track(target, request);
```
我们就看看 requestManager # track()

## RequestManager # track()

```java
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
    //说明资源加载完成，
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
* 如果是 WAITING_FOR_SIZE ，则会执行 onSizeReady() ，最后通过 Engine 的 load() 得到数据，同样执行 onResourceReady() ，详情可以看[ 源码分析 - Glide4 之 缓存原理 ](../../../../2020/04/20/glide4-cache/)。
* 如果 model 为 null ，则调用 onLoadFailed()
* target.onLoadStarted()方法，传入了一个 loading 占位图，在也就说，在图片请求开始之前，会先使用这张占位图代替最终的图片显示。

我们就先看看加载成功，执行到 onResourceReady() 的逻辑吧

###  显示加载成功的图片
SingleRequest # onResourceReady()
```java
//SingleRequest.java
public synchronized void onResourceReady(Resource<?> resource, DataSource dataSource) {
  stateVerifier.throwIfRecycled();
  loadStatus = null;
  // resource 为 null ，执行 onLoadFailed() ,显示加载失败的，然后返回

  //resource 不为 null ，说执行到了 onResourceReady() 中
  onResourceReady((Resource<R>) resource, (R) received, dataSource);
}

private synchronized void onResourceReady(Resource<R> resource, R result , DataSource dataSource) {
  ...
    if (!anyListenerHandledUpdatingTarget) {
      Transition<? super R> animation = animationFactory.build(dataSource, isFirstResource);
      target.onResourceReady(result, animation);
    }
  } finally {
    isCallingCallbacks = false;
  }
  notifyLoadSuccess();
}
```
关键的也就是 target.onResourceReady(),这个 target 有很多实现类，如下图

![添加图片](../../../../images/glide_target.png)

这里的 Target 就是我们在第一步的时候生成的那个 Target ，也就是 ImageViewTarget 。

```java
//ImageViewTarget.java
public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
   if (transition == null || !transition.transition(resource, this)) {
     setResourceInternal(resource);
   } else {
     maybeUpdateAnimatable(resource);
   }
 }

 private void setResourceInternal(@Nullable Z resource) {
   setResource(resource);
   maybeUpdateAnimatable(resource);
 }

```
setResource() 是一个抽象方法，有三个子类实现了。 BitmapImageViewTarget , DrawableImageViewTarget 和 ThumbnailImageViewTarget ，不过里面最终执行了setImageXXX()

```java
//BitmapImageViewTarget
protected void setResource(Bitmap resource) {
 view.setImageBitmap(resource);
}

 //DrawableImageViewTarget
protected void setResource(@Nullable Drawable resource) {
 view.setImageDrawable(resource);
}

 //ThumbnailImageViewTarget
protected void setResource(@Nullable T resource) {
  ViewGroup.LayoutParams layoutParams = view.getLayoutParams();
  Drawable result = getDrawable(resource);
  if (layoutParams != null && layoutParams.width > 0 && layoutParams.height > 0) {
    result = new FixedSizeDrawable(result, layoutParams.width, layoutParams.height);
  }
  view.setImageDrawable(result);
}
```
终于到我们熟悉的view.setImageDrawable()或者view.setImageBitmap()了，这样就展示到 UI 上了。

再来瞧瞧加载失败，执行到 onLoadFailed() 的逻辑吧

### 显示加载失败的图片
SingleRequest # onLoadFailed()

```java
private synchronized void onLoadFailed(GlideException e, int maxLogLevel) {
    ...
      if (!anyListenerHandledUpdatingTarget) {
        //设置占位图
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
  if (error == null) {
    error = getErrorDrawable();
  }
  if (error == null) {
    error = getPlaceholderDrawable();
  }
  target.onLoadFailed(error)
}
```
里面调用了 setErrorPlaceholder() ，会先去获取一个 fallback 图，若获取不到 fallback 图则获取 error 的占位图，如果获取不到的话会再去获取一个 PlaceholderDrawable 占位图。

最后通过 target.onLoadFailed(error) 显示到 UI 上。其实内部还是调用了 view.setImageDrawable(drawable) 来显示到 UI 上的

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
