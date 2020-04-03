---
layout: post
title: 源码分析 - OkHttp
category: 源码分析
tags: Android OkHttp
---

* content
{:toc}

## 前言
把大象放到冰箱里面，一共需要几步。
1. 打开冰箱门
2. 把大象放进去
3. 关上冰箱门

其实OkHttp 请求也很简单
1. 装配Request（涉及到了请求方法，url，cookie请求）
2. Client 端发送Request请求
3. 接收服务端返回的Response数据

so easy

OkHttp 的使用请参考 [扫盲系列 - OkHttp 基本用法](../../../../../article-detail/2018/03/25/OkHttp/)

以下是OKHTTP一个常见异步用法实例

```java
OkHttpClient okHttpClient = new OkHttpClient();//创建OkHttpClient对象
Request request = new Request.Builder()//
    .url("http://www.baidu.com")//请求接口，如果需要传递参数，拼接到后面
    .build();//创建Request对象
    //发送Request请求
okHttpClient.newCall(request).enqueue(new Callback() {

    @Override
    public void onFailure(Call call, IOException e) {
    }
    //接收返回的Response 数据
    @Override
    public void onResponse(Call call, Response response) throws IOException {
        if (response.isSuccessful()) {//请求成功
            Log.e("hoyouly", "onResponse currentThread: " + Thread.currentThread() //
                + " \n code: " + response.code() //
                + " \n body: " + response.body().string()//
                + " \n message: " + response.message());
        }
    }
});
```
1. 创建 OkHttpClient 对象    这个相当于弄一个冰箱
2. 创建 Request 请求        打开冰箱门
3. 发送Request          放入大象
4. 接收Response         关闭冰箱门

这比把大象放入冰箱多了一步，就是找冰箱的过程。整体的流程图如下

![](../../../../../article-detail/images/okhttp_liuchengtu.webp)

接下来就一步一步说


## 创建 OkHttpClient 对象
这个就没啥好说的，可以通过直接new 一个OkHttpClient,也可以通过OkHttpClient.Builder  build()一个，建造者模式嘛。

```java

public OkHttpClient() {
  this(new Builder());
}

public Builder() {
   dispatcher = new Dispatcher();//分发器
   protocols = DEFAULT_PROTOCOLS;//协议
   connectionSpecs = DEFAULT_CONNECTION_SPECS;//传输层版本和连接协议
   eventListenerFactory = EventListener.factory(EventListener.NONE);
   proxySelector = ProxySelector.getDefault();//代理选择
   if (proxySelector == null) {
     proxySelector = new NullProxySelector();
   }
   cookieJar = CookieJar.NO_COOKIES;//cookie
   socketFactory = SocketFactory.getDefault();//socket 工厂
   hostnameVerifier = OkHostnameVerifier.INSTANCE;//  主机名字确认
   certificatePinner = CertificatePinner.DEFAULT;//  证书链
   proxyAuthenticator = Authenticator.NONE;//代理身份验证
   authenticator = Authenticator.NONE;// 本地身份验证
   connectionPool = new ConnectionPool(); //连接池,复用连接
   dns = Dns.SYSTEM;//域名
   followSslRedirects = true;//安全套接层重定向
   followRedirects = true; //本地重定向
   retryOnConnectionFailure = true;//重试连接失败
   callTimeout = 0;
   connectTimeout = 10_000;//连接超时
   readTimeout = 10_000;//read 超时
   writeTimeout = 10_000;//write 超时
   pingInterval = 0;
  }
```
这些都是默认的，几乎都和OkHttp有关。可以通过build模式进行修改。如下就是对OkHttp的一些修改接口

![](../../../../../article-detail/images/okhttp_build.png)

OkHttpClient 创建完成。接下来就是创建Request 对象

## 创建 Request 对象。
这个和OkhttpClient差不多的，同样是通过Build模式，创建一个Request，里面设置了URL，还可以设置其他一些属性的，例如
![](../../../../../article-detail/images/request_build.png)
这个就不多讲了，
接下来就看发送Request

## newCall
在发送Request之前，需要有一个创建一个Call对象，使用这个Call对象来发送Request，
所以可以看到 newCall()中先创建了一个Call对象，即 RealCall，它包装了Request和OKHttpClient这两个类的实例。这点很重要

```java
//OkHttpClient.java
@Override public Call newCall(Request request) {
   return RealCall.newRealCall(this, request, false /* for web socket */);
 }

//RealCall.java
 static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
   // Safely publish the Call instance to the EventListener.
   RealCall call = new RealCall(client, originalRequest, forWebSocket);
   call.eventListener = client.eventListenerFactory().create(call);
   return call;
 }

```
RealCall 实现了Call 这个接口，大部分的操作都在这个Call，这个接口是OkHttp框架的操作核心。
```java

public interface Call extends Cloneable {
  Request request();
  Response execute() throws IOException;
  void enqueue(Callback responseCallback);
  void cancel();
  boolean isExecuted();
  interface Factory {
    Call newCall(Request request);
  }
}
```
接下来就是异步执行了 enqueue(),用来发送Request。
## 发送Request
异步请求enqueue().

