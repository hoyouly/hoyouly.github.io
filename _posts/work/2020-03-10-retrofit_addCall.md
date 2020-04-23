---
layout: post
title: 工作填坑 - Retrofit 中对 addCallAdapterFactory() 的理解
category: 工作填坑
tags: Retrofit   
---
<!-- * content -->
<!-- {:toc} -->

使用 Retrofit 的时候， addConverterFactory() 和 addCallAdapterFactory() 好像是必备的，

在上一篇中的 我们知道了 addConverterFactory() 其实就是为了处理请求数据和响应数据的，但是 addCallAdapterFactory() 这个是干嘛的呢，今天就来看一下。

addCallAdapterFactory() 官方给的解释是

`Add a call adapter factory for supporting service method return types other than Call.`

我的理解：根据返回类型，可以确定使用哪种适配器进行请求网络。看来和网络请求有关。

我们知道。在调用接口后，不管是得到一个 Observer ,还是得到一个 Call 对象，这个是还并没有真正的请求数据，真正执行访问网络，请求数据的是在下一步，例如
```java
// observable 请求网络
AbsObserver baseObserver = observable.subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribeWith(observer);

//Call 对象 请求网络
call.execute();
```

我们先看看 Call 是 怎么来处理的。


看过 Retofit 代码的人，应该也都知道，执行某个接口的时候，最终会通过动态代理，执行到
```java
  ServiceMethod<Object, Object> serviceMethod =(ServiceMethod<Object, Object>) loadServiceMethod(method);
  OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
  return serviceMethod.adapt(okHttpCall);
```
如果没看过，可以参考 [ 源码分析 - Retrofit ](../../../../2018/03/26/Retrofit-Source-Code-Analysis/)

我们就来看看 serviceMethod.adapt(okHttpCall) 这行代码的奥秘

```java
//ServiceMethod.java
T adapt(Call<R> call) {
    return callAdapter.adapt(call) ;
}
```
这个 callAdapter 是什么时候赋值的吗？ 是在 ServiceMethod.build() 中，有 callAdapter = createCallAdapter() 就是 callAdapter
详情 可以看 [ 工作填坑 - Retrofit 同时支持对 Json 格式和原始数据 ](../../../../2020/03/09/retrofit_json_rawdata/) 这篇文章

接着看  createCallAdapter()

```java
//ServiceMethod.java
private CallAdapter<T, R> createCallAdapter() {
  ...
  return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
  ...
}

//retrofit.java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
 return nextCallAdapter(null, returnType , annotations);
}

public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType ,
     Annotation[] annotations) {
  ...
   int start = callAdapterFactories.indexOf(skipPast) + 1;
   for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
     CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations , this);
     if (adapter != null) {
       return adapter;
     }
   }
  ...
   throw new IllegalArgumentException(builder.toString());
}
```
这个流程和 CallConver 差不多，想要添加一个新的 Factory ，还是要自定义一个 Factory ，然后返回一个 CallAdapter ,由对应的 CallAdapter ，才能返回对应的 Call 对象 ，根据对应的 Call 对象，然后请求网络。

下面是对 CallAdapter 方法的解释。

```java
public interface CallAdapter<R, T> {
  //返回此适配器将 HTTP 响应正文转换为 Java 时使用的值类型对象。
	//例如, Call <Repo>的响应类型是 Repo 。 这个类型用于准备传递给 adapt 的 call 。
  Type responseType();

//这个方法很重要，会在 Retrofit 的create(final Class<T> service)方法中被调用，
//并把 okHttpCall 即 Call 传进来serviceMethod.adapt(okHttpCall)，接下来就可以对 Call 进行操作了
  T adapt(Call<R> call);

//CallAdapter工厂， retrofit 默认的 DefaultCallAdapterFactory 其中不对 Call 做处理，是直接返回 Call 。
abstract class Factory {
// 在这个方法中判断是否是我们支持的类型， returnType 即Call<Requestbody>和`Observable<Requestbody>`
// RxJavaCallAdapterFactory 就是判断 returnType 是不是 Observable<?> 类型
// 不支持时返回null
//返回值必须是 Custom 并且带有泛型（参数类型），根据 APIService 接口中的方法返回值，确定 returnType ，如： CustomCall<String> getCategories();，那确定 returnType 就是CustomCall<String>
public abstract @Nullable CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
    Retrofit retrofit);

 // 用于获取泛型的参数 如 Call<Requestbody> 中 Requestbody
protected static Type getParameterUpperBound(int index, ParameterizedType type) {
  return Utils.getParameterUpperBound(index, type);
}

 // 用于获取泛型的原始类型 如 Call<Requestbody> 中的 Call
//  如Call<Requestbody>拿到的原始类型就是Call
protected static Class<?> getRawType(Type type) {
  return Utils.getRawType(type);
      }
   }
 }
```
