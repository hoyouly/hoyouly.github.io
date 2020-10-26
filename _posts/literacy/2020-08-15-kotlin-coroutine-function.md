---
layout: post
title: 扫盲系列 - kotlin 之协程相关函数
category: 扫盲系列
tags:  
---
<!-- * content -->
<!-- {:toc} -->


### await()

await()签名并不是我们看到的 `public suspend fun await(): T`,而是
```kotlin
kotlinx/coroutines/Deferred.await (Lkotlin/coroutines/Continuation;)Ljava/lang/Object;
```
是接收一个 Continuation 实例，返回 Object 的这么个函数，所以前面的代码我们可以大致理解为：
```kotlin
//注意以下不是正确的代码，仅供大家理解协程使用
GlobalScope.launch(Dispatchers.Main) {
    gitHubServiceApi.getUser("bennyhuo").await(object: Continuation<User>{ //await 中接收一个object的实例
            override fun resume(value: User) {
                showUser(value)
            }
            override fun resumeWithException(exception: Throwable){
                showError(exception)
            }
    })
}
```
而await()大致的就是通过Handler切换线程的，类似
```kotlin
//注意以下并不是真实的实现，仅供大家理解协程使用
fun await(continuation: Continuation<User>): Any {
    ... // 切到非 UI 线程中执行，等待结果返回
    try {
        val user = ...
        handler.post{ continuation.resume(user) }
    } catch(e: Exception) {
        handler.post{ continuation.resumeWithException(e) }
    }
}
```

记住： **从执行机制上来讲，协程跟回调没有什么本质的区别。**

### join 和 await()
* join() 只关心是否执行完，至于怎么完成的，不关心，在协程出现异常的时候，不会抛出异常，
* await() 期望 deferred 能够给我们提供一个合适的结果，在协程出现异常的时候，会抛出异常，协程体后面的代码就不会执行，

```kotlin
val deferred = GlobalScope.async<Int> {
    throw  java.lang.ArithmeticException();
}
try {
    val value = deferred.await()
    log("1 -> $value")
} catch (e: Exception) {
    log("2 -> $e")
}
```
输出的结果是
```kotlin
[DefaultDispatcher-worker-1] 2 -> java.lang.ArithmeticException
```
因为  deferred 没有提供一个合适的结果，抛出了异常，所以就执行到了catch中。

而换成 join()的时候，输出的结果就不一样了

```kotlin
val deferred = GlobalScope.async<Int> {
    throw  java.lang.ArithmeticException();
}
try {
    deferred.join()
    log("1 ")
} catch (e: Exception) {
    log("2 -> $e")
}
```
输出的结果
```kotlin
[DefaultDispatcher-worker-1] 1
```
因为 join()不关心结果，只关心是否执行完，抛出异常也算是执行完的一种，所以可以继续往下面执行


### join()和start()区别
launch()返回的是一个Job ,
调用job.start(),主动触发协程的调度执行
调用job.join(),隐式的触发协程的调用执行

```kotlin
log(1)
val job = GlobalScope.launch(start = CoroutineStart.LAZY) {
    log(2)
}
log(3)
job.start()
log(4)
```
结果可能是4 在2 前面，如果运气好的话，也可能出现2 在4前面先执行

可是如果要是换成job.join(),
```kotlin
log(1)
val job = GlobalScope.launch(start = CoroutineStart.LAZY) {
    log(2)
}
log(3)
job.join()
log(4)
```
那么一定是 2 在4 之前执行的，因为要等待协程执行完毕，join 有点像是同步了，但是执行的代码却不是在同一个线程中

```kotlin
suspend fun main() {
    log(1)
    val job = GlobalScope.launch(start = CoroutineStart.ATOMIC) {
        log(2)
    }
    job.cancel()
    log(3)
}
```
创建了协程后立即 cancel，但由于是 ATOMIC 模式，因此协程一定会被调度，因此 1、2、3 一定都会输出，只是 2 和 3 的顺序就难说了。
如果是 DEFAULT 模式，在第一次调度该协程时如果 cancel 就已经调用,2 有可能会输出，也可能不会


