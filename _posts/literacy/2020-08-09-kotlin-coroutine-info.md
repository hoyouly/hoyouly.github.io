---
layout: post
title: 扫盲系列 - kotlin 之协程简介
category: 扫盲系列
tags:  
---
<!-- * content -->
<!-- {:toc} -->
# 常识
* 异步 vs 同步 阻塞 vs 非阻塞   
同步和异步的关注点是 是否同时进行    
堵塞和非堵塞关注的是 能否继续进行
* 四种基础控制流
  * 逐条执行   按照顺序，一行一行啊执行
  * 选择执行   if  else
  * 迭代 执行
  * 递归执行
* 切换
包括 切走和切回来，即挂起，和恢复
  * 挂起：保存程序的当前状态，暂停当前程序；
  * 恢复：恢复程序状态，继续执行程序；

挂起（suspend）和恢复（resume） 是主动行为，堵塞是被动行为

## 进程
* 单个进程只能干一件事，进程中的代码依旧是串行执行。
* 执行过程如果堵塞，整个进程就会挂起，即使进程中某些工作不依赖于正在等待的资源，也不会执行。
* 多个进程间的内存无法共享，进程间通讯比较麻烦。

## 线程
* 由 线程ID、程序计数器、寄存器组合和堆栈 共同组成。线程的引入减小了程序并发执行时的开销，提高了操作系统的并发性能。
* 线程的出现是为了降低上下文切换消耗，提高系统的并发性，并突破一个进程只能干一件事的缺陷，使得 进程内并发 成为可能
* 线程大多数的实现是映射到内核的线程的。也就是说线程中的代码是线程抢占到CPU的时间片的时候才会执行。否则就得歇着
* 线程切换上下文环境 包括 `线程Id + 线程状态 + 堆栈 + 寄存器状态等`

### 进程和线程区别
* 一个程序至少有一个进程，一个进程至少有一个线程，可以把进程理解做线程的容器；
* 进程在执行过程中拥有 独立的内存单元，该进程里的多个线程 共享内存；
* 进程可以拓展到 多机，线程最多适合 多核
* 每个独立线程有一个程序运行的入口、顺序执行列和程序出口，但不能独立运行，需依存于应用程序中，由应用程序提供多个线程执行控制；
* 进程是资源分配的最小单元，线程是CPU调度的最小单元
* 进程和线程都是一个时间段的描述，即 CPU工作时间段的描述，只是颗粒大小不同。

## 协程

一种非抢占式(协作式)的任务调度模式，程序可以主动挂起或者恢复执行。

### 轻量级原因
1. 线程的上下文切换都需要内核参与，而协程的上下文切换，完全由用户控制，避免大量的中断参与，减少的上下文切换和调度消耗的资源
2. 线程会映射到Java虚拟机的内核线程中，协程不会映射到内存的线程或者其他重的资源，它的调度在用户状态就可以搞定。任务之间的调度非抢占式，而是协作式的。
3. 协程基于线程，但是相对于线程来轻量许多，可以理解为在用户层模拟线程操作。
4. 每创建一个协程，都有一个内核态进程动态绑定，用户态下实现调度，切换，但真正执行任务的还是内核线程

上面提到内核态和用户态，简单copy过来一些相关知识。

### 内核态和用户态
* 内核态：CPU可以访问内存所有数据, 包括外围设备, 例如硬盘, 网卡. CPU也可以将自己从一个程序切换到另一个程序
* 用户态：只能受限的访问内存, 且不允许访问外围设备. 占用CPU的能力被剥夺, CPU资源可以被其他程序获取

由于需要限制不同的程序之间的访问能力, 防止他们获取别的程序的内存数据, 或者获取外围设备的数据, 并发送到网络, CPU划分出两个权限等级 :用户态 和 内核态

### 协程的特点
1. 轻量高效
2. 简单好用
3. 用同步的方式编写异步代码

# kotlin 中的协程
需要关注这两点：
* 仅仅隐藏了异步实现细节，让我们可以用同步的写法来写异步操作罢了。
* 是假协程，只是对底层Thread的一次良好封装。

## 启动协程的方式

创建一个协程可以有很多中方式，可以通过 launch 启动一个协程。例如
```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext, // 上下文
    start: CoroutineStart = CoroutineStart.DEFAULT,  // 启动模式
    block: suspend CoroutineScope.() -> Unit // 协程体
): Job
```
我们可以看到，一个协程包括一下三部分：
* 上下文
* 启动模式
* 协程体