```java
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  eventListener.callStart(this);
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

client.dispatcher()  就是  Dispatcher ，它是一个分发器，它持有线程池，异步任务队列和同步任务队列，会依照不同的策略执行。

### Dispatcher
中有三个队列,等待异步执行的队列
```java
/** Ready async calls in the order they'll be run. */
 private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

 /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
 private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

 /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
 private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

接下来继续 看 enqueue()

```java
//Dispatcher.java
void enqueue(AsyncCall call) {
    synchronized (this) {
      readyAsyncCalls.add(call);
    }
    promoteAndExecute();
  }

private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }
    return isRunning;
  }


public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }

```

AsyncCall  是RealCall的一个内部类，持有RealCall的引用，是对RealCall进行了封装

ExecutorService 是一个核心线程数为0的线程池，其实就是 一个 CachedTheadPool类型的线程池，关于线程池的知识，可以查看  [ Android 线程池 ](../../../../../article-detail/2018/05/12/Android-ThreadPoolExecutor/)

在Dispatcher 中会根据情况，把Call加入请求队列还是等待队列，在请求队列中，就会在线程池中执行这个请求。


```java
//AsyncCall.java
void executeOn(ExecutorService executorService) {
    assert (!Thread.holdsLock(client.dispatcher()));
    boolean success = false;
    try {
      executorService.execute(this);
      success = true;
    } catch (RejectedExecutionException e) {
      InterruptedIOException ioException = new InterruptedIOException("executor rejected");
      ioException.initCause(e);
      eventListener.callFailed(RealCall.this, ioException);
      responseCallback.onFailure(RealCall.this, ioException);
    } finally {
      if (!success) {
        client.dispatcher().finished(this); // This call is no longer running!
      }
    }
  }
```

因为 AsyncCall extends NamedRunnable  ，NamedRunnable 实现了Runnable，所以就可以直接查看run()方法了。

```java
//，NamedRunnable.java
@Override public final void run() {
  String oldName = Thread.currentThread().getName();
  Thread.currentThread().setName(name);
  try {
    execute();
  } finally {
    Thread.currentThread().setName(oldName);
  }
}
```

最终执行到了 AsyncCall 的execute()方法

```java
@Override protected void execute() {
  boolean signalledCallback = false;
  timeout.enter();
  try {
    Response response = getResponseWithInterceptorChain();
    if (retryAndFollowUpInterceptor.isCanceled()) {
      signalledCallback = true;
      responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
    } else {
      signalledCallback = true;
      responseCallback.onResponse(RealCall.this, response);
    }
  } catch (IOException e) {
    e = timeoutExit(e);
    if (signalledCallback) {
      // Do not signal the callback twice!
      Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
    } else {
      eventListener.callFailed(RealCall.this, e);
      responseCallback.onFailure(RealCall.this, e);
    }
  } finally {
    client.dispatcher().finished(this);
  }
}

```

感觉已经很靠近了，因为我们看到了 onResponse()/onFailure() 这两个方法， 这个responseCallback 就是在enqueue()中传递过来的 Callback responseCallback，

那么请求网络的地方，就是  getResponseWithInterceptorChain() 这个了，


```java
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  //添加开发者应用层自定义的Interceptor
  interceptors.addAll(client.interceptors());
  //这个Interceptor是处理请求失败的重试，重定向
  interceptors.add(retryAndFollowUpInterceptor);
  //这个Interceptor工作是添加一些请求的头部或其他信息
  //并对返回的Response做一些友好的处理（有一些信息你可能并不需要）
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  //这个Interceptor的职责是判断缓存是否存在，读取缓存，更新缓存等等
  interceptors.add(new CacheInterceptor(client.internalCache()));
  //这个Interceptor的职责是建立客户端和服务器的连接
  interceptors.add(new ConnectInterceptor(client));
  if (!forWebSocket) {
    //添加开发者自定义的网络层拦截器
    interceptors.addAll(client.networkInterceptors());
  }
  //这个Interceptor的职责是向服务器发送数据，并且接收服务器返回的Response
  interceptors.add(new CallServerInterceptor(forWebSocket));
  //一个包裹这request的chain
  Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
      originalRequest, this, eventListener, client.connectTimeoutMillis(),
      client.readTimeoutMillis(), client.writeTimeoutMillis());
  //把chain传递到第一个Interceptor手中
  return chain.proceed(originalRequest);
}
```
1. Interceptor 的执行是有顺序的。也就意味这当我们自定义Interceptor是需要注意添加顺序
2. 在开发者自定义的Interceptor,是有两种不同的拦截器可以自定义的。