### suspend 函数
一个普通的函数通常支持两种操作，call 和return，因为每个函数都支持调用，同时也支持获取它的返回值，当一个函数上声明为suspend的之后，这个函数会支持额外两种操作。suspend 和resume.
* suspend 指函数可以在一个特定的时间点被挂起，好像函数突然停止运行了，然后它会被存储到一个状态，暂停它的运行，
* resume 值可以恢复之前的挂起函数的状态，让它从被挂起的状态，继续往下执行。

suspend 函数是Kotlin编译期对 协程支持的唯一黑魔法，一个函数如果声明为suspend,这个函数就变成了挂起函数.

<font color="#ff000" > 一个挂起函数，它只能在另外一个挂起函数或者在一个协程作用域中才能使用</font>


```kotlin
@GET("users/{login}")
suspend fun getUser(@Path("login") login: String): User
```
这种情况Retrofit会根据接口声明来构造一个Continuation, 并且在内部封装一个Call的异步请求，进而得到一个User实例。
使用方法
```kotlin
GlobalScope.launch {
    try {
      val user = gitHubServiceApi.getUser("bennyhuo")
      showUser(user)
    } catch (e: Exception) {
      showError(e)
    }
}
log("aaa")
```

当调用 gitHubServiceApi.getUser("bennyhuo") 的时候，Retrofit会开启一个IO线程，在IO线程中进行网络请求，同时，将当前的协程挂起。但是主线程可以正常运行的，所以log("aaa")这个是可以正常输出的，不管Retrofit网络请求花费多长时间，都完全不会阻塞主线程的。

当网络请求结束后，Retorfit会将之前挂起的协程再恢复，恢复到gitHubServiceApi.getUser("bennyhuo") 被挂起的位置，得到服务器响应的数据后，将它复制到user对象上，代码就可以正常的继续下一步进行了，再继续调用showUser()函数，将数据显示到界面上。

### delay
类似发送一个延迟消息，这个时候她释放CPU资源，在JVM中，delay 实际上是在ScheduledExcecutor 中添加了一个延迟任务，因此会发生线程切换

Thread.sleep()  堵塞当前线程 ,
delay() 是一个 挂起函数(suspend function)，可在不堵塞线程的情况下延迟协程；



### withContext
withContext 函数可以切换到指定的线程，并在闭包内的逻辑执行结束后，自动把线程切换回上下文继续执行

不会创建新的协程，常用于 切换代码执行所运行的线程。 它也是一个挂起方法，直到结束返回结果。多个withContext是串行执行的， 所以很适合那种一个任务依赖上一个任务返回结果的情况

当withContext运行结束后，最后一行代码会作为返回值返回，

### runBlocking

runBlocking{}函数是 kotlin 建立的一种来堵塞当前线程的协程，可实现 Thread.sleep()的效果  runBlocking {  delay(2000)  }




- - - -
搬运地址：    

[Kotlin Jetpack 实战｜00. 写给 Java 开发者的 Kotlin 入坑指南](https://juejin.im/post/6844904191098355719)

[写给Android开发者的Kotlin入门](https://www.cnblogs.com/it-tsz/p/10751332.html)

[Kotlin 教程](https://www.runoob.com/kotlin/kotlin-tutorial.html)

[破解 Kotlin 协程](https://juejin.im/user/2365804754513085/posts)

[枯燥的Kotlin协程三部曲(上)——概念启蒙篇](https://juejin.im/post/6854573213704912910)

[枯燥的Kotlin协程三部曲(中)——应用实战篇](https://juejin.im/post/6860464281272451080)

[GDG上海实录回顾，带你快速上手Kotlin协程](https://www.ershicimi.com/p/84a42a96313db61133eb10a37b703421)
