---
layout: post
title: Retorfit 源码分析
category: 技术
tags: Android Retrofit 源码分析
---

* content
{:toc}
# 创建Retrofit 的实例

```java
Retrofit retrofit=new Retrofit.Builder()//
				.baseUrl("http://fanyi.youdao.com/")//
				.addConverterFactory(GsonConverterFactory.create())//
				.addCallAdapterFactory(RxJavaCallAdapterFactory.create())//
				.build();
```
## new Retrofit.Builder()
 通过Builder 模式建立Retrofit实例，既然是链式结构，那就一步步往下看吧，先看 `new Retrofit.Builder()`
```java
//Builder.java
public Builder() {
    this(Platform.get());
  }
  Builder(Platform platform) {
      this.platform = platform;
      converterFactories.add(new BuiltInConverters());
    }

// Platform.java
    class Platform {
      private static final Platform PLATFORM = findPlatform();

      static Platform get() {
        return PLATFORM;
      }

      private static Platform findPlatform() {
        try {
          Class.forName("android.os.Build");
          if (Build.VERSION.SDK_INT != 0) {
            return new Android();
          }
        } catch (ClassNotFoundException ignored) {
        }
        try {
          Class.forName("java.util.Optional");
          return new Java8();
        } catch (ClassNotFoundException ignored) {
        }
        try {
          Class.forName("org.robovm.apple.foundation.NSObject");
          return new IOS();
        } catch (ClassNotFoundException ignored) {
        }
        return new Platform();
      }
      .........
}
```
Builder的构造函数里面，通过Platform.get()获得当前的设备平台信息，并且把内置的转换器工厂（BuiltInConverters）加添到工厂集合中，它的主要作用就是当使用多种Converters的时候能够正确的引导并找到可以消耗该类型的转化器。

## baseUrl()
接着看baseUrl()吧
```java
public Builder baseUrl(String baseUrl) {
     checkNotNull(baseUrl, "baseUrl == null");
     HttpUrl httpUrl = HttpUrl.parse(baseUrl);
     if (httpUrl == null) {
       throw new IllegalArgumentException("Illegal URL: " + baseUrl);
     }
     return baseUrl(httpUrl);
   }
```
通过源码可知，传递的字符串baseurl,会通过HttpUrl的parse()转换成一个HttpUrl对象，然后调用重载方法，baseUrl(httpUrl),

```java
private HttpUrl baseUrl;

public Builder baseUrl(HttpUrl baseUrl) {
    checkNotNull(baseUrl, "baseUrl == null");
    List<String> pathSegments = baseUrl.pathSegments();
    if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
      throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
    }
    this.baseUrl = baseUrl;
    return this;
  }

```
这里面做了简单处理后，然后保存到了全局变量baseUrl中，

