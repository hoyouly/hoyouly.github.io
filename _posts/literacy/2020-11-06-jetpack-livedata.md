---
layout: post
title: 扫盲系列 - Jetpack 之Livedata
category: 扫盲系列
tags:  Jetpack Livedata
---
<!-- * content -->
<!-- {:toc} -->

持有从数据源获取到的数据，并且它可以被 DataBinding 组件观察，当 Activity 被销毁时，它将被取消订阅。
不支持线程切换，其次不支持背压
## 优点
* 数据变更的时候更新UI
* 没有内存泄漏
* 不会因为停止Activity崩溃
* 无需手动处理生命周期
* 共享资源
## 使用方式
* observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T>
最常用的方法，需要提供Observer处理数据变更后的处理。LifecycleOwner则是我们能够正确处理声明周期的关键！
* setValue(T value) 设置数据
* getValue():T 获取数据
* postValue(T value) 在主线程中更新数据
## 使用场景

绝大部分的LiveData都是配合其他Android Jetpack组件使


在Jetpack组件中，为各个组件提供了对应的scope,比如
ViewModel对应的viewModelScope

AppCompatActivity 或 Fragment中则可以使用 lifecycleScope


您可能还需要在生命周期的某个状态 (启动时/恢复时等) 执行一些操作，这时您可以使用 launchWhenStarted、launchWhenResumed、launchWhenCreated 这些方法:

launchWhenStarted 中设置了一个操作，当 Activity 被停止时，这个操作也会被暂停，直到 Activity 被恢复 (Resume)。

## liveData 协程构造方法


```kotlin

class MyViewModel {
    val result = liveData {
        emit(doComputation())
    }
}
```
liveData 协程构造方法提供了一个协程代码块，这个块就是 LiveData 的作用域

当 LiveData 被观察的时候，里面的操作就会被执行，

当 LiveData 不再被使用时，里面的操作就会取消。

而且该协程构造方法产生的是一个不可变的 LiveData，可以直接暴露给对应的视图使用。

而 emit() 方法则用来更新 LiveData 的数据。

查看源码可知
```kotlin
un <T> liveData(
    context: CoroutineContext = EmptyCoroutineContext,
    timeoutInMs: Long = DEFAULT_TIMEOUT,
    @BuilderInference block: suspend LiveDataScope<T>.() -> Unit
): LiveData<T> = CoroutineLiveData(context, timeoutInMs, block)
```

可以一个 Dispatcher 作为参数，这样您就可以将这个协程移至另一个线程。


使用 emitSource() 方法从另一个 LiveData 获取更新的结果:

```kotlin
liveData(Dispatchers.IO) {
    emit(LOADING_STRING)
    emitSource(dataSource.fetchWeather())
}
```
- - - -
搬运地址：    

[理解协程、LiveData 和 Flow](https://www.jianshu.com/p/16aa5eaa60d7)

[写给Android开发者的Kotlin入门](https://www.cnblogs.com/it-tsz/p/10751332.html)

[Kotlin 教程](https://www.runoob.com/kotlin/kotlin-tutorial.html)

[破解 Kotlin 协程](https://juejin.im/user/2365804754513085/posts)

[枯燥的Kotlin协程三部曲(上)——概念启蒙篇](https://juejin.im/post/6854573213704912910)

[枯燥的Kotlin协程三部曲(中)——应用实战篇](https://juejin.im/post/6860464281272451080)

[GDG上海实录回顾，带你快速上手Kotlin协程](https://www.ershicimi.com/p/84a42a96313db61133eb10a37b703421)
