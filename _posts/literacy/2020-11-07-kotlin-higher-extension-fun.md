---
layout: post
title: 扫盲系列 - kotlin 之高阶函数和扩展函数
category: 扫盲系列
tags:  suspendCoroutine
---
<!-- * content -->
<!-- {:toc} -->

kotlin 也学了一点点，算是入门了，然后想着写点实际的项目吧。可是想要写好网络请求，却不知道怎么下手了，就在网上抄了一段代码，虽然能正常运行，可是却看得我傻眼了。
首先看这段代码
# 傻眼一
```kotlin
object NetworkManager {
    private val httpApi = ApiRetrofit.getHttpApi()

    suspend fun getVehicleInfo(page: String): BaseResponse<VehicleInfo> = httpApi.getVehicleInfo(page).await()

    private suspend fun <T> Call<T>.await(): T {
      return suspendCoroutine { continuation ->
            enqueue(object : Callback<T> {
                override fun onResponse(call: Call<T>, response: Response<T>) {
                    val body = response.body()
                    if (body != null) {
                        continuation.resume(body)
                    } else {
                        continuation.resumeWithException(RuntimeException("response body is null"))
                    }
                }
                override fun onFailure(call: Call<T>, t: Throwable) {
                    continuation.resumeWithException(t)
                }
            })
        }
    }
}
```

前面的还能看懂，就是普通的Retrofit得到一个ApiService，即httpApi,然后执行HttpApi的getVehicleInfo(),可是后面await()就看不懂了，这是啥玩意儿
看await()的声明 `private suspend fun <T> Call<T>.await(): T {}`

先说说我看懂的吧。
1. suspend 是一个挂起函数，可以在supend函数或者协程里面执行。这个我还明白。
2. <T> 泛型,该函数返回类型是T，我也能看懂，

可是主要看不懂的是 Call<T>.await()这个函数名称，这是啥玩意儿啊，在java中从来没见过了，而且看里面的实现，竟然能直接调用Call的enqueue()方法，

还有 suspendCoroutine 又是啥啊，

慢慢来找答案吧。
先说这个Call<T>.await()吧，

查资料才知道。这是Kotlin的一个特性，扩展函数。在学习kotline的时候，也了解了一点点，说的例子就是什么在Activity直接shotTaost()，就是使用扩展函数实现的，但是当时没太注意。原来这个就是扩展函数的实际使用场景啊。

## 扩展函数

扩展函数可以在已有的类中添加新的方法，不会对原类做修改,扩展函数定义形式

```kotlin
fun receiverType.functionName(params){
    body
}
```
* receiverType：表示函数的接收者，也就是函数扩展的对象
* functionName：扩展函数的名称
* params：扩展函数的参数，可以为NULL

现在再看看这个 private suspend fun <T> Call<T>.await(): T {} 这个声明，应该就明白了

其实就是对Call这个对象扩展了一个await()方法，因为是对Call进行的扩展，所以可以执行Call里面的方法，例如enqueue()方法。

接下来说几个扩展函数特点吧
### 特点一
扩展函数是静态解析的，并不是接收者类型的虚拟成员，在调用扩展函数时，具体被调用的的是哪一个函数，由调用函数的的对象表达式来决定的，而不是动态的类型决定的:
```kotlin
open class C

class D: C()

fun C.foo() = "c"   // 扩展函数 foo

fun D.foo() = "d"   // 扩展函数 foo

fun printFoo(c: C) {
    println(c.foo())  // 类型是 C 类
}

fun main(arg:Array<String>){
    printFoo(D())  //结果是 c,而不是d,
}
```
### 特点二

若扩展函数和成员函数一致，则使用该函数时，会优先使用成员函数。

另外，伴生对象和作用域以及成员都可以进行扩展的。

然后说说suspendCoroutine

## suspendCoroutine

suspendCoroutine 是一个函数，看着像是和协程有关，但是并不是帮我们启动协程的，它运行在协程当中并且帮我们获取到当前协程的 Continuation 实例，也就是拿到回调，方便后面我们调用它的 resume 或者 resumeWithException 来返回结果或者抛出异常。

suspendCoroutine 将外部的代码执行权拿走，并转入传入的 Lambda 表达式中，而这个表达式当中的操作就对应异步的耗时操作了，在这里我们操作网络请求，网络请求成功，调用 continuation.resume 将得到的结果body传了出去，这个同时也是 suspendCoroutine 的返回值，即 body，如果失败，则调用continuation.resumeWithException()把异常抛出去。这个也同时是suspendCoroutine 的返回结果。即抛出异常。