上下文和启动模式都有默认值的，协程体可以理解为就是Thread中run()中的内容。

启动一个协程，返回的是一个Job对象，可以理解为就是Java中的Thread，可以看看这两个源码
![添加图片](../../../../images/job_thread.png)

我们可以看到，Thread和Job基本功能一致，他们都承载了一段代码逻辑，Thread是通过run()方法，Job可以通过协程构造的lambda或者函数，也都包含这段代码的运行态。


### 协程上下文
这是完成某件事情所需要的前置资源,是完成某项事务所需的外部环境。

类型是 CoroutineContext ,通常见的上下文类型是
* CombinedContext ,上下文组合，表示很多具体的上下文集合
* EmptyCoroutineContext,什么都没有

CoroutineContext 是一个数据结构,可以理解为一个以key为索引的List
```kotlin
@SinceKotlin("1.3")
public interface CoroutineContext {
    public operator fun <E : Element> get(key: Key<E>): E?
    public fun <R> fold(initial: R, operation: (R, Element) -> R): R
    public operator fun plus(context: CoroutineContext): CoroutineContext = ...
    public fun minusKey(key: Key<*>): CoroutineContext

    public interface Key<E : Element>

    public interface Element : CoroutineContext {
        public val key: Key<*>
        ...
    }
}
```
每一个Element都有一个key,因此可以作为元素出现，同时也是Coroutine的子接口，因此可以作为集合出现
可以通过上下文为协程添加一些特性，一个很好的例子就是为协程添加名称，方便调试

```kotlin
GlobalScope.launch(CoroutineName("Hello")) {
    ...
}
```

如果需要多个上下文，直接使用`+`就可以了
```kotlin
GlobalScope.launch(Dispatchers.Main + CoroutineName("Hello")) {
    ...
}
```

以键值对的方式存储不同的元素
<font color="#ff000" > Job(协程唯一标识) + CoroutineDispatcher(调度器) + ContinuationInterceptor(拦截器) + CoroutineName(协程名称，一般调试时设置)</font>

#### 协程拦截器
也是一个上下文的实现方向，，即 CoroutineContext 的子类
```kotlin
public interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>

    public fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
    ...
}
```
可以左右协程的执行，同时为了保证它的功能正常执行，协程上下文集合永远把它放在最后面，

协程的本质是回调+黑魔法，而这个回调就是被拦截的 Continuation。

协程的拦截器和OkHttp的拦截器一样
协程的调度器是拦截器的一种

所有协程启动的时候，都会有一次Continuation.resumeWith的操作，这一次操作对于调度器来说，就是一次调度机会，协程有机会调度到其他线程的关键就在于此

#### 调度器
本身是协程上下文的子类，同时实现了拦截器的接口
```kotlin
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    ...
    public abstract fun dispatch(context: CoroutineContext, block: Runnable)
    ...
}
```
dispatch 方法会在拦截器的方法 interceptContinuation 中调用，进而实现协程的调度

所以如果我们想要实现自己的调度器，继承这个类就可以了，不过通常我们都用现成的，它们定义在 Dispatchers 当中：

```kotlin
val Default: CoroutineDispatcher
val Main: MainCoroutineDispatcher
val Unconfined: CoroutineDispatcher
```


suspendCoroutine 这个方法并不是帮我们启动协程的，它运行在协程当中并且帮我们获取到当前协程的 Continuation 实例，也就是拿到回调，方便后面我们调用它的 resume 或者 resumeWithException 来返回结果或者抛出异常。

#### 上下文作用：
1. 携带参数，拦截协程执行等，多数情况下不需要自己实现，只需要使用现成的就行了，
2. 一个重要作用就是线程切换。Dispatchers.Main 就是一个官方提供的上下文，确保协程体运行到UI线程中。

```kotlin
GlobalScope.launch(Dispatchers.Main) {
    try {
        showUser(githubApiServiceApi.getUser("bennyhuo").await())
    }catch (ex:Exception){
        showError(ex)
    }
}
```
虽然getUser()执行的时候确实切换了线程，但是返回结果的时候，会再次切换回来？？？？

await()是一个 suspend 函数，这个函数只能在协程体或者其他suspend函数内部调用。他就像回调的语法糖，通过一个叫Continuation的接口实例来返回
```kotlin
@SinceKotlin("1.1")
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resume(value: T)
    public fun resumeWithException(exception: Throwable)
}
```
其实就是异步回调，