## 拦截器
使用责任链模式模式，添加了各种 interceptors，然后执行了 RealInterceptorChain 的 proceed()

```java
//RealInterceptorChain.java
@Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }
  //这个方法用来获取list中下一个Interceptor，并调用它的intercept（）方法

public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    //如果我们已经有一个stream。确定即将到来的request会使用它
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1) + " must retain the same host and port");
    }

  //如果我们已经有一个stream， 确定chain.proceed()唯一的call
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)+ " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.取出来下一个拦截器，然后执行 intercept()
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout, writeTimeout);
  //从list中获取到第一个Interceptor
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException( "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```

这个有点像是递归调用。

关键代码是  
```java
// 实例化下一个拦截器对应的RealIterceptorChain对象， index刚开始是0，这里加一，下次取的时候就是下一个 interceptors
RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
    connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
    writeTimeout);
//得到当前的拦截器：
Interceptor interceptor = interceptors.get(index);
//调用当前拦截器的intercept()方法，并将下一个拦截器的RealIterceptorChain对象传递下去
Response response = interceptor.intercept(next);
```
1. 实例化下一个拦截器对应的RealIterceptorChain对象，这个对象会再传递给当前的拦截器
2. 得到当前的拦截器：interceptors是存放拦截器的ArryList
3. 调用当前拦截器的intercept()方法，并将下一个拦截器的RealIterceptorChain对象传递下去
4. 在下一个Interceptor的 intercept()中，会执行 chain.proceed(request)，这样就进入到下次循环中了

这也是为啥，我们自定义的Interceptor ，在intercept()中，都会执行  chain.proceed(request) 这行代码的缘故
如果不执行这行代码，那么责任链模式就没办法进行了。
说到这里，就可以解释一下  addInterceptor()和 addNetworkInterceptor()区别了

### addInterceptor()和 addNetworkInterceptor()区别
addInterceptor() 添加应用拦截器
* 不需要担心中间过程的响应，如重定向和重试
* 总是只调用一次，即便是HTTP从缓存中获取
* 观察应用程序的初衷，不关心Okhttp注入信息的头信息，如If-NONE-Match
* 允许短路而不调用Chain.proceed(),即终止调用
* 允许重试，试Chain.proceed()调用多次
addNetworkInterceptor()  添加网络拦截器
* 能够操作中间过程的响应，如重定向和重试
* 当网络短路而返回缓存的时候不调用
* 只观察在网络传输中的数据
* 携带请求来访问连接


注意，如果添加的 addInterceptor()中不执行 chain.proceed(request) ，那么就不会执行到下面的 网络重试，以及最关键的 CallServerInterceptor 这个Interceptor了。


一个Interceptor集合，从第一个开始，然后执行 intercept()的方法，里面会创建 RealInterceptorChain,然后执行下一个proceed()方法，就这样一直到了最后一个，
也就是 CallServerInterceptor中，

除了 在client中自己设置的interceptor,第一个调用的就是retryAndFollowUpInterceptor


### RetryAndFollowUpInterceptor

负责失败重试以及重定向

