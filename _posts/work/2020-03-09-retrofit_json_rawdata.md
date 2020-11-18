---
layout: post
title: 工作填坑 - Retrofit 同时支持对 Json 格式和原始数据
category: 工作填坑
tags: Retrofit  
---
<!-- * content -->
<!-- {:toc} -->

最近在封装 HTTP 请求的时候，发现了一个挺悲催的事情，服务端返回的数据，有时候是 json 格式，有时候又不是 json 格式，身为乙方的我，又没办法要求甲方去修改成统一的 json 格式，可是如果统一返回原始数据，又感觉要做太多的无用代码，毕竟使用的是 retrofit + rxjava+ OKhttp ,得充分发挥他们三个的作用才行。

然后就想着自己封装的框架能不能同时兼容 Json 格式和非 Json 格式呢？

感觉应该是可以的，因为我们通常使用 Retrofit 解析 json 格式是按照下面的方式，
```java
retrofit = new Retrofit.Builder()
            .baseUrl(sBaseUrl)
            .addConverterFactory(GsonConverterFactory.create(buildGson()))
            .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
            .client(httpClientBuilder.build())
            .build();
```
里面都是 addConverterFactory() 和 addCallAdapterFactory() ，是 add 而不是 set ,并且通过看源码也能看出来，是把 Factory 添加到一个集合里面，所以，应该是可以支持多次 add 的，

```java
public Builder addConverterFactory(Converter.Factory factory) {
      converterFactories.add(checkNotNull(factory, "factory == null"));
      return this;
    }
```

那么就需要研究一下，在什么情况下，使用 GsonConverterFactory 去解析，什么时候，使用另外一种方式解析数据。

在解决这个问题之前，我们就需要了解 addConverterFactory() 到底是干嘛的
它主要是对数据转化用的，请求网络的数据，在这里转换成我们需要的数据类型，

## addConverterFactory()

### 源码解析
闲话少说，看代码咯。

看过 Retofit 代码的人，应该也都知道，执行某个接口的时候，最终会通过动态代理，执行到
```java
  ServiceMethod<Object, Object> serviceMethod =(ServiceMethod<Object, Object>) loadServiceMethod(method);
  OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
  return serviceMethod.adapt(okHttpCall);
```
  如果没看过，可以参考 [ 源码分析 - Retrofit ](../../../../2018/03/26/Retrofit-Source-Code-Analysis/)

这三行代码，我们先看 loadServiceMethod()

```java
ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```
因为首次调用该接口的时候， serviceMethodCache 没有该方法，所以会执行到 build() 方法,创建一个，然后添加到 serviceMethodCache 里面。[ Retrofit 中对 addCallAdapterFactory() 的理解 ](../../../../2020/03/10/retrofit_addCall/)
我们继续查看 build() 方法

```java
public ServiceMethod build() {
     callAdapter = createCallAdapter();
     responseType = callAdapter.responseType();
     ...
     responseConverter = createResponseConverter();
    ...
     return new ServiceMethod<>(this);
   }
```
其他的我们不比关心，只看着几行代码。  callAdapter = createCallAdapter() 涉及到 addCallAdapterFactory() ，会在另外一篇文章中分析

```java
private Converter<ResponseBody, T> createResponseConverter() {
    ...
    return retrofit.responseBodyConverter(responseType, annotations);
    ...
}

//Retrofit.java
public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[]annotations) {
    return nextResponseBodyConverter(null, type , annotations);
}

public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type , Annotation[] annotations) {
...
  int start = converterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = converterFactories.size(); i < count; i++) {
    //这行代码是关键
    Converter<ResponseBody, ?> converter =
        converterFactories.get(i).responseBodyConverter(type, annotations , this);
    if (converter != null) {
      return (Converter<ResponseBody, T>) converter;
    }
  }
...
  throw new IllegalArgumentException(builder.toString());
}

```
流程就是
1. 先通过 addConverterFactory() 往 converterFactories 中添加Factory
2. 在调用该接口的时候，第一次缓存中没有，通过 ServiceMethod build() 方法创建一个
3. 在创建的过程中，通过 createResponseConverter() 创建一个 Converter
4. createResponseConverter() 其实就是遍历 converterFactories 这个集合，找到第一个匹配 Converter 直接返回，不再遍历。匹配的依据就是根据 Converter.Factory中的 responseBodyConverter() 这个方法
5. 遍历结束没找到合适的，就直接抛出异常。