上面那段代码流程执行的本质就是`异步回调`,而代码之所以看起来是同步的，只不过是编译期的黑魔法


### 协程启动模式
是一个枚举，一共四种

```kotlin
public enum class CoroutineStart {
    DEFAULT,
    LAZY,
    @ExperimentalCoroutinesApi
    ATOMIC,
    @ExperimentalCoroutinesApi
    UNDISPATCHED;
}
```
简介如下表格
<style>
table th:first-of-type {
  width: 80px;
}
table th:nth-of-type(2) {
    width: 80px;
}
</style>

| 模式 |  功能|
|:----|:------|
|DEFAULT|launch 调用后，会立即进入待调度状态，一旦调度器 OK 就可以开始执行,饿汉式启动|
|LAZY|只有在需要的情况下运行,懒汉式启动|
|ATOMIC|立即执行协程体，在遇到第一个挂起点之前，它的执行是不会停止的|
|UNDISPATCHED|立即执行协程体，不经过任何调度器即开始执行协程体。当然遇到挂起点之后的执行就取决于挂起点本身的逻辑以及上下文当中的调度器了。|




## 异常

### 异常传播

* GlobalScope
  一个独立的顶级协程作用域.通过 GlobeScope 启动的协程“自成一派”。
* coroutineScope{...}  
  继承外部 Job 的上下文创建作用域,在其内部的取消操作是双向传播的，子协程未捕获的异常也会向上传递给父协程
  任何一个子协程异常退出，那么整体都将退出，简单来说就是”一损俱损“
* supervisonScope{...}
  继承外部作用域的上下文，但其内部的取消操作是单向传播的，父协程向子协程传播，反过来则不然，
  子协程出了异常并不会影响父协程以及其他兄弟协程。它更适合一些独立不相干的任务,任何一个任务出问题，并不会影响其他任务的工作，简单来说就是”自作自受“
  supervisorScope 只作用域其直接子协程。

launch函数中添加try-catch,是可以正常的捕获协程体里面的异常的，可是如果把try-catch移动到launch函数外边，那么就不能捕获协程体里面的异常。

而如果把try-catch放到async函数中，并且 deferred.await 这个方法是在 try-catch 作用域当中的，这种异常不一定能捕获。

协程内部的异常通过传统的 try-catch 方式捕获没有问题，但是永远不要去做跨协程的异常捕获。

全新的方式：CoroutineExceptionHandler。

调用 CoroutineExceptionHandler 函数，并且给它传递一个 lambda 表达式，然后在 lambda 中接收异常的返回信息，lambda 中的 throwable 参数就是具体抛出的异常。可以将 CoroutineExceptionHandler 应用到 CoroutineScrop 函数当中，因为 CoroutineExceptionHandler 实际上也是一个 CoroutineContext，所以它可以用加号进行连接。
CoroutineExceptionHandler 只能放到顶层的协程里面，不要在子协程中使用它

## 结构化并发
每个并发操作都在处理一个任务，它可能属于某个父任务，也可能有自己的子任务，每个任务都拥有自己的生命周期。子任务的生命周期应该继承父任务的生命周期。
这就是业务结构化，kotlin 中的协程就是结构化并发。

实际上，我们很少需要一个全局的协程，因为它总是跟程序中某个局部作用域有关，这个局部作用域就是一个生命周期有限的实体。比如某次网络加载，新建的协程对象和父协程保持着级联关系。

协程支持嵌套，这种嵌套是有父子结构的。

线程里面虽然可以再开线程，可是这两个线程是没任何父子关系的。不管在线程中开启多少个线程，都是一个个独立的线程，跟创建它的外层线程之间没有任何关联。


具体表现就是协程必须在作用域中才能启动。


## 协程作用域   CoroutineScope
协程必须在作用域中才能启动，作用域中定义了一些父子协程的规则，Kotlin协程通过作用域来管控域中的所有协程。

作用域可并列和包含，组成一个树状结构。细分主要包括顶级作用域，协同作用域，主从作用域三种。

* 顶级作用域：没有父协程的协程所在的作用域；   GlobalScope，在整个JVM虚拟中只有一份对象实例，生命周期贯穿整个JVM，故使用时需要警惕 内存泄漏！！
* 协同作用域：协程中启动新协程(子协程)，此时子协程所在的作用域默认为协同作用域，子协程抛出的未捕获异常都将传递给父协程处理，父协程同时也会被取消；   coroutineScope()
* 主从作用域：与协同作用域父子关系一致，区别在于子协程出现未捕获异常时不会向上传递给父协程。  supervisorScope()

