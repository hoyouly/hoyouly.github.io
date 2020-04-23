---
layout: post
title: 工作填坑 - Retrofit 上传图片
category: 工作填坑
tags: Retrofit  
---
<!-- * content -->
<!-- {:toc} -->
# 前言
最近遇到一个很奇怪的接口，说奇怪吧，是因为我使用 Postman 能调试通，使用 xutils 框架也能测试通，单独使用 OkHttp 测试通过。可是唯独使用 Retrofit + Rxjava + OkHttp 这个组合框架一直测试不通过。
先看看接口长啥样子吧。这是上传图片，接口真让我眼前一亮

![添加图片](../../../../images/upload_image.png)

参数就一句话：
```
Post 的 body 内容是图片的二进制数据，把图片数据读出来直接写入 body 就可以
```
第二个接口是可以通过返回的字符串，拼接成这个图片的路径。用来验证是否上传成功。
# 使用 xutils 框架
代码如下
```java
RequestParams requestParams = new RequestParams(BASE_URL + "/api/uploadimg");
requestParams.addBodyParameter("body", file , "image/jpg");
String result = "";
try {
    result = x.http().postSync(requestParams, String.class);
    Log.d("hoyouly", " uploadPic:  " + result);
} catch (Throwable throwable) {
    throwable.printStackTrace();
    Log.d("hoyouly", "postSync  " + throwable.getMessage());
}
```
得到的字符串 根据第二个接口拼接成一个 URL ，是可以查看到这个图片的。说明上传成功了。
# 使用 OkHttp 框架
代码如下
```java
File file = new File(path);
RequestBody requestFile = RequestBody.create(MediaType.parse("image/jpg"), file);
//创建Request
final Request request = new Request.Builder().url(BASE_URL +"uploadimg").post(requestFile).build();
OkHttpClient mOkHttpClient = new OkHttpClient();
final Call call = mOkHttpClient.newBuilder().writeTimeotTimeUnit.SECONDS).build().newCall(request);
try {
    Response response = call.execute();
    if (response.isSuccessful()) {
        String result = response.body().string();
        Log.d("hoyouly", "okhttp : pic  " + result);
    } else {
        Log.e("hoyouly", "okhttp-post-err:" + response.code());
    }
} catch (Exception ex) {
    ex.printStackTrace();
    Log.d("hoyouly", " okhttp   execute error " + ex.getMessage());
}
```
同样可以更加得到的字符串拼接成一个 URL ，查看到这个图片的。说明也上传成功了。

# 使用 Retrofit 框架。