所以，<font color="#ff000" >如果我们想要定义一种新的数据转换的类，分两步。</font>
1. 继承  Converter.Factory 这个抽象类，然后实现 responseBodyConverter() , 返回一个 Converter
2. 把这个类添加到 converterFactories 这个集合的首位。

## 自定义数据转换类
主要包括三个， RawConverterFactory 和 RawResponseBodyConverter 以及 RawRequestBodyConverter
### RawConverterFactory
```java
public final class RawConverterFactory extends Converter.Factory {
    public static RawConverterFactory create() {
        return create(new Gson());
    }

    @SuppressWarnings("ConstantConditions") // Guarding public API nullability.
    public static RawConverterFactory create(Gson gson) {
        if (gson == null) {
            throw new NullPointerException("gson == null");
        }
        return new RawConverterFactory(gson);
    }

    private final Gson gson;

    private RawConverterFactory(Gson gson) {
        this.gson = gson;
    }

    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type responseType, Annotation[] annotations,
        Retrofit retrofit) {
        if (responseType == String.class) {//返回类型是字符串，
            return new RawResponseBodyConverter<>();
        } else {
            return null;
        }
    }

    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type,
        Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
        TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
        return new RawRequestBodyConverter<>(gson, adapter);
    }
}

```
* responseBodyConverter() 中进行判断，如果 responseTyp 的类型是 String ,则认为是需要原始数据，则直接返回一个 RawResponseBodyConverter ，
* 因为请求数据格式是 json 样式，所以 requestBodyConverter() 实现的和 GsonRequestBodyConverter 类似，如果请求格式改了，例如 xml 的话，可以在这里进行处理

### RawResponseBodyConverter
这个 RawResponseBodyConverter 是 Converter 的实现类，在 convert() 中直接返回原始数据。

```java

final class RawResponseBodyConverter<T> implements Converter<ResponseBody, T> {
    RawResponseBodyConverter() {
    }
    @Override
    public T convert(ResponseBody value) throws IOException {
        String response = value.string();
        return (T) response;
    }
}
```
因为 RawConverterFactory 是首先 add 的，并且返回的参数是 String 类型，所以就使用 RawConverterFactory 来进行数据的处理

### RawRequestBodyConverter
这里没有直接使用 GsonRequestBodyConverter 而是重新定义了一个 RawRequestBodyConverter ，是因为 GsonRequestBodyConverter 不是 public 。
```java
final class GsonRequestBodyConverter<T> implements Converter<T, RequestBody> {}
```
不能直接使用。只好重新 copy 一份。
```java
final class RawRequestBodyConverter<T> implements Converter<T, RequestBody> {
    private static final MediaType MEDIA_TYPE = MediaType.parse("application/json; charset=UTF-8");
    private static final Charset UTF_8 = Charset.forName("UTF-8");

    private final Gson gson;
    private final TypeAdapter<T> adapter;

    RawRequestBodyConverter(Gson gson, TypeAdapter<T> adapter) {
        this.gson = gson;
        this.adapter = adapter;
    }

    @Override
    public RequestBody convert(T value) throws IOException {
        Buffer buffer = new Buffer();
        Writer writer = new OutputStreamWriter(buffer.outputStream(), UTF_8);
        JsonWriter jsonWriter = gson.newJsonWriter(writer);
        adapter.write(jsonWriter, value);
        jsonWriter.close();
        return RequestBody.create(MEDIA_TYPE, buffer.readByteString());
    }
}

```
## 使用
当我们定义返回类型是 String 的时候，就返回的是原始数据，如果是一个其他实体类的话，返回的就是该对象。

```java
// 得到 Json 格式数据，最终通过 GsonConverterFactory 来处理
@Headers("Content-type:application/json")
@POST("/sinoapi/updatepwd")
Observable<ModifyPwdResult> updatepwd(@Body ReqModifyPwd ben);  


//得到原始数据，最终通过 RawConverterFactory 来处理
@Multipart
@POST("/sinoapi/uploadimg")
Observable<String> updatePic(@PartMap Map<String, RequestBody> requestBodyMap, @Part MultipartBody.Part imgs);
```
添加 RawConverterFactory 这个 Factory ，再添加  。
```java

retrofit = new Retrofit.Builder()
    .baseUrl(sBaseUrl)
    .addConverterFactory(RawConverterFactory.create(buildGson()))
    .addConverterFactory(GsonConverterFactory.create(buildGson()))
    //支持RxJava2
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
    .client(httpClientBuilder.build())
    .build();
```

齐活。

---
搬运地址：    