# 傻眼二
又看到了这样一段代码，又开始崩溃了
```kotlin
object VehicleRepository {

    fun getVehicleInfo(info: String) = fire {
        val vehicleInfo: BaseResponse<VehicleInfo> = NetworkManager.getVehicleInfo(info)
        if (vehicleInfo != null) {
            Result.success(vehicleInfo.result)
        } else {
            Result.failure(NullPointerException(" info is null"))
        }
    }
  }

fun <T> fire(block: suspend () -> Result<T>) =
    liveData {
        val result = try {
            block()
        } catch (e: Exception) {
            Log.e("hoyouly", e.toString())
            Result.failure<T>(e)
        }
        emit(result)
    }  
```

这又是啥啊，fire()函数的声明更怪了，什么跟什么啊，Kotlin难道就喜欢玩这种让人看不懂的玩意吗。可是谁让咱要接受新知识呢，慢慢研究吧。经过研究才明白，这里面其实涉及到以下几个点
1. 泛型，这个容易理解，<T> ,标准的泛型
2. lambda表达式。
3. LiveData 协程构造方法
4. 高阶函数


1 就不说了，2后面会说道，先说说3， Lviedata 的协程构造方法,先看源码吧。

```kotlin
fun <T> liveData(
    context: CoroutineContext = EmptyCoroutineContext,
    timeoutInMs: Long = DEFAULT_TIMEOUT,
    @BuilderInference block: suspend LiveDataScope<T>.() -> Unit
): LiveData<T> = CoroutineLiveData(context, timeoutInMs, block)
```
这是在LifeCycle 2.2.0  之后出现的，它提供了一个协程代码块，这个块就是 LiveData 的作用域，当 LiveData 被观察的时候，里面的操作就会被执行，当 LiveData 不再被使用时，里面的操作就会取消。而且该协程构造方法产生的是一个不可变的 LiveData，可以直接暴露给对应的视图使用。而 emit() 方法则用来更新 LiveData 的数据。

因为 CoroutineContext 和 timeoutInMs 都有默认值，只有这个代码块是必须要传递的，而根据Lambda表达式规则，函数最后一个是Lambda表达式的时候，可以把这个Lambda表达式放到函数体外面，也就成了`livedata{}`这种格式了。

如果把上面的还原回来，是这样的
```kotlin
fun <T> fire(bl: suspend () -> Result<T>) {
    liveData( block = {
        val result = try {
            bl()
        } catch (e: Exception) {
            Result.failure<T>(e)
        }
        emit(result)
    })
}
```
为了方便区分，我把参数block改为了bl,因为liveData的一个构造参数就是block，

这样应该就看起来清晰多了吧

创建了一个liveData对象，然后执行了里面的bl(),最后把结果发送出去了，
这个bl() 就是一个函数，它声明是`bl: suspend () -> Result<T>`,一个可挂起函数，suspend,返回一个Result<T> 类型

简单来说，就是
fire()是接收一个可挂起函数作为参数的函数，
在fire中，创建一个Livedata对象，然后里面执行这个可挂起函数，最后把结果返回

接收一个函数的作为参数的函数，那不就是高阶函数吗，没错，fire()就是一个高阶函数。

## 高阶函数
将函数用作一个函数的参数或者返回值的函数。

其实是依赖编译器强大的功能将高阶函数语法转换成java代码，其内部实现是通过匿名内部类实现的。

但是这样的话，每一次调用高阶函数，都会创建一个匿名类，造成性能上的开销，为了处理这个问题，kotlin提供了内联函数的功能，它可以将lambda表达式所带来的运行时开销消除

Kotlin将内联函数中的代码在编译的时候自动替换到调用它的地方。从而减少内存开销。即避免压栈，出栈的开销

将高阶函数声明成内联函数是一种良好的编程习惯

这样就大概明白真个代码的意思了。


- - - -
搬运地址：    

[Kotlin Jetpack 实战｜00. 写给 Java 开发者的 Kotlin 入坑指南](https://juejin.im/post/6844904191098355719)

[写给Android开发者的Kotlin入门](https://www.cnblogs.com/it-tsz/p/10751332.html)

[Kotlin 教程](https://www.runoob.com/kotlin/kotlin-tutorial.html)

[Java转Kotlin：内联函数](https://juejin.im/post/6844904198102843399)