MainScope()  为了在Android/JavaFx等场景中更方便的使用，官方提供了 MainScope() 函数快速创建基于主线程协程作用域。

父子协程间的规则
* 父协程被取消，所有子协程均被取消；
* 父协程需等待子协程执行完毕后才会最终进入完成状态，而不管父协程本身的协程体是否已执行完；
* 子协程会继承父协程上下文中的元素，如果自身有相同Key的成员，则覆盖对应Key，覆盖效果仅在自身范围内有效。

协程取消
如果父协程取消的时候，子协程已经开始执行了，那么子协程不会立即停止，而是会执行完，因为协程的取消是不会影响到。所以在子协程 执行耗时逻辑之前，都需要来做一次协程是否处于运行状态的检查。
```kotlin
fun main() =runBlocking { //runBlocking 会启动一个 Job，因此这里也存在默认的作用域
    val startTime = System.currentTimeMillis()
    val job1 = launch(Dispatchers.IO) {
        var nextPrintTime = startTime
        var i = 1
        while (i <= 5 ) {
            if (System.currentTimeMillis() >= nextPrintTime) {
                log("${i++}")
                nextPrintTime += 500L
            }
        }
        log("while循环结束")
        val user = getUserCoroutine()
        log(user)
    }
    delay(1000)
    log("取消")
    job1.cancel()
    log("完成")
}
```
结果如下

```kotlin
22:08:27:886 [DefaultDispatcher-worker-1] 1
22:08:28:341 [DefaultDispatcher-worker-1] 2
22:08:28:841 [DefaultDispatcher-worker-1] 3
22:08:28:854 [main] 取消
22:08:28:855 [main] 完成
22:08:29:341 [DefaultDispatcher-worker-1] 4
22:08:29:841 [DefaultDispatcher-worker-1] 5
22:08:29:841 [DefaultDispatcher-worker-1] while循环结束
```
虽然也执行了取消操作，可是输出的结果while循环还是执行完了，并且log("while循环结束") 也打印出来了，但是getUserCoroutine()却没执行，感觉是getUserCoroutine()这里算是真正cancel掉了，其实也是这样的，因为getUserCoroutine()是一个suspend函数，在这里会检查协程是否取消，如果取消，就不会执行了，

如果想在耗时中检查协程是否取消，就需要通过isActive进行判断。

```kotlin
fun main() = runBlocking { //runBlocking 会启动一个 Job，因此这里也存在默认的作用域
    val startTime = System.currentTimeMillis()
    val job1 = launch(Dispatchers.IO) {
        var nextPrintTime = startTime
        var i = 1
        while (i <= 5 && isActive) {
            if (System.currentTimeMillis() >= nextPrintTime) {
                log("${i++}")
                nextPrintTime += 500L
            }
        }
        log("while循环结束")
        val user = getUserCoroutine()
        log(user)
    }
    delay(1000)
    log("取消")
    job1.cancel()
    log("完成")
}
```
这次只是在while循环中多添加了一个判断，isActive用来判断协程是否存活。结果就不一样了
```kotlin
22:14:40:791 [DefaultDispatcher-worker-2] 1
22:14:41:249 [DefaultDispatcher-worker-2] 2
22:14:41:749 [DefaultDispatcher-worker-2] 3
22:14:41:760 [main] 取消
22:14:41:761 [DefaultDispatcher-worker-2] while循环结束
22:14:41:761 [main] 完成
```
收到协程取消的状态，while循环就直接结束了。

- - - -
搬运地址：    

[Kotlin Jetpack 实战｜00. 写给 Java 开发者的 Kotlin 入坑指南](https://juejin.im/post/6844904191098355719)

[写给Android开发者的Kotlin入门](https://www.cnblogs.com/it-tsz/p/10751332.html)

[Kotlin 教程](https://www.runoob.com/kotlin/kotlin-tutorial.html)

[破解 Kotlin 协程](https://juejin.im/user/2365804754513085/posts)

[枯燥的Kotlin协程三部曲(上)——概念启蒙篇](https://juejin.im/post/6854573213704912910)

[枯燥的Kotlin协程三部曲(中)——应用实战篇](https://juejin.im/post/6860464281272451080)

[GDG上海实录回顾，带你快速上手Kotlin协程](https://www.ershicimi.com/p/84a42a96313db61133eb10a37b703421)

[大话版用户态和内核态](https://www.cnblogs.com/shangxiaofei/p/5567776.html)
