---
layout: post
title: 扫盲系列 - RxJava 2.0 -- 背压
category: 扫盲系列
tags:  RxJava
---
* content
{:toc}
## backpressure 背压
背压，这个玩意是Rxjava 绕不过去的坎。面试被问到了几次，都没说明白，因为糊里糊涂的。只知道 Rxjava 2.0 支持背压，Rxjava 1.0 不支持背压。但是为啥，怎么支持，就不清楚了。所以就想着查查资料搞懂这个玩意

什么是背压呢？

背压是指在异步场景中，被观察者发送事件速度远远快于观察者的处理速度的情况下，一种告诉被观察者降低发送速度的策略。

背压是流速控制的一种策略

* 背压的一个前提是异步，也就是说观察者和被观察者在不同的线程。如果 观察者和被观察者在同一个线程中，这个时候被观察者发送的事件，必须等到观察者接收处理完以后才能发送下一个事件
* 背压并是不像flatmap一样可以在程序中直接使用的操作符，他只是一种控制事件流速的策略

主要目的是控制流量的，水缸的控制能力毕竟有限，因此还得从源头控制。

Rxjava 2.0 支持背压，Rxjava 1.0 不支持背压。可是 在 Rxjava 1.0 上，怎么解决背压的问题呢？

## Rxjava 1.0 对背压的操作

### 延迟发送数据
如果可以的话，使被观察者适当的延迟发送数据。
### 过滤限流
使用过滤限流操作符来处理。
这里简单说几种
* Sample  在一段时间内，只处理最后一个数据
* throttleFirst   在一段时间内只响应第一次的操作
* filter  自己写过滤条件

更多过滤操作符可以查看 [ RxJava 2.x 使用详解(三) 过滤操作符 ](https://maxwell-nc.github.io/android/rxjava2-3.html)

### 缓存
主要包括一下两个操作符
* buffer  将多个事件打包放入一个List中，再一起发射
* window  将多个事件打包放入一个Observable中，再一起发射

### 使用背压操作符
通过一些背压操作符，来转化成支持背压的Observable
* onBackpressurebuffer() 把Observable 发送过来的事件做缓存，当Request方法被调用的时候，给下层流发送一个item,如果给设置了缓存区的大小，那么超过这个缓冲区大小就会抛出异常
* onBackpressureDrop()  将Observable 的事件抛弃掉，直到Subscriber再次调用 request(n)方法的时候，就会发送给它这之后的n个事件
* onBackpressureLatest()  只保留最新的事件，

## Rxjava 2.0 对背压的操作
Rxjava 2.0 的 Flowable 支持背压，
```java
Flowable.create((FlowableOnSubscribe<String>) emitter -> {
            emitter.onNext("test1");
            emitter.onNext("test2");
            emitter.onComplete();
        }, BackpressureStrategy.ERROR).subscribe(new Subscriber<String>() {
            private Subscription subscription;

            @Override
            public void onSubscribe(Subscription s) {
                Log.d("hoyouly", "Flowable : onSubscribe " + s);
                subscription = s;
                subscription.request(1);
            }

            @Override
            public void onNext(String s) {
                Log.d("hoyouly", "Flowable : onNext " + s);
            }

            @Override
            public void onError(Throwable t) {
                t.printStackTrace();
                Log.d("hoyouly", "Flowable : onError " + t.getMessage());
            }

            @Override
            public void onComplete() {
                Log.d("hoyouly", "Flowable : onComplete ");
            }
        });
// 结果
hoyouly : Flowable : onSubscribe 0
hoyouly : Flowable : onNext test1
hoyouly : Flowable : onNext test2
hoyouly : Flowable : onComplete
```
Subscription  和 Disposable 一样，也是一个开关，调用 cancel()就可以切断上下游直接的关系。
，通过调用 request() 来控制流速

注意 request()一定要在 onStart()中通知被观察者发送第一个事件，并且在onNext()最后通知被观察者发送下一个事件。否则就会出现著名的 MissingBackpressureException

如果 onSubscribe() 中没有执行 request() ,输出的结果如下

```java
hoyouly : Flowable : onSubscribe 0
hoyouly : Flowable : onError create: could not emit value due to lack of requests
```

如果 onSubscribe() 中执行了 request()，但是 onNext()中没有执行 request(),那么输出的结果如下

```java
hoyouly : Flowable : onSubscribe 0
hoyouly : Flowable : onNext test1
hoyouly : Flowable : onError create: could not emit value due to lack of requests
```
onNext 执行了一次，这一次是因为  onSubscribe()中的 request(),
### request（）
request() 就是开启循环的钥匙，

记得我们通过 handler 发送循环消息的时候，首先通过 handler.sendMessage(),然后在 handleMessage()中，处理完Message，还会执行 handler.sendMessage()，这样才能循环执行。

这个 request()就相当于 sendMessage(), handleMessage()相当于 onNext(),这样是不是就容易理解了。

request()同时是一种能力，告诉上游能处理几个，而被观察者一直发送数据。

### Flowable 的新思想
RxJava的观察者模型中，被观察者是主动的推送数据给观察者，观察者是被动接收，

而 Flowable 刚好相反，是响应式拉取 观察者主动从被观察者那里拉取数据，而被观察者变成被动的等待通知发送数据。

![okio](../../../../images/request.png)

观察者可以根据自身实际情况按需拉取数据，而不是被动接收（也就相当于告诉上游观察者把速度慢下来），最终实现了上游被观察者发送事件的速度的控制，实现了背压的策略。



---
搬运地址：

[关于RxJava最友好的文章](https://juejin.im/post/580103f20e3dd90057fc3e6d)

[关于RxJava最友好的文章——背压（Backpressure）](https://juejin.im/post/582d413c8ac24700619cceed)

[关于 RxJava 最友好的文章—— RxJava 2.0 全新来袭](https://juejin.im/post/582b2c818ac24700618ff8f5)

[RxJava 2.x 使用详解系列](https://maxwell-nc.github.io/)

[给初学者的RxJava 2.0教程系列](https://www.jianshu.com/u/c50b715ccaeb)