之前也用过 Retrofit 做过上传图片。首先想到的就是 @Part / @PartMap 啊，因为这两个携带的数据类型更加丰富，包括数据流，所以适用于有文件上传的场景
因为上传的接口中不需要什么参数，直接使用 @Part 注解，然后使用 MultipartBody.Part 类型作为文件的参数，还得使用到 Multipart 这个注解。
具体详情可以参考 [扫盲系列 - Retrofit 基本用法 ](../../../../2018/03/20/Retrofit/#multipart)。
所以就把接口定义成如下了。
```java
@Multipart
@POST("api/uploadimg")
Observable<String> updatePic(@Part MultipartBody.Part imgs);
```
然后开始上传数据吧。
```java
File file = new File(path);
RequestBody requestFile = RequestBody.create(null, file);
MultipartBody.Part part = MultipartBody.Part.create(requestFile);
return httpServer.updatePic(requestFile)
   .retry(2)// 失败重连
   .subscribeOn(Schedulers.io());
```
我连 content_type 都设置为 null ，就是为了保证上传的全部是图片的二进制。
也返回了一个字符串，可是发现拼接后发现图片打不开。郁闷了，哪里出问题了呢？


## 发现原因
可是使用 Retrofit 就怎么也不成功，虽然也返回了字符串，可是通过这个返回的字符串，去查看这个图片，就怎么也显示不出来。
到底是哪里出问题了呢？？

然后通过 Charles 抓包看看数据吧,看看 发送的数据是啥吧

<img src="../../../../images/xutils_content.png" alt="xutils_content" width="300" height="300" align="left" />
<img src="../../../../images/retrofit_content.png" alt="retrofit_content" width="300" height="300" align="center" />

傻眼了，这能看出来啥啊。一堆乱码，也对，发送的是二进制，能看出来才怪呢，那该怎么查呢？
不过我注意到一个事情，
1. 通过 Retortfit ，打开查看 content 的时候，很卡，可以说 Charles 都卡死了，但是查看xUtils/OkHttp 访问的 content 的时候，很顺溜的就打开了，并且不会卡死 Charles
2. 为啥 retrofit 的 content 前面有一小片是空白啊？？

可是到现在我也不清楚为啥，然后就看了另外一列， overview 发现了一点端倪：同一张图片, body 中的字节竟然不一样，如下图


<img src="../../../../images/xutils_reponse.png" alt="xutils_reponse" width="300" height="300" align="left" />
<img src="../../../../images/retrofit_reponse.png" alt="retrofit_reponse" width="300" height="300" align="center" />

retrofit 上传图片的时候， body 中字节数竟然比 okhttp 和 xutils 要多一些，这些到底是啥呢，什么时候添加上去呢？ 这是一个值得思考的问题。

因为使用的 Multipart 注解，最后封装成了一个 MultipartBody ，然后在这里面，发现了一些端倪。其实还是通过 断点一点点查出来的。
```java
//MultipartBody.java
private long writeOrCountBytes( @Nullable BufferedSink sink, boolean countBytes) throws IOException {
    long byteCount = 0L;
    Buffer byteCountBuffer = null;
    if (countBytes) {
      sink = byteCountBuffer = new Buffer();
    }
    for (int p = 0, partCount = parts.size(); p < partCount; p++) {
      Part part = parts.get(p);
      Headers headers = part.headers;
      RequestBody body = part.body;

      sink.write(DASHDASH);
      sink.write(boundary);
      sink.write(CRLF);

      if (headers != null) {
        for (int h = 0, headerCount = headers.size(); h < headerCount; h++) {
          sink.writeUtf8(headers.name(h))
              .write(COLONSPACE)
              .writeUtf8(headers.value(h))
              .write(CRLF);
        }
      }

      MediaType contentType = body.contentType();
      if (contentType != null) {
        sink.writeUtf8("Content-Type: ")
            .writeUtf8(contentType.toString())
            .write(CRLF);
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        sink.writeUtf8("Content-Length: ")
            .writeDecimalLong(contentLength)
            .write(CRLF);
      } else if (countBytes) {
        // We can't measure the body's size without the sizes of its components.
        byteCountBuffer.clear();
        return -1L;
      }
      sink.write(CRLF);
      if (countBytes) {
        byteCount += contentLength;
      } else {
        body.writeTo(sink);
      }
      sink.write(CRLF);
    }
    sink.write(DASHDASH);
    sink.write(boundary);
    sink.write(DASHDASH);
    sink.write(CRLF);
    if (countBytes) {
      byteCount += byteCountBuffer.size();
      byteCountBuffer.clear();
    }
    return byteCount;
  }

private static final byte[] COLONSPACE = {':', ' '};
private static final byte[] CRLF = {'\r', '\n'};
private static final byte[] DASHDASH = {'-', '-'};
private final ByteString boundary;  

```

尽管 headers 为 null ， contentType 中也为 null ，可是在开始之前，还是写入了下面一段文案
```java
sink.write(DASHDASH);
sink.write(boundary);
sink.write(CRLF);
```
并且在结束后，又追加了下面这一段文案
```java
sink.write(DASHDASH);
sink.write(boundary);
sink.write(DASHDASH);
sink.write(CRLF);
```
这些，这就导致上传服务器的时候，多上传了一部分，然后服务器根据这些流写入一个文件，就不是一张图片了，所以打不开

原因找到了，可是怎么处理呢？？
## 试图解决
刚开始我想着看能不能绕过这个方法，直接把文件转换成流。
首先想到的就是通过自定义  Converter.Factory，然后在 requestBodyConverter() 中返回一个新的 Converter ，在这个 Converter 中通过 cover() 方法，把文件转成一个 ResponseBody ,详情 [  Retrofit 2.0文件上传  ](https://blog.csdn.net/jdsjlzx/article/details/51588617)。

可是发现不行。因为不管你使用哪种方式，都会执行到 RequestBuilder 的构造函数用来创造一个 RequestBuilder.
```java
RequestBuilder(String method, HttpUrl baseUrl , @Nullable String relativeUrl,
    @Nullable Headers headers, @Nullable MediaType contentType, boolean hasBody ,
 boolean isFormEncoded , boolean isMultipart) {
  ...
  if (isFormEncoded) {
    // Will be set to 'body' in 'build'.
    formBuilder = new FormBody.Builder();
  } else if (isMultipart) {
    // Will be set to 'body' in 'build'.
    multipartBuilder = new MultipartBody.Builder();
    multipartBuilder.setType(MultipartBody.FORM);
  }
}
```
而我们使用了 Multipart 注解， isMultipart 就是 true , Builder() 中就会根据 UUID 随机生成 一个 boundary ，这个 boundary 就是前面要写进去的。

```java
public Builder() {
     this(UUID.randomUUID().toString());
   }

public Builder(String boundary) {
 this.boundary = ByteString.encodeUtf8(boundary);
}
```
这怎么也绕不过去了啊。陷入困境了。

## 再次尝试

可是为啥 OkHttp 单独使用就行呢，难道 Retort 封装了 Okhttp ,把一些功能阉割了吗，不应该吧。然后我又再次看了看 代码。
发现 OKHttp 是直接把 RequestBody 提交上去了，
```java
RequestBody requestFile = RequestBody.create(MediaType.parse("image/jpg"), file);
//创建Request
final Request request = new Request.Builder().url(BASE_URL +"uploadimg").post(requestFile).build();
OkHttpClient mOkHttpClient = new OkHttpClient();
final Call call = mOkHttpClient.newBuilder().writeTimeotTimeUnit.SECONDS).build().newCall(request);
```
那我也试试咯。参数改成 RequestBody 。如下
```java
@Multipart
@POST("/api/uploadimg")
Observable<String> updatePic(@Part() RequestBody imgs);
```
可是又碰了一鼻子灰。因为直接使用 RequestBody 当做参数的话，那会报另外一个错误的    
`@Part annotation must supply a name or use MultipartBody.Part parameter type.`  
<font color="#ff000" >@Part 注解，可以使用任何对象作为参数，但是必须设置一个 name ,MultipartBody.Part 类型除外。</font>

可是如果我设置了 name ,我不知道参数是啥啊，写个先瞎写一个 body 字符串吧。
```java
@Multipart
@POST("/api/uploadimg")
Observable<String> updatePic(@Part("body") RequestBody imgs);
```
可还行不行。
因为
1. 这个 body 字符串还会写到二进制流中的啊
2. 虽然你设置了参数类型是 ResponseBody ,可是最后还是转换成了 MultipartBody.Part 类型。

```java
//ServiceMethod.java
private ParameterHandler<?> parseParameterAnnotation(int p, Type type , Annotation[] annotations, Annotation annotation) {
  ...
} else if (annotation instanceof Part) {// 设置的是 Part 注解。
      if (!isMultipart) {
        throw parameterError(p, "@Part parameters can only be used with multipart encoding.");
      }
      Part part = (Part) annotation;
      gotPart = true;

      String partName = part.value();
      Class<?> rawParameterType = Utils.getRawType(type);
      if (partName.isEmpty()) {//设置注解的 name 是空
        ...
        } else if (MultipartBody.Part.class.isAssignableFrom(rawParameterType)) {
          return ParameterHandler.RawPart.INSTANCE;
        } else {//熟悉吧，之前的报错就来自这里
          throw parameterError(p,"@Part annotation must supply a name or use MultipartBody.Part parameter type.");
        }
      } else {
        Headers headers = Headers.of("Content-Disposition", "form-data; name=\""
                + partName + "\"","Content-Transfer-Encoding", part.encoding());
        ...
          Converter<?, RequestBody> converter =
              retrofit.requestBodyConverter(type, annotations , methodAnnotations);
          return new ParameterHandler.Part<>(headers, converter);
        }
      }
}
```
1. partName 就是使用 Part 注解的时候，设置的 name 为空，则会判断类型是不是 MultipartBody.Part ，不是的话，那就出错了，之前的报错就是在这里
2. partName 不为空，则会转成成 ParameterHandler.Part，并且还添加了 headers ，这就导致了通过这种方式上传的图片， body 的值更大些，这更不合理啊。
![添加图片](../../../../images/requestbody.png)

## 柳暗花明
难道真的就无解了吗？ 感觉陷入了死局啊。可是还是不死心，主要是闲得无聊。然后我就又看了看 Postman ,发现了一个我之前没注意到的地方。

![添加图片](../../../../images/postman.png)

body 类型是 Binary ，之前一直以为是 raw , 其实这个接口本质就是上传二进制，并且前面接口也说了，上传二进制流，并不是图片，上传图片的本质也是二进制流

所以可以换个思路，从 Binary 下手试试。然后我就继续查找。Google + baidu 搜索 retrofit 上传 二进制。然后就看到了然我豁然开朗的一篇文章。    
[ retrofit的请求方式总概(包含 binary 上传文件方式) ](https://www.jianshu.com/p/02e62feb060d)  

主要是这段文字

![添加图片](../../../../images/body_binary.png)

谁说 上传图片必须使用 @Part或者 @PartMap 啊，@Body 不行吗？

然后我就改了一下。
```java
// @Multipart
@POST("api/uploadimg")
Observable<String> updatePic(@Body RequestBody imgs);
```
记得把上面的 @Multipart 注解注释掉，因为他们两个不能一块使用

然后再次测试一把。竟然成功了，可以了， ok 了。卧槽，操。。。

踏破铁鞋无觅处，得来全不费工夫。

一直在 Part 注解上纠结，差点要改 Retort 源码试试了，竟然使用常用的 Body 就可以处理。

# 总结
看来有时候经验可以帮人，也可以害人啊。害得老子昨晚忙到快一点了，也没查出原因。


---
搬运地址：    
[Retrofit 2.0文件上传](https://blog.csdn.net/jdsjlzx/article/details/51588617)

[retrofit的请求方式总概(包含 binary 上传文件方式)](https://www.jianshu.com/p/02e62feb060d)