## addConverterFactory()
然后看addConverterFactory()方法，接受一个转换器工厂，它主要是对数据转化用的，请网络请求获取的数据，将会在这里被转化成我们所需要的数据类型，比如通过Gson将json数据转化成对象类型。
```java
public Builder addConverterFactory(Converter.Factory factory) {
     converterFactories.add(checkNotNull(factory, "factory == null"));
     return this;
   }
```
## addCallAdapterFactory()
该方法主要是针对Call转换了，比如对Rxjava的支持，从返回的call对象转化为Observable对象。
```java
public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
      adapterFactories.add(checkNotNull(factory, "factory == null"));
      return this;
    }

```
当然，除了这几个方法之外，还有其他几个比较常见的，
## client()
```java
public Builder client(OkHttpClient client) {
     return callFactory(checkNotNull(client, "client == null"));
   }
```
这个是可选的，如果没有传入则就默认为OkHttpClient，在这里可以对OkHttpClient做一些操作，比如添加拦截器打印log等，然后执行了callFactory()方法，因为OkHttpClient 实现了Call.Factory
## callFactory()
```java
public Builder callFactory(okhttp3.Call.Factory factory) {
     this.callFactory = checkNotNull(factory, "factory == null");
     return this;
   }
```
## callbackExecutor()
看可以得知应该是回调执行者，也就是Call对象从网络服务获取数据之后转换到UI主线程中。
```java
public Builder callbackExecutor(Executor executor) {
     this.callbackExecutor = checkNotNull(executor, "executor == null");
     return this;
   }
```
## validateEagerly()
```java
public Builder validateEagerly(boolean validateEagerly) {
      this.validateEagerly = validateEagerly;
      return this;
    }

```
## build()
然后在看看 build()方法
 ```java
 // Builder.java
 public Retrofit build() {
       if (baseUrl == null) {
         throw new IllegalStateException("Base URL required.");
       }

       okhttp3.Call.Factory callFactory = this.callFactory;
       if (callFactory == null) {
         callFactory = new OkHttpClient();
       }

       Executor callbackExecutor = this.callbackExecutor;
       if (callbackExecutor == null) {
         callbackExecutor = platform.defaultCallbackExecutor();
       }

       // Make a defensive copy of the adapters and add the default Call adapter.
       List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
       adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

       // Make a defensive copy of the converters.
       List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

       return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
           callbackExecutor, validateEagerly);
     }
 ```

 我们看到，baseurl 为null的时候，直接抛异常了，这个baseUrl就是我们之前通过baseUrl()方法设置的，所以可知。``创建Retrofit的时候，必须执行baseUrl()方法，传递的值必须不能为null，并且需要符合一定的规则。`
 创建Retrofit，六个参数的，除了baseUrl必须在Builder中指定，其他都有默认值的，
# 定义 API 并
```java
//MovieService.java
public interface MovieService  {
    ////获取豆瓣Top250 榜单
    @GET("top250") //标签后面是这个接口的 尾址top250,完整的地址应该是 baseUrl+尾址
    //参数 使用@Query标签，如果参数多的话可以用@QueryMap标签，接收一个Map
    //@Query表示请求参数，将会以key=value的方式拼接在url后面
    Call<MovieObject> getTop250(@Query("start") int start, @Query("count") int count);
}
```
这个就没啥可说的，定义了一个接口，接口格式如上，这个挺怪的，返回的是 `Call<MovieObject>` 而且参数里面还有注解，搞毛线啊，先往后看吧，没准后面就知道为啥这样写了 ，然后就是获取API实例了

# 获取 API 实例
```java
MovieService service=retrofit.create(MovieService.class);
```
执行了Retrofit的creat()方法，那就看看这个方法到底有什么魔力吧

```java
 public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          @Override
          public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
首先判断是否是合法的接口 ，既然来了，就看看是怎么判断的吧
```java
static <T> void validateServiceInterface(Class<T> service) {
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }
    // Prevent API interfaces from extending other interfaces. This not only avoids a bug in
    // Android (http://b.android.com/58753) but it forces composition of API declarations which is
    // the recommended pattern.
    if (service.getInterfaces().length > 0) {
      throw new IllegalArgumentException("API interfaces must not extend other interfaces.");
    }
  }
```
1. 首先这个的是一个Interface，而不能是一个Class,
2. 这个Interface不能extend 其他的interface，
这样就算是合法的接口
然后看到了这个 `Proxy.newProxyInstance（）`，我擦，竟然使用的是动态代理技术，
## 动态代理
动态代理其实已经封装的很简单了，
使用newProxyInstance()方法来动态生成接口的实现类（当然生成实现类有缓存机制），并创建其实例（称之为代理），其中它内部需要传递一个类的加载器，类本身以及一个InvocationHandler处理器，
主要的动作都是在InvocationHandler中进行的，它里面只有一个方法invoke()方法，每当我们调用代理类里面的方法时invoke()都会被执行，并且我们可以从该方法的参数中获取到所需要的一切信息，比如从method中获取到方法名，从args中获取到方法名中的参数信息等，除了执行真正的逻辑（例如再次转发给真正的实现类对象），我们还可以进行一些有用的操作，例如统计执行时间、进行初始化和清理、对接口调用进行检查等。

Retrofit在这里使用到动态代理也不会例外
1. 首先，通过method把它转换成ServiceMethod ；
2. 然后，通过serviceMethod, args获取到okHttpCall 对象；
3. 最后，再把okHttpCall进一步封装并返回Call对象。

## loadServiceMethod()
```java
  ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```
