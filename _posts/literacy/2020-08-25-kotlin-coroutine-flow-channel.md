---
layout: post
title: 扫盲系列 - kotlin 之协程 Flow 和Channel
category: 扫盲系列
tags:  
---
<!-- * content -->
<!-- {:toc} -->


## Channel

一个队列，并且是并发安全的，他可以用来连接协程，实现不同协程的通信

## Flow
是 Kotlin 协程与响应式编程模型结合的产物


delay 不能在 SequenceScope 的扩展成员当中被调用，因而不能在序列生成器的协程体内调用了。

flow 的 执行体内部也可以调用其他挂起函数，比如delay，但是sequence中就可以。


```kotlin
val inflow = flow {
      //执行体内部也可以调用其他挂起函数
      (1..3).forEach {
          emit(it)
          delay(100)
      }
      throw NullPointerException()
  }.catch { t: Throwable ->
      //直接调用catch进行异常捕获,这个只能捕获上游的异常
      log(t)
      //尽管是 catch中，也可以使用 emit 重新生产新元素出来
      emit(10)
  }.onCompletion {//流完成时执行逻辑，可以使用 onCompletion,类似 try catch finally中的finally，无论前面是否有异常，都会被调用
      log("onCompletion")
  }

  GlobalScope.launch {
      //设定运行时所使用的调度器，这个只会对他之前的的操作有影响，
      inflow.flowOn(Dispatchers.Default).collect {
          //最终消费 intFlow 需要调用 collect 函数，这个函数也是一个挂起函数
          log(it)
      }
      //可以进行重复消费
      inflow.flowOn(Dispatchers.IO).collect {
          log(it)
      }
  }.join()
```

### 操作符
* 末端操作符
由于 Flow 的消费端一定需要运行在协程当中，因此末端操作符都是挂起函数。
collect  是最基本的末端操作符， collect 所在协程的调度器则与 observeOn 指定的调度器对应。
flowOn() 与 subscribeOn 对应
* 集合类型转换操作  toList、toSet 等。
* 聚合操作
包括将 Flow 规约到单值的 reduce、fold 等操作，以及获得单个元素的操作包括 single、singleOrNull、first 等

### Flow 的取消
Flow 的取消主要依赖于末端操作符所在的协程的状态


不能在 Flow 中直接切换调度器,也就是不能在flow中执行withContext()
想要在生成元素时切换调度器，就必须使用 channelFlow 函数来创建 Flow：
```kotlin
val channelFlow = channelFlow {
    send(6)
    log(4)
    withContext(Dispatchers.Default) {
        send(2)
        log(2)
    }
}
channelFlow.collect()
```

### Flow 的背压
* 添加缓冲，buffer()
* 使用 conflate 解决背压问题，与 Channel 的 Conflate 模式一致，新数据会覆盖老数据
* collectLatest  只处理最新的数据 ,不会直接用新数据覆盖老数据，而是每一个都会被处理，只不过如果前一个还没被处理完后一个就来了的话，处理前一个数据的逻辑就会被取消。

### Flow 拼接

* flattenConcat  按顺序拼接的，结果的顺序仍然是生产时的顺序
* flattenMerge   并发拼接，因此结果不会保证顺序


- - - -
搬运地址：    

[Kotlin Jetpack 实战｜00. 写给 Java 开发者的 Kotlin 入坑指南](https://juejin.im/post/6844904191098355719)

[写给Android开发者的Kotlin入门](https://www.cnblogs.com/it-tsz/p/10751332.html)

[Kotlin 教程](https://www.runoob.com/kotlin/kotlin-tutorial.html)

[破解 Kotlin 协程](https://juejin.im/user/2365804754513085/posts)

[枯燥的Kotlin协程三部曲(上)——概念启蒙篇](https://juejin.im/post/6854573213704912910)

[枯燥的Kotlin协程三部曲(中)——应用实战篇](https://juejin.im/post/6860464281272451080)

[GDG上海实录回顾，带你快速上手Kotlin协程](https://www.ershicimi.com/p/84a42a96313db61133eb10a37b703421)
