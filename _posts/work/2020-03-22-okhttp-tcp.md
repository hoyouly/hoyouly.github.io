---
layout: post
title: 工作填坑 --- OkHttp 支持 TCP 请求
category: 工作填坑
tags: Retrofit  
---
* content
{:toc}

我们都知道，OkHttp 是我们常用的网络请求框架，可是都是用在了HTTP请求上去了，但是我们的项目中既有HTTP，又有TCP请求，我已经使用了Retrofit+OkHttp+Rxjava的架构，难道再写一套网络请求框架吗，能再这上面改造一下，也同时支持TCP请求吗？然后就有了这篇研究。
在这之前，对OkHttp了解少的，可以先查阅 [OkHttp 源码分析](http://hoyouly.fun/2020/03/16/Okhttp-Source-Code-Analysis/) 对OkHttp 源码有一个基本认识。

OkHttp 真正请求的地方是 CallServerInterceptor 中的。
所有的 Interceptor 存放到了一个集合中，一个一个遍历，但是中间都会执行到 Chain.proceed(），这样才能进去到下一个 Interceptor 中，直到 CallServerInterceptor 这个拦截器中，然后请求网络，
如果我自定义的 Interceptor 不执行 Chain.proceed(），那么就不会执行到了CallServerInterceptor， 并且这个 自定义的 IInterceptor 直接通过TCP方式请求数据，然后直接把数据返回，这样不就可以了吗？

思路有了，可是具体步骤呢？我想应该分三步骤吧！
1. 从RequestBody 中得到请求的action和参数
2. 发送TCP请求得到数据
3. 把请求结果封装成一个Response并返回

# 从Request 中得到请求的action和参数
## 得到action

这个比较容易拿到的，通过Request可以得到url，然后直接调用url的encodedPath() ，就行了，不过这里面有一个坑
我这边得到的参数 是 `/busi/login`,而真正的action 是`busi/login`,所以需要截取一下。
即 String action = url.encodedPath().substring(1);

## 得到参数
这个就有点麻烦了，因为参数是 封装到了 RequestBody 中，所以在我们自定义的 Interceptor中，就需要解析RequestBody ，
下图是 RequestBody的结构

![okio](../../../../images/requestbody_1.png)

红框里面就是我们想要的参数数据！！！     
可是怎么拿到红框里面的参数呢我们知道，这RequestBody 是一个 抽象类，这个实现类是  RequestBuilder.ContentTypeOverridingRequestBody, 可是这个类也仅仅是一个代理类而已，
```java
private static class ContentTypeOverridingRequestBody extends RequestBody {
   private final RequestBody delegate;
   private final MediaType contentType;

   ContentTypeOverridingRequestBody(RequestBody delegate, MediaType contentType) {
     this.delegate = delegate;
     this.contentType = contentType;
   }

   @Override public MediaType contentType() {
     return contentType;
   }

   @Override public long contentLength() throws IOException {
     return delegate.contentLength();
   }

   @Override public void writeTo(BufferedSink sink) throws IOException {
     delegate.writeTo(sink);
   }
 }
```
我之前是想着通过添加addFactory(),自定义一个RequestBody，然后来处理的，可是发现就算我自定义一个类，也白搭，最后得到的还是这个代理类，根本没办法，

然后我就看到了里面有一个 writeTo(BufferedSink sink)方法，这是干嘛的啊，之前没见过BufferedSink 这个类。
后来查资料才知道，这是Okio中的一个类，详情参照上面，是不是可以在这上面做文章呢，然后研究了一下 BufferedSink 这个类的用法
然后就知道了Buffer这个类，实现了 BufferedSink，这个类可以说是神一般的存在，技能读数据，又能写数据。

```java
Buffer buffer = new Buffer();
body.writeTo(buffer);
buffer.close();//数据的真正写入是在close()中，之前的只是在缓存中。
ByteString byteString = buffer.readByteString();
String utf8 = byteString.utf8();
```
创建一个Buffer对象，然后writeTo(),最后close(),

刚开始我使用的是flush(),想着和以前的IO操作一样，可是发现flush()之后并没有作用，数据还是没过来，然后我又研究 CallServerInterceptor ，看他们的数据是怎么过来的，最后发现竟然是调用close()之后，才会写入操作，有点意思。数据真正写入是在close()中，详情参考 [ OKio - 重新定义了“短小精悍”的IO框架  ](https://juejin.im/post/5856680c8e450a006c6474bd),这里面有详细介绍 close()写入数据的流程。
#### Buffer 转 String
好了，继续，
写入Buffer后，可是怎么转String呢，发现了readByteString()转成ByteString,关于ByteString，我刚好看到过，可以直接转String，那这样不就齐活了吗？
```java
ByteString byteString = buffer.readByteString();
String utf8 = byteString.utf8();
```
其实Buffer也提供了转String的方法，buffer.readString(Charset.forName("utf-8"))，这样也转成了utf-8类型的字符串
然后打印结果，发现就是我想要的数据
```
intercept utf8: {"action":"busi/login","data":{"DevID":"M15","DevType":0,"Password":"4QrcOUm6Wau+VuBX8g+IPg==","Username":"test3"}}
```
所以这一段代码就如下

```java
Buffer buffer = new Buffer();
body.writeTo(buffer);
buffer.close();//数据的真正写入是在close()中，之前的只是在缓存中。
String utf8 = buffer.readString(Charset.forName("utf-8"));
Log.d("hoyouly", "intercept utf8: "+utf8);

Response response = null;
if (request.url() != null) {
    // 得到 action  和参数 json
    HttpUrl url = request.url();
    //因为得到的action多带有一个“/”,所以需要单独处理一下，
    String action = url.encodedPath().substring(1);
  ...
}
```
第一步已经完成，第二步发送数据也必将简单，
# 发送TCP请求得到数据
这是之前已经现成的，直接通过action和参数发送即可
```java
//调用 NettyClient.getInstance().send()
String result = NettyClient.getInstance().send(action, utf8);
```
关于NettyClient 的代码，其实就是使用Netty框架，封装的TCP请求。这里就不再多说了。

还剩下最后一步，把得到的结果封装成一个Response，然后返回，这里面同样用到了Okio 操作
# 把请求结果封装成一个Response

其实主要是还是 ResponseBody，
在创建 Response的时候，有一个参数是ResponseBody，这里面就是网络请求返回的数据，所以需要想办法封装一个ResponseBody。
可是查看ResponseBody,发现有三个参数必须要设置的，contentLength，contentType和BufferedSource，里面并没有
```java
// Response.java
public abstract @Nullable MediaType contentType();
public abstract long contentLength();
public abstract BufferedSource source();
```
* contentType 还行，直接通过request就能拿到，
* contentLength 是内容长度，前面得到请求结果，也能设置长度
可是 BufferedSource 是啥啊，没见过啊，肯定还是Okio咯，那么怎么把String转换BufferedSource呢

前面我们能把BufferedSink转成一个String，那么也应该可以的，上面说了Buffer是一个神一般的存在，能读取数据，也能写入数据，那么这次就让他写吧。
## String 转换 BufferedSource

```java
buffer = new Buffer();
buffer.writeString(result, Charset.forName("utf-8"));
Source source = Okio.source(buffer.inputStream());
BufferedSource bufferedSource = Okio.buffer(source))
```
刚开始我还这样写，后来我发现，Buffer就是 BufferedSource的实现类，直接调用 Buffer的 writeUtf8() 就行，然后close(),结束战斗

```java
buffer = new Buffer();
buffer.writeUtf8(result);
buffer.close();
response = new Response.Builder()
    .body(new RealResponseBody(body.contentType().type(), result.length(), buffer))
    .request(request)
    .protocol(Protocol.HTTP_1_0)
    .code(200)
    .message(action + "  is success")
    .build();
```
## build()
因为build()中有规定，所以需要单独设置request,protocol,code,message

```java
public Response build() {
     if (request == null) throw new IllegalStateException("request == null");
     if (protocol == null) throw new IllegalStateException("protocol == null");
     if (code < 0) throw new IllegalStateException("code < 0: " + code);
     if (message == null) throw new IllegalStateException("message == null");
     return new Response(this);
   }
```
* Request 容易 ，直接设置 ，`.request(request)`
* code也容易，请求成功，当然是200，`.code(200)`
* message 也简单，请求成功，直接返回 success ，为了能清楚是哪个action的，所以带上了action。即 `message(action + "  is success")`
### 定义protocol()
protocol 比较头疼，因为 Protocol 是一个Enum，一共支持的协议一共就那么几种。
```java
public enum Protocol {
  HTTP_1_0("http/1.0"),
  HTTP_1_1("http/1.1"),
  SPDY_3("spdy/3.1"),
  HTTP_2("h2"),
  H2_PRIOR_KNOWLEDGE("h2_prior_knowledge"),
  QUIC("quic");
}
```
唯一看着像的就是 SPDY_3，因为 Socket 使用的是Netty框架，而Netty 中是可以实现 SPDY 的，虽然我不知道这到底是啥，但是既然可以实现SPDY，那就SPDY 了，可是发现这个竟然 deprecated，不管了，先这样设置吧，好像没多大关系的。

# 完整代码
所以 这个 Interceptor的代码就是

```java
public class SocketInterceptor implements Interceptor {

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        if (request == null) {
            Log.d("SocketInterceptor", "intercept request is null");
            return null;
        }
        RequestBody body = request.body();
        if (body == null) {
            Log.d("SocketInterceptor", "intercept body is null");
            return null;
        }

        Buffer buffer = new Buffer();
        body.writeTo(buffer);
        buffer.close();//数据的真正写入是在close()中，之前的只是在缓存中。
        String utf8 = buffer.readString(Charset.forName("utf-8"));

        Response response = null;
        if (request.url() != null) {
            // 得到 action  和参数 json
            HttpUrl url = request.url();
            String action = url.encodedPath().substring(1);
            try {
                //调用 NettyClient.getInstance().send()
                String result = NettyClient.getInstance().send(action, utf8);
                if (result == null) {
                    Log.d("SocketInterceptor", "intercept result  is null");
                    return null;
                }
                //把得到的结果 封装成一个response 返回即可
                buffer = new Buffer();
                buffer.writeUtf8(result);
                buffer.close();
                response = new Response.Builder()
                    .body(new RealResponseBody(body.contentType().type(), result.length(), buffer))
                    .request(request)
                    .protocol(Protocol.SPDY_3)
                    .code(200)
                    .message(action + "  is success")
                    .build();

            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return response;
    }
}
```
在创建Okhttp的时候，addInterceptor()上去就行了。

httpClientBuilder.addInterceptor(new SocketInterceptor());

注意，不能使用 addNetworkInterceptor(),因为会报错，原因还在查找中。

---
搬运地址：   