loadServiceMethod()方法很好理解，
这里面实现了一个缓存逻辑，同一个 API 的同一个方法，只会创建一次，这里由于我们每次获取 API 实例都是传入的 class 对象，而 class 对象是进程内单例的，所以获取到它的同一个方法 Method 实例也是单例的，所以这里的缓存是有效的。
首先从缓存serviceMethodCache中取，如果有直接返回。如果没取到，就通过ServiceMethod.Builder(this, method).build() 创建一个，并且添加到缓存中，然后返回ServiceMethod对象result
那么主要工作就在 ServiceMethod.Builder(this, method).build() 里面了
## ServiceMethod.Builder(this, method)
```java
 public Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }

public ServiceMethod build() {
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }
      responseConverter = createResponseConverter();
      ...
      return new ServiceMethod<>(this);
    }

 ServiceMethod(Builder<T> builder) {
    this.callFactory = builder.retrofit.callFactory();
    this.callAdapter = builder.callAdapter;
    this.baseUrl = builder.retrofit.baseUrl();
    this.responseConverter = builder.responseConverter;
    this.httpMethod = builder.httpMethod;
    this.relativeUrl = builder.relativeUrl;
    this.headers = builder.headers;
    this.contentType = builder.contentType;
    this.hasBody = builder.hasBody;
    this.isFormEncoded = builder.isFormEncoded;
    this.isMultipart = builder.isMultipart;
    this.parameterHandlers = builder.parameterHandlers;
  }
```
先通过Builder的构造函数进行一些初始化操作，
注意: 这里面有一个 methodAnnotations 数组，保存的是该方法内所有的注解，后面我们还会介绍到。

然后执行build(),创建一个ServiceMethod对象并返回，在ServiceMethod的构造函数中，我们重点关注四个对象。`callFactory，callAdapter，responseConverter 和 parameterHandlers。`
1. callFactory 负责创建HTTP请求，HTTP请求被抽象为了okhttp3.Call 类，它表示一个已经准备好，可以随时执行的HTTP请求
2. callAdapter 把`retrofit2.Call<T>` 转为T，（注意和okhttp3.Call 区分开，`retrofit2.Call<T>` 表示的是一个Retrofit方法的调用），这个过程会发送一个HTTP请求，拿到服务器返回的数据（okhttp3.Call实现），并把数据转换为声明T类型的对象，通过（`Converter<F,T>`实现）
3. responseConvert是Conver<ResponsetBody,T>类型，负责吧服务器返回的数据（JSON，XML，二进制或者其他格式，由ResponseBody封装）转换成T类型对象，
4. parameterHandlers 负责解析API定义事每个方法的参数，并在构造HTTP请求时设置参数

### callFactory
this.callFactory = builder.retrofit.callFactory()，所以 callFactory 实际上由 Retrofit 类提供，而我们在构造 Retrofit 对象时，可以指定 callFactory，如果不指定，将默认设置为一个 okhttp3.OkHttpClient。

### callAdapter
callAdapter 还是由 Retrofit 类提供。在 Retrofit 类内部，将遍历一个 CallAdapter.Factory 列表，让工厂们提供，如果最终没有工厂能（根据 returnType 和 annotations）提供需要的 CallAdapter，那将抛出异常。而这个工厂列表我们可以在构造 Retrofit 对象时进行添加。

### responseConverter
同样，responseConverter 还是由 Retrofit 类提供，而在其内部，逻辑和创建 callAdapter 基本一致，通过遍历 Converter.Factory 列表，看看有没有工厂能够提供需要的 responseBodyConverter。工厂列表同样可以在构造 Retrofit 对象时进行添加。

### parameterHandlers
每个参数都会有一个ParameterHanders，由ServiceMethod#parseParameter()方法负责创建，其主要内容就是解析每个参数使用的注解类型，（诸如 Path，Query，Field等），对每种类型进行单独的处理，构造HTTP请求时，我们传递的参数都是字符串，Retrofit是如何把我们传递的各种参数都转化为String呢，还是由Retrofit类提供的convert，

Converter.Factory 除了提供上一小节提到的 responseBodyConverter，还提供 requestBodyConverter 和 stringConverter，API 方法中除了 @Body 和 @Part 类型的参数，都利用 stringConverter 进行转换，而 @Body 和 @Part 类型的参数则利用 requestBodyConverter 进行转换。

这三种 converter 都是通过“询问”工厂列表进行提供，而工厂列表我们可以在构造 Retrofit 对象时进行添加。


### 工厂让各个模块得以高度解耦
上面提到了三种工厂 okhttp3.Call.Factory,CallAdapter.Factory和Converter.Factory，分别负责提供不同的模块，至于怎么提供，提供何种模块，统统交给工厂，Retrofit完全不参合，它只负责提供用于决策的信息，例如参数，返回值类型，注解。

