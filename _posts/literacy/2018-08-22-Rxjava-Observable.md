---
layout: post
title: 扫盲系列 - RxJava 2.0 -- 观察者模式
category: 扫盲系列
tags:  RxJava
---
<!-- * content -->
<!-- {:toc} -->


RxJava 其中 Rx 是 ReactiveX 的缩写， ReactiveX 又是 Reactive Extensions 的缩写   
所以 RxJava 就是 java 上异步和基于事件响应式编程

RxJava 基于观察者模式，主要包括 观察者，被观察者，订阅，事件。   

观察者模式主要分以下几种
* Observable 和 Observer
* Flowable 和 Subscriber
* Single 和 SingleObserver
* Completable 和 CompletableObserver
* Maybe 和 MaybeObserver

## Observeable 和 Observer

例子如下
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
            @Override//Disposable 一次性，理解为订阅事件的开关，调用 dispose() 方法 方法后，会切断联系
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

* ObservableEmitter 用来发送事件的，可以发送三种类型事件， next 事件、complete事件和 error 事件
  * 上游可以无限的发送 onNext ,下游可以无限的接收 onNext
  * 当上游发送了一个 onComplete 后，可以继续发送发送其他事件，但是下游只可以收到这个 onComplete ，后续的事件就收不到了
  * 当上游发送了一个 onError 后，可以继续发送发送其他事件，但是下游只可以收到这个 onError ，后续的事件就收不到了
  * 上游可以不发送 onError 或者 onComplete
  * onError 或者 onComplete 必须唯一并且互斥，即不能发送多个 onComplete 也不能发送多个 onError ，也不能先发一个 onError ，在发送一个 onComplete

* Disposable  订阅事件的开关，调用 dispose() 方法 方法后，会切断联系
  如上所示， onNext 2 之后，调用了 dispose ，下游就收不到数据了，但是上游却还在发送数据。

subscribe() 有很多重载方法
![](../../../../images/observer_subscribe.png)

这里面看到了又是 Action ，又是 Consumer ，这两个区别，和 Observer 又是啥关系呢？

### Action  和 Consumer
* Action 无参数类型  
* Consumer<T>  单一参数类型。
* BigConsumer<T,R>  双参数类型，
* Consumer<Object[]> 多参数类型

这里其实最终都是封装成一个 Observer 对象。只不过使用 Action 和 Consumer 简化观察者而已。

## Flowable 和 Subscriber
```java
//被观察者
Flowable<String> flowable = Flowable.create(new FlowableOnSubscribe<String>() {
    @Override
    public void subscribe(FlowableEmitter<String> emitter) throws Exception {
        emitter.onNext("test1");
        emitter.onNext("test2");
        emitter.onComplete();
    }
}, BackpressureStrategy.BUFFER);

flowable.subscribe(new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {

    }
}, new Consumer<Throwable>() {
    @Override
    public void accept(Throwable throwable) {

    }
}, new Action() {//相当于 onComplete ,
    @Override
    public void run() {

    }
}, new Consumer<Subscription>() {//相当于onSubscribe
    @Override
    public void accept(Subscription subscription) throws Exception {

    }
});
```
这里就使用了 Consumer 和 Action ，其实里面还是把这四个封装成了一个 Subscriber 对象，使用 Consumer 和 Action 只不过是简化观察者而已。

同理， subscribe() 有多重重载方法，如下
![](../../../../images/flower_subscribe.png)

### 五种背压策略
Flowable 支持背压，默认的背压策略是 128 。所以 Flowable 的 create() 比 Observable 的多了一个参数，BackpressureStrategy.BUFFER，看着像是背压策略，一共有五种。
* BackpressureStrategy.MISSING   没有指定背压策略，需要下游操作符指定背压策略
* BackpressureStrategy.DROP  如果 Flowable 的异步缓存池满了，则会丢掉将要放入缓存池中的数据
* BackpressureStrategy.LATEST  如果缓存池满了，会丢掉将要放入缓存池中的数据。这一点与 DROP 策略一样，不同的是，不管缓存池的状态如何， LATEST 策略会将最后一条数据强行放入缓存池中。
* BackpressureStrategy.BUFFER  Flowable的异步缓存池同 Observable 的一样，没有固定大小，可以无限制添加数据，不会抛出 MissingBackpressureException 异常，但会导致OOM.
* BackpressureStrategy.ERROR   如果放入 Flowable 的异步缓存池中的数据超限(默认是128)了，则会抛出 MissingBackpressureException 异常

