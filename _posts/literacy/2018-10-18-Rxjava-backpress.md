---
layout: post
title: 扫盲系列 - RxJava 2.0 -- 背压
category: 扫盲系列
tags:  RxJava 2.0
---
* content
{:toc}

```java
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override//Emitter 发射器，用来发送事件的，
            public void subscribe(ObservableEmitter<Integer> e) throws Exception {
                Log.d(TAG, "subscribe:1 ");
                e.onNext(1);
                Log.d(TAG, "subscribe:2 ");
                e.onNext(2);
                Log.d(TAG, "subscribe:3 ");
                e.onNext(3);
                Log.d(TAG, "subscribe:onComplete ");
                e.onComplete();
            }
        }).subscribe(new Observer<Integer>() {
            private Disposable mDisposable;
            private int i;
            @Override//Disposable 一次性，理解为订阅事件的开关，调用dispose()方法 方法后，会切断联系
            public void onSubscribe(Disposable d) {
                Log.d(TAG, "onSubscribe: ");
                mDisposable = d;
            }
            @Override
            public void onNext(Integer value) {
                Log.d(TAG, "onNext: " + value);
                i++;
                if (i == 2) {
                    Log.d(TAG, "onNext: dispose");
                    mDisposable.dispose();
                    Log.d(TAG, "onNext: isDispose: " + mDisposable.isDisposed());
                }
            }
            @Override
            public void onError(Throwable e) {
                Log.d(TAG, "onError: " + e.getMessage());
            }
            @Override
            public void onComplete() {
                Log.d(TAG, "onComplete: ");
            }
        });

//结果
D hoyouly : smapleTest2:
D hoyouly : onSubscribe:
D hoyouly : subscribe:1
D hoyouly : onNext: 1
D hoyouly : subscribe:2
D hoyouly : onNext: 2
D hoyouly : onNext: dispose
D hoyouly : onNext: isDispose: true
D hoyouly : subscribe:3
D hoyouly : subscribe:onComplete         
```

* ObservableEmitter 用来发送事件的，可以发送三种类型事件，next事件、complete事件和error事件
  * 上游可以无限的发送 onNext ,下游可以无限的接收 onNext
  * 当上游发送了一个 onComplete 后，可以继续发送发送其他事件，但是下游只可以收到这个 onComplete，后续的事件就收不到了
  * 当上游发送了一个 onError 后，可以继续发送发送其他事件，但是下游只可以收到这个 onError，后续的事件就收不到了
  * 上游可以不发送 onError 或者 onComplete
  * onError 或者 onComplete 必须唯一并且互斥，即不能发送多个 onComplete 也不能发送多个 onError，也不能先发一个 onError，在发送一个 onComplete

* Disposable  订阅事件的开关，调用 dispose()方法 方法后，会切断联系
  如上所示，onNext 2 之后，调用了dispose，下游就收不到数据了，但是上游却还在发送数据。


## backpressure 背压
主要目的是控制流量的，水缸的控制能力毕竟有限，因此还得从源头控制。

背压是指在异步场景中，被观察者发送事件速度远远快于观察者的处理速度的情况下，一种告诉上游的被观察者降低发送速度的策略
背压是流速控制的一种策略
* 背压的一个前提是异步，也就是说观察者和被观察者在不同的线程。如果 观察者和被观察者在同一个线程中，这个时候被观察者发送的事件，必须等到观察者接收处理完以后才能发送下一个事件
* 背压并是不像flatmap一样可以在程序中直接使用的操作符，他只是一种控制事件流速的策略



### 背压的策略


1. 对于 支持 背压的操作符，可以使用 request() 方法，响应式拉取

RxJava的观察者模型中，被观察者是主动的推送数据给观察者，观察者是被动接收，
响应式拉取 刚好相反，观察者主动从被观察者那里拉取数据，而被观察者变成被动的等待通知发送数据。

![okio](../../../../images/request.png)

注意  request()一定要在 onStart()中通知被观察者发送第一个事件，并且在onNext()最后通知被观察者发送下一个事件。

2. 不支持背压的Observable 控制流速的方式
* 过滤，抛弃，类似的操作符 Sample ,ThrottleFrist,filter等
* 缓存， 类似的操作符 buffer,window,
* 被观察者适当的延迟发送，
3. 两个特殊的操作符

* onBackpressurebuffer() 把Observable 发送过来的事件做缓存，当Request方法被调用的时候，给下层流发送一个item,如果给设置了缓存区的大小，那么超过这个缓冲区大小就会抛出异常
* onBackpressureDrop()  将Observable 的事件抛弃掉，直到Subscriber再次调用 request(n)方法的时候，就会发送给它这之后的n个事件
* onBackpressureLatest()  只保留最新的事件，

使用这两个操作符，可以让不支持背压操作的Observable 支持背压

Flowable 默认有一个 128 大小的缓存

BackpressureStrategy.DROP  直接把存不下的事件丢弃掉
BackpressureStrategy.LATEST  保留最新的事件
BackpressureStrategy.BUFFER   和 Observable 一样了，
BackpressureStrategy.ERROR   上下游流速不均衡的时候直接抛出一个异常

---
搬运地址：

[关于RxJava最友好的文章](https://juejin.im/post/580103f20e3dd90057fc3e6d)

[关于RxJava最友好的文章——背压（Backpressure）](https://juejin.im/post/582d413c8ac24700619cceed)

[关于 RxJava 最友好的文章—— RxJava 2.0 全新来袭](https://juejin.im/post/582b2c818ac24700618ff8f5)

[RxJava 2.x 使用详解系列](https://maxwell-nc.github.io/)

[给初学者的RxJava 2.0教程系列](https://www.jianshu.com/u/c50b715ccaeb)