高内聚低耦合，第一步就是面向接口变成，模块之间，类之间，通过接口进行依赖，创建怎么样的实例，则交给工厂负责，工厂同意也是接口，添加怎么样的工厂，则在最迟构造Retrofit对象的时候决定，各个模块之间完全解耦，每个模块只专注与自己的的职责。

## ServiceMethod.Builder(this, method).build()
下面来详细的解释下build()方法，完全理解了该方法则便于理解下面的所有执行流程。
### 构建CallAdapter对象
该对象将在第三步中起着重要作用。
```java
  private CallAdapter<?> createCallAdapter() {
      Type returnType = method.getGenericReturnType();
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      Annotation[] annotations = method.getAnnotations();
      try {
        return retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }
```
creatCallAdapter()方法主要就是获取method的类型和注解，然后调用retrofit.callAdapter(returnType, annotations)方法

```java
 public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }
  public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
	...
  }
```
最后执行到了Retrofit的nextCallAdapter()方法中，在for循环中分别从adapterFactories中来获取CallAdapter对象，这个adapterFactories又是什么时候赋值的呢，查看源码可知，adapterFactories 只在Retrofit 的构造函数中赋值过一次，并且得到的是一个 只读不可写的集合(把传递过来的集合对象进行了unmodifiableList()处理)，从其他对它的操作也可以看出，只是get操作，没有add或者remove，

```java
Retrofit(okhttp3.Call.Factory callFactory, HttpUrl baseUrl,
      List<Converter.Factory> converterFactories, List<CallAdapter.Factory> adapterFactories,
      Executor callbackExecutor, boolean validateEagerly) {
	...
    this.adapterFactories = unmodifiableList(adapterFactories); // Defensive copy at call site.
	...
  }
```
而这个构造函数我们知道，是在Builder.build()中执行的。
```java
public Retrofit build() {
     ...
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
	...
      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
```
adapterFactories 这个集合再那里被add的呢？就是addCallAdapterFactory（）方法

```java
    public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
      adapterFactories.add(checkNotNull(factory, "factory == null"));
      return this;
    }
```
如果我们不调用这个方法呢，没关系，它会设置一个默认的，就是Platform.defaultCallAdapterFactory()方法中得到一个
```java
  CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
    if (callbackExecutor != null) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    return DefaultCallAdapterFactory.INSTANCE;
  }
```
callbackExecutor 不为null，就算为null，所以，adapterFactories中至少会add一个CallAdapter，也就是ExecutorCallAdapterFactory ，
再来看看这个for循环、

```java
  public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
	...
    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
	...
  }
```
我们知道adapterFactories中至少会add一个CallAdapterFactory ，就是ExecutorCallAdapterFactory，那么我们就看看ExecutorCallAdapterFactory的get()方法是什么吧

```java
//ExecutorCallAdapterFactory.java
@Override
  public CallAdapter<Call<?>> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public <R> Call<R> adapt(Call<R> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
```
这里面创建一个CallAdapter对象,然后返回，这个CallAdapter对象就会赋值给serviceMethod中的callAdapter，


### 构建responseConverter转换器对象
它的作用是寻找适合的数据类型转化
该对象的构建和构建CallAdapter对象的流程基本是一致的，这里就不在赘述。同学们可自行查看源码。

### 解析注解
通过for循环，得到方法中所有的注解，然后一一解析
```java
   for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }
```
在Builder的构造函数中，对这个methodAnnotations 进行赋值，保存的是该方法中所有的注解。

通过for循环，一个一个解析该方法上的注解。然后查看 parseMethodAnnotation()

```java
  private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
        if (!Void.class.equals(responseType)) {
          throw methodError("HEAD method must use Void as response type.");
        }
      } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
      } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
      } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
      } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
      } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
      } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
          throw methodError("@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
      } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isMultipart = true;
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }
```
然后就看到了对各种注解的解析，例如GET，POST，