### Observeable/Observer 与 Flowable/Subscriber 的区别

Observeable 用于 订阅 Observer ，不支持背压 ，
使用场景：
* 不超过 1000 个元素，随着时间的流逝，基本不会出现 OOM ，
* GUI事件或者 1000Hz 频率以下的元素，
* 平台不支持Java Stream (Java 8 新特性)， Observable 的开销比 Flowable 小

Flowable 用于订阅 Subscriber ，支持背压，
使用场景：
* 超过 10K 的元素，
* 读取硬盘操作，
* 通过 JDBC 读取数据库，
* 网络 IO 操作

## Single 和 SingleObserver
单一的连续事件流，即只有一个 onNext() 事件，接着就触发 onComplete() 或者 onError() , 可以使用 Single
只包含两个事件，一个是正常处理成功的 onSuccess ,一个是处理失败的 onError() ,它只发送一次，所以不存在背压问题。

```java
Single single = Single.create(new SingleOnSubscribe<String>() {
            @Override
            public void subscribe(SingleEmitter<String> emitter) throws Exception {
                emitter.onSuccess("success");
                emitter.onSuccess("again");//错误，重复调用也不会执行。
            }
        });

        single.subscribe(new SingleObserver<String>() {
            @Override
            public void onSubscribe(Disposable d) {
            }

            @Override
            public void onSuccess(String o) {
                //相当于 onNext 和 onCompelete()
                Log.d("hoyouly", "onSuccess " + o);
            }

            @Override
            public void onError(Throwable e) {
            }
        });
```
也可以使用 Actions 简化,就是使用的 BiConsumer 这个带有两个参数的Actions

```java
//使用 BiConsumer 简化
single.subscribe(new BiConsumer<String, Throwable>() {
    @Override
    public void accept(String o, Throwable o2) throws Exception {

    }
});
```
可以直接转换成 Flowable 或者 Observable
```java
single.toFlowable();
single.toObservable();
```

## Completable 和 CompletableObserver
不关心 onNext() ,只有 onComplete() 和onError()
```java
Completable.create(emitter -> {
           emitter.onComplete();//单一的 onComplete 事件

       }).subscribe(new CompletableObserver() {
           @Override
           public void onSubscribe(Disposable d) {

           }

           @Override
           public void onComplete() {
               Log.d("hoyouly", "Completable : onComplete ");
           }

           @Override
           public void onError(Throwable e) {
               Log.d("hoyouly", "Completable : onError ");
           }
       });
```
其他的 和 Single 类似，可以通过 Actions 简化观察者，也可以转成 Flowable 或者 Observable

## Maybe 和 MaybeObserver
可能发送一个需求，也可能不发送需求，就可以使用 这个。是 Single 和 Completable 的混合体，可能调用一下其中一种情况
* onSuccess()或者onError()
* onComplete() 或者 onError()

注意： onSuccess()和 onComplete() 是互斥的存在

```java
//判断是否登陆
Maybe.just(isLogin())
    //可能涉及到 IO 操作，放在子线程
    .subscribeOn(Schedulers.newThread())
    //取回结果传到主线程
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new MaybeObserver<Boolean>() {
            @Override
            public void onSubscribe(Disposable d) {
            }

            @Override
            public void onSuccess(Boolean value) {
                if(value){
                    ...
                }else{
                    ...
                }
            }

            @Override
            public void onError(Throwable e) {
            }

            @Override
            public void onComplete() {
            }
        });

```
执行 onSuccess() 就不会执行 onComplete() ,同理，执行 onComplete() 就肯定不会执行 onSuccess().
感觉这个观察者有点扯，反正我是不会用的。太不确定性了。





---
搬运地址：

[关于 RxJava 最友好的文章](https://juejin.im/post/580103f20e3dd90057fc3e6d)

[关于 RxJava 最友好的文章——背压（Backpressure）](https://juejin.im/post/582d413c8ac24700619cceed)

[关于 RxJava 最友好的文章—— RxJava 2.0 全新来袭](https://juejin.im/post/582b2c818ac24700618ff8f5)

[RxJava 2.x 使用详解系列](https://maxwell-nc.github.io/)

[给初学者的RxJava 2.0教程系列](https://www.jianshu.com/u/c50b715ccaeb)

[RxJava2实战--第八章 RxJava 的背压](https://www.cnblogs.com/wangjiaghe/p/11890867.html)
