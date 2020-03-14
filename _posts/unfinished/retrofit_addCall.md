其实 调用接口后，不管是得到一个Observer,还是得到一个Call 对象，这个是还并没有真正的请求数据，真正请求数据的是在下一步，才是真正的执行访问网络。例如
```java
AbsObserver baseObserver = observable.subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribeWith(observer);
```
或者 是
```java
call.execute();
```
所以 我们还需要定制一个 CallAdapter ,用来访问网络，执行 addCallAdapterFactory()
## addCallAdapterFactory()

addCallAdapterFactory()
解释的是  `Add a call adapter factory for supporting service method return types other than Call.`,我的理解是，根据返回类型，可以确定使用哪种适配器进行请求网络。

### 源码分析
在 serviceMethod.adapt(okHttpCall) 中，

```java
//ServiceMethod.java
T adapt(Call<R> call) {
    return callAdapter.adapt(call);
}
```
这个 callAdapter  是什么时候赋值的吗，其实记得之前 ServiceMethod.build()中，有 callAdapter = createCallAdapter(); 就是callAdapter

```java
//ServiceMethod.java
private CallAdapter<T, R> createCallAdapter() {
    ...
    //noinspection unchecked
    return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
		...
  }

//retrofit.java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
   return nextCallAdapter(null, returnType, annotations);
 }

public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
       Annotation[] annotations) {
    ...
     int start = callAdapterFactories.indexOf(skipPast) + 1;
     for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
       CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
       if (adapter != null) {
         return adapter;
       }
     }
    ...
     throw new IllegalArgumentException(builder.toString());
   }
```
这个流程和CallConver差不多，关键还是要定义一个Factory，然后返回一个CallAdapter,由对应的CallAdapter，才能返回对应的Call对象 ，根据对应的Call对象，然后请求网络。

```java
public interface CallAdapter<R, T> {
  //返回此适配器将HTTP响应正文转换为Java时使用的值类型对象。
	//例如, Call <Repo>的响应类型是Repo。 这个类型用于准备传递给adapt的call。
  Type responseType();

//这个方法很重要，会在 Retrofit的create(final Class<T> service)方法中被调用，
//并把okHttpCall即Call传进来serviceMethod.callAdapter.adapt(okHttpCall)，接下来就可以对Call进行操作了
  T adapt(Call<R> call);

//CallAdapter工厂，retrofit默认的DefaultCallAdapterFactory其中不对Call做处理，是直接返回Call。
abstract class Factory {
// 在这个方法中判断是否是我们支持的类型，returnType 即Call<Requestbody>和`Observable<Requestbody>`
// RxJavaCallAdapterFactory 就是判断returnType是不是Observable<?> 类型
// 不支持时返回null
//返回值必须是Custom并且带有泛型（参数类型），根据APIService接口中的方法返回值，确定returnType，如： CustomCall<String> getCategories();，那确定returnType就是CustomCall<String>
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



下面贴上 CallAdapter 各个方法的介绍

### RawCall
```java
public class RawCall<R> {

    public final Call<R> call;

    public RawCall(Call<R> call) {
        this.call = call;
    }
    /**
     * 异步请求数据
     *
     * @param callback
     */
    public void enqueue(Callback<R> callback) {
        call.enqueue(callback);
    }
    /**
     * 同步请求数据
     *
     * @return
     * @throws IOException
     */
    public Response<R> execute() throws IOException {
        return call.execute();
    }
}
```
这个很简单，其实就是对 Call的一种代理。

然后在RawCallAdapterFactory 中的 get()方法中就可以进行判断，什么情况下是执行到这个逻辑，
如果返回的原始类型是 RawCall，并且 真实类型 是 ParameterizedType，则认为要使用RawCallAdapter来处理

get()方法就是判断是否是我们支持的类型，RawCallAdapterFactory 支持的就是 RawCall类型的Call

### RawCallAdapterFactory

```java
public class RawCallAdapterFactory extends CallAdapter.Factory {

    public static RawCallAdapterFactory create() {
        return new RawCallAdapterFactory();
    }

    @Nullable
    @Override
    public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
        //获取原始类型
        Class<?> rawType = getRawType(returnType);
        //返回值必须是RawCall并且带有泛型（参数类型），根据APIService接口中的方法返回值，确定returnType
        //如 RawCall<String> getCategories();，那确定returnType就是RawCall<String>
        if (rawType == RawCall.class && returnType instanceof ParameterizedType) {
            Type callReturnType = getParameterUpperBound(0, (ParameterizedType) returnType);
            return new RawCallAdapter(callReturnType);
        }
        return null;
    }

}
```

### RawCallAdapter

```java
public class RawCallAdapter implements CallAdapter<R, RawCall<R>> {

    private final Type responseType;

    public RawCallAdapter(Type responseType) {
        this.responseType = responseType;
    }
    /**
     * 真正数据的类型，如Call<T> 中的 T，这个T会作为  Converter.Factory.responseConverter的第一个参数
     * @return
     */
    @Override
    public Type responseType() {
        return responseType;
    }

    @Override
    public RawCall<R> adapt(Call<R> call) {
        return new RawCall<>(call);
    }
}
```

所以我们在执行的RawCall的excute(),其实就是在执行call(这个就是OKHTTP中的call ) 的execute()方法。

```java
@Multipart
@POST("/sinoapi/uploadimg")
RawCall<String> updatePic(@PartMap Map<String, RequestBody> requestBodyMap, @Part MultipartBody.Part imgs);


RawCall<String> rawCall = apiServer.updatePic(generateRequestBody(params), part);
Response<String> response = requestRawData(rawCall);

public <R> Response<R> requestRawData(RawCall<R> call) throws IOException {
			 return call.execute();
	 }
```