```java
  private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      if (this.httpMethod != null) {
        throw methodError("Only one HTTP method is allowed. Found: %s and %s.",
            this.httpMethod, httpMethod);
      }
      this.httpMethod = httpMethod;
      this.hasBody = hasBody;

      if (value.isEmpty()) {
        return;
      }

      // Get the relative URL path and existing query string, if present.
      int question = value.indexOf('?');
      if (question != -1 && question < value.length() - 1) {
        // Ensure the query string does not have any named parameters.
        String queryParams = value.substring(question + 1);
        Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
        if (queryParamMatcher.find()) {
          throw methodError("URL query string \"%s\" must not have replace block. "
              + "For dynamic query parameters use @Query.", queryParams);
        }
      }

      this.relativeUrl = value;
      this.relativeUrlParamNames = parsePathParameters(value);
    }
```

最后把得到的注解中设置的值保存到了relativeUrl，看意思这个应该就是相对的URL，后面还会用到这个值


## 通过serviceMethod, args获取到okHttpCall 对象
第二步相对比较简单，就是对象传递：
```java
OkHttpCall(ServiceMethod<T> serviceMethod, Object[] args) {
    this.serviceMethod = serviceMethod;
    this.args = args;
  }
```

## 把okHttpCall进一步封装并返回Call对象
也就是 serviceMethod.callAdapter.adapt(okHttpCall) 这行代码
先看看 adapt()是什么玩意吧
`<R> T adapt(Call<R> call);`
我曹，这是毛线啊，但是我们在之前分析过了，serviceMethod.callAdapter其实就是ExecutorCallAdapterFactory 中通过get()方法new 出来的一个，那么就直接看这一部分吧

```java
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  @Override
  public CallAdapter<Call<?>> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public <R> Call<R> adapt(Call<R> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
  ...
  }
```
所以 serviceMethod.callAdapter.adapt(okHttpCall) 就是执行了 new ExecutorCallbackCall<>(callbackExecutor, call) 这行代码。
```java
 ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }
```
我们知道，这个callbackExecutor，它主要是把现在运行的线程切换到主线程中去
而delegate,其实就是传递过来的 okHttpCall 对象，这个对象就是真真正正的执行网络操作的对象
而这个ExecutorCallbackCall 对象，就是最后通过动态代理生成接口对象，执行接口方法返回的Call对象，即call
```java
 Call<MovieObject> call=movieService.getTop250(0,20);
```

到这里先告一段落，然后继续看我们写的代码
当我们得到接口代理实例后，就可以通过代理接口调用里面的方法了，就会触发InvocationHandler对象中的invoke()方法，从而完成上面的三步骤返回一个Call对象，通过Call对象，就可以去完成我们的请求了，Retrofit为我们提供了两种请求方式，
```java
 //进行网络异步请求
        call.enqueue(new Callback<MovieObject>() {
            @Override
            public void onResponse(Call<MovieObject> call, Response<MovieObject> response) {
                Log.e("hoyouly", "onResponse: "+response.body().subjects+);
            }

            @Override
            public void onFailure(Call<MovieObject> call, Throwable t) {
                t.printStackTrace();
                Log.e("hoyouly", "onFailure: "+t.getMessage());
            }
        });
```
enqueue()方法接受一个Callback对象，然后里面有两个回调函数，分别代表请求成功onResponse()和失败onFailure()方法，
之前我们知道，返回的这个call 对象，其实就是ExecutorCallAdapterFactory 中ExecutorCallbackCall对象，
那么我们就直接看ExecutorCallbackCall中的enqueue()方法吧

```java
  @Override public void enqueue(final Callback<T> callback) {
      if (callback == null) throw new NullPointerException("callback == null");

      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }
```
通过代码可知，执行了delegate的enqueue()方法，而在上面，我们就知道，这个delegate就是okHttpCall对象，那么看看okHttpCall 中的

```java
 @Override public void enqueue(final Callback<T> callback) {
    if (callback == null) throw new NullPointerException("callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }

    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }
```

其实它就是这里面封装了一个okhttp3.Call，直接利用okhttp进行网络的异步操作，至于okhttp是怎么进行网络请求的我们就不再这里讲解了，等以后分析到再说

搬运地址：

[Android：手把手带你深入剖析 Retrofit 2.0 源码](https://blog.csdn.net/carson_ho/article/details/73732115)

[Android 网络框架之Retrofit2使用详解及从源码中解析原理](https://www.cnblogs.com/guanmanman/p/6085200.html)

[Retrofit分析-漂亮的解耦套路](https://www.jianshu.com/p/45cb536be2f4)

[RxJava 与 Retrofit 结合的最佳实践](http://gank.io/post/56e80c2c677659311bed9841)