```java
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    ...
    Response priorResponse = null;
    while (true) {
      ...
      try {
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        ...
      }
      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder().priorResponse(priorResponse.newBuilder().body(null).build()).build();
      }
      ...
      priorResponse = response;
    }
  }
```
看到没有，这里面也执行了 realChain.proceed(），所以会传递到下一个拦截器中，即 BridgeInterceptor

### BridgeInterceptor

负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应的

```java
@Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();
    //检查request。将用户的request转换为发送到server的请求
    RequestBody body = userRequest.body();
    ...
    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder().request(userRequest);
    ...
    return responseBuilder.build();
  }

```
同样执行了 chain.proceed(），那么就可以继续传递拦截器。即 CacheInterceptor

### CacheInterceptor
缓存拦截器

```java

@Override public Response intercept(Chain chain) throws IOException {
    Response cacheCandidate = cache != null ? cache.get(chain.request()) : null;

    long now = System.currentTimeMillis();

    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    //如果我们禁止使用网络，且缓存为null，失败
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    //没有网络请求，跳过网络，返回缓存
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      //如果我们因为I/O或其他原因崩溃，不要泄漏缓存体
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    //如果我们有一个缓存的response，然后我们正在做一个条件GET
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

      //更新缓存，在剥离content-Encoding之前
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }
    return response;
  }

```
缓存策略
1. 根据request来判断cache中是否有缓存的response，如果设置了缓存，即 在OkHttpClient.Builder 中 执行了cache()，那么 cache 就不为null，
2. 如果有，得到这个response，然后进行判断当前response是否有效，没有将cacheCandate赋值为空
3. 根据request判断缓存的策略，是否要使用了网络，缓存 或两者都使用
4. 调用下一个拦截器，决定从网络上来得到response
5. 如果本地已经存在cacheResponse，那么让它和网络得到的networkResponse做比较，决定是否来更新缓存的cacheResponse
6. 缓存未经缓存过的response

同样也是执行了 chain.proceed(networkRequest)，继续传递下一个，即 ConnectInterceptor

### ConnectInterceptor
建立连接

```java
@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Request request = realChain.request();
  StreamAllocation streamAllocation = realChain.streamAllocation();

  // We need the network to satisfy this request. Possibly for validating a conditional GET.
  boolean doExtensiveHealthChecks = !request.method().equals("GET");
  HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
  RealConnection connection = streamAllocation.connection();

  return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

其实就是创建了一个 HttpCodec对象。 是对HTTP协议的一种抽象，有两个实现：Http1Codec和Http2Codec，顾名思义，它们分别对应 HTTP/1.1 和 HTTP/2 版本的实现
也执行了 realChain.proceed(），那么继续，如果不是webSocket,那么就是 NetworkInterceptors

### NetworkInterceptors

配置OkHttpClient时设置的 NetworkInterceptors。这里面一般也会执行的，如果不执行realChain.proceed(），那么就执行不到下一步了，即CallServerInterceptor，真正的发送和接收数据
### CallServerInterceptor
发送和接收数据
```java
@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  HttpCodec httpCodec = realChain.httpStream();
  StreamAllocation streamAllocation = realChain.streamAllocation();
  RealConnection connection = (RealConnection) realChain.connection();
  Request request = realChain.request();

  long sentRequestMillis = System.currentTimeMillis();

  realChain.eventListener().requestHeadersStart(realChain.call());
  httpCodec.writeRequestHeaders(request);
  realChain.eventListener().requestHeadersEnd(realChain.call(), request);

  Response.Builder responseBuilder = null;
  if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
    if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
      httpCodec.flushRequest();
      realChain.eventListener().responseHeadersStart(realChain.call());
      responseBuilder = httpCodec.readResponseHeaders(true);
    }

    if (responseBuilder == null) {
      // Write the request body if the "Expect: 100-continue" expectation was met.
      realChain.eventListener().requestBodyStart(realChain.call());
      long contentLength = request.body().contentLength();
      CountingSink requestBodyOut =
          new CountingSink(httpCodec.createRequestBody(request, contentLength));
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

      request.body().writeTo(bufferedRequestBody);
      bufferedRequestBody.close();
      realChain.eventListener()
          .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
    } else if (!connection.isMultiplexed()) {
      streamAllocation.noNewStreams();
    }
  }

  httpCodec.finishRequest();

  if (responseBuilder == null) {
    realChain.eventListener().responseHeadersStart(realChain.call());
    responseBuilder = httpCodec.readResponseHeaders(false);
  }

  Response response = responseBuilder
      .request(request)
      .handshake(streamAllocation.connection().handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build();

  int code = response.code();
  if (code == 100) {
    // server sent a 100-continue even though we did not request one.
    // try again to read the actual response
    responseBuilder = httpCodec.readResponseHeaders(false);

    response = responseBuilder
            .request(request)
            .handshake(streamAllocation.connection().handshake())
            .sentRequestAtMillis(sentRequestMillis)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build();

    code = response.code();
  }

  realChain.eventListener().responseHeadersEnd(realChain.call(), response);

  if (forWebSocket && code == 101) {
    // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
    response = response.newBuilder()
        .body(Util.EMPTY_RESPONSE)
        .build();
  } else {
    response = response.newBuilder()
        .body(httpCodec.openResponseBody(response))
        .build();
  }

  if ("close".equalsIgnoreCase(response.request().header("Connection"))
      || "close".equalsIgnoreCase(response.header("Connection"))) {
    streamAllocation.noNewStreams();
  }

  if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
    throw new ProtocolException(
        "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
  }

  return response;
}

```

这里面没有执行 chain.proceed(request)，所以责任链到此结束，在这个Intercept中，真正得到网络数据，然后再通过 chain.proceed(request)一级一级返回 response，最后到   RealCall 的 getResponseWithInterceptorChain()中，这样就能继续执行下面的操作了，即返回这个result，还是执行callFailed().



---   

搬运地址：    


[从OKHttp框架看代码设计](https://juejin.im/post/581311cabf22ec0068826aff)

[OKHttp源码解析](https://www.jianshu.com/p/27c1554b7fee)

[OkHttp中的Socket连接](https://blog.csdn.net/qqqq245425070/article/details/100162964)

[Andriod 网络框架 OkHttp 源码解析](https://juejin.im/post/5bc89fbc5188255c713cb8a5)

[Retrofit使用 addInterceptor和addNetworkInterceptor的区别](https://blog.csdn.net/qq_38219041/article/details/72935142)
