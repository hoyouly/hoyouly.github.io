---
layout: post
title: 工作填坑 - Retrofit 同时支持对 Json 格式和原始数据
category: 工作填坑
tags: Retrofit  
---
* content
{:toc}

最近在封装HTTP请求的时候，发现了一个挺悲催的事情，服务端返回的数据，有时候是json 格式，有时候又不是json格式，身为乙方的我，又没办法要求甲方去修改成统一的json格式，可是如果统一返回原始数据，又感觉要做太多的无用代码，毕竟使用的是retrofit + rxjava+ OKhttp ,得充分发挥他们三个的作用才行。

然后就想着自己封装的框架能不能同时兼容Json格式和非Json格式呢？

感觉应该是可以的，因为我们通常使用Retrofit 解析json格式是按照下面的方式，
```java
retrofit = new Retrofit.Builder()
            .baseUrl(sBaseUrl)
            .addConverterFactory(GsonConverterFactory.create(buildGson()))
            .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
            .client(httpClientBuilder.build())
            .build();
```
里面都是 addConverterFactory() 和 addCallAdapterFactory()，是add而不是set,并且通过看源码也能看出来，是把Factory 添加到一个集合里面，所以，应该是可以支持多次add的，

```java
public Builder addConverterFactory(Converter.Factory factory) {
			converterFactories.add(checkNotNull(factory, "factory == null"));
			return this;
		}
```

那么就需要研究一下，在什么情况下，使用 GsonConverterFactory 去解析，什么时候，使用另外一种方式解析数据。

在解决这个问题之前，我们就需要了解 addConverterFactory()到底是干嘛的
它主要是对数据转化用的，请求网络的数据，在这里转换成我们需要的数据类型，

## addConverterFactory()

### 源码解析
闲话少说，看代码咯。

看过Retofit代码的人，应该也都知道，执行某个接口的时候，最终会通过动态代理，执行到
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
因为首次调用该接口的时候，serviceMethodCache 没有该方法，所以会执行到build()方法,创建一个，然后添加到serviceMethodCache里面。
我们继续查看 build()方法

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
其他的我们不比关心，只看着几行代码

```java
private Converter<ResponseBody, T> createResponseConverter() {
    ...
    return retrofit.responseBodyConverter(responseType, annotations);
    ...
}

//Retrofit.java
public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[]annotations) {
    return nextResponseBodyConverter(null, type, annotations);
}

 public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
  ...
    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
  ...
    throw new IllegalArgumentException(builder.toString());
  }

```
流程就是
1. 先通过 addConverterFactory()往 converterFactories 中添加Factory
2. 在调用该接口的时候，第一次缓存中没有，通过ServiceMethod build()方法创建一个，在创建的过程中
3. 需要设置要通过 createResponseConverter()创建一个 Converter
4. createCallAdapter()其实就是遍历 callAdapterFactories 这个集合，找到第一个匹配的就直接返回，不再遍历，然后返回。就是改接口要执行的 数据转换的类。遍历结束没找到合适的，就直接跑出异常了。

所以，如果我们想要定义某个接口执行其他的数据转换，那么就要通过responseBodyConverter(),返回一个Converter即可，并且是第一个.

可以先add我们定义的返回原始数据的 Factory，即 RawConverterFactory , 然后在add Json格式的Factory.即 GsonConverterFactory

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
如果返回的类型是Strig,则认为是需要原始数据，则直接返回一个 RawResponseBodyConverter ，这个RawResponseBodyConverter 是实现Converter的，在convert()中直接返回原始数据。

### RawResponseBodyConverter

requestBodyConverter()的转换，还和Json一样，请求数据格式是json样式，如果要是其他样式，可以在这里进行处理

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
因为 RawConverterFactory是首先add的，并且返回的参数是String类型，所以就使用 RawConverterFactory 来进行数据的处理

### RawRequestBodyConverter

因为数据请求最终是通过json 串的形式，所以需要把实体类转换成json 串，
原本可以 使用GsonRequestBodyConverter来处理的，可是谁知道
```java
final class GsonRequestBodyConverter<T> implements Converter<T, RequestBody> {
```
这个类不是public的，所以没办法直接用，只好重新copy 一份。叫 RawRequestBodyConverter。
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
这样，当我们定义返回类型是String的时候，就返回的是原始数据，如果是一个其他实体类的话，返回的就是该对象。

```java
@Headers("Content-type:application/json")
@POST("/sinoapi/updatepwd")
Observable<ModifyPwdResult> updatepwd(@Body ReqModifyPwd ben);  


@Multipart
@POST("/sinoapi/uploadimg")
Observable<String> updatePic(@PartMap Map<String, RequestBody> requestBodyMap, @Part MultipartBody.Part imgs);

```
这样就齐活了，记得先添加 RawConverterFactory这个Factory，再添加GsonConverterFactory 才行。

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

---
搬运地址：    
