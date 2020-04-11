---
layout: post
title: 扫盲系列 - RxJava 2.0 -- subscribeOn 和 observeOn
category: 扫盲系列
tags:  RxJava
---
* content
{:toc}

我们都知道   
* subscribeOn()  是指定上游发送事件的线程   
* observeOn()    指定下游接收事件的线程

他们之间可以多种组合，可是会有啥效果呢，实践出真知。那就来做几个试验吧。

## 试验一
正常的情况，一个 subscribeOn 和 一个 observeOn ，并且 subscribeOn 上 observeOn 下
```java
Observable
      .create((ObservableOnSubscribe<Integer>) e -> {
          Log.d(TAG, "create  currentThread: " + Thread.currentThread().getName());
          e.onNext(1);
      })
      //上游发送事件在 IO 线程中
      .subscribeOn(Schedulers.io())
      //下游接受事件在主线程中
      .observeOn(AndroidSchedulers.mainThread())
      .subscribe(integer -> Log.d(TAG, "subscribe accept: "Thread.currentThread().getName() + "   value:" + integer));
//结果
hoyouly : create  currentThread: RxCachedThreadScheduler-1
hoyouly : subscribe accept: main   value:1

```
通过结果可以看出了    
subscribeOn() 上面的 create() 在 子线程 RxCachedThreadScheduler-1 中执行， RxCachedThreadScheduler 是 Schedulers.io 线程   
observeOn() 下面的 subscribe() 在主线程 main 中执行   

## 试验二
如果使用两个 subscribeOn() ，并且设置不同的线程，会怎么样呢？那个才会生效呢，
```java
Observable
    .create((ObservableOnSubscribe<Integer>) e -> {
        Log.d(TAG, "create  currentThread: " + Thread.currentThread().getName());
        e.onNext(1);
    })
    //上游发送事件在 IO 线程中
    .subscribeOn(Schedulers.io())
    //上游发送事件在新线程中
    .subscribeOn(Schedulers.newThread())
    //下游接受事件在主线程中
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(integer -> Log.d(TAG, "subscribe accept: " + Thread.currentThread().getName() + "   value:" + integer));
//结果
hoyouly : create  currentThread: RxCachedThreadScheduler-1
hoyouly : subscribe accept: main   value:1
```
create() 在 子线程 RxCachedThreadScheduler-1 中执行，好像 Schedulers.newThread()没起作用。


难道和顺序有关，两个调换一下。

```java
Observable
    .create((ObservableOnSubscribe<Integer>) e -> {
       Log.d(TAG, "create  currentThread: " + Thread.currentThread().getName());
       e.onNext(1);
    })
    ////上游发送事件在新线程中
    .subscribeOn(Schedulers.newThread())
    //上游发送事件在 IO 线程中
    .subscribeOn(Schedulers.io())
    //下游接受事件在主线程中
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(integer -> Log.d(TAG, "subscribe accept: " + Thread.currentThread().getName() + "   value:" + integer));
//结果
hoyouly : create  currentThread: RxNewThreadScheduler-2
hoyouly : subscribe accept: main   value:1       
```
这次 create() 在 RxNewThreadScheduler-2 中执行，Schedulers.io() 有不起作用了

真的和调用顺序有关，后面的不起作用吗？    
会不会因他们两个同时设置一个 creat() ,总的有一个失效的。    
如果 两个 subscribeOn() 中间间隔一个操作符，那么这个操作符会不会就在另外一个线程中执行呢。

再来一个实验。这一次 subscribeOn() 中间加一个 map() 操作符。

## 试验三

```java
Observable
    .create((ObservableOnSubscribe<Integer>) e -> {
        Log.d(TAG, "create  currentThread: " + Thread.currentThread().getName());
        e.onNext(1);
    })
    //上游发送事件在 IO 线程中
    .subscribeOn(Schedulers.io())
    .map(integer -> {
        Log.d("hoyouly", "map :  currentThread " + Thread.currentThread().getName() + "   value:" + integer);
        return "map_"+integer;
    })
    .subscribeOn(Schedulers.newThread())
    //下游接受事件在主线程中
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(integer -> Log.d(TAG, "subscribe currentThread: " + Thread.currentThread().getName() + "   value:" + integer));
//结果
hoyouly : create  currentThread: RxCachedThreadScheduler-1
hoyouly : map    currentThread:  RxCachedThreadScheduler-1   value:1
hoyouly : subscribe   currentThread: main   value:map_1
```
发现 create() 和 map() 都是在 同一个线程 RxCachedThreadScheduler-1 中执行的, Schedulers.newThread() 没其作用。

看来 <font color="#ff000" > subscribeOn() 是先入为主的原则，后面不管你怎么多次设置，都是在这一个线程中执行的。</font>

可是我又想到一种 情况

如果 subscribeOn 在 observeOn 下面执行，那到底谁其作用呢？？
## 试验四

```java
Observable
    .create((ObservableOnSubscribe<Integer>) e -> {
        Log.d(TAG, "create  currentThread: " + Thread.currentThread().getName());
        e.onNext(1);
    })
    //上游发送事件在 IO 线程中
    .subscribeOn(Schedulers.io())
    //下游接受事件在主线程中
    .observeOn(AndroidSchedulers.mainThread())
    .map(integer -> {
        Log.d("hoyouly", "map   currentThread :  " + Thread.currentThread().getName() + "   value:" + integer);
        return "map_" + integer;
    })
    .subscribeOn(Schedulers.newThread())
    .subscribe(integer -> Log.d(TAG, "subscribe currentThread: " + Thread.currentThread().getName() + "   value:" + integer));
//结果
hoyouly : create  currentThread: RxCachedThreadScheduler-1
hoyouly : map   currentThread :  main   value:1
hoyouly : subscribe currentThread: main   value:map_1
```
map 在主线程中执行，难道真的是 observeOn() 的权利更大一些？

还是因为 两个挣一个导致的呢？总的有一个落败，如果设置两个试试，两个 map ，一前一后，是不是就可以均分了呢？
## 试验五
```java
Observable
    .create((ObservableOnSubscribe<Integer>) e -> {
        Log.d(TAG, "create  currentThread: " + Thread.currentThread().getName());
        e.onNext(1);
    })
    //上游发送事件在 IO 线程中
    .subscribeOn(Schedulers.io())
    //下游接受事件在主线程中
    .observeOn(AndroidSchedulers.mainThread())
    .map(integer -> {
        Log.d("hoyouly", "map  前   currentThread :  " + Thread.currentThread().getName() + "   value:" + integer);
        return "map_" + integer;
    })
    .map(s -> {
        Log.d("hoyouly", "map  后  currentThread :  " + Thread.currentThread().getName() + "   value:" + s);
        return "map_" + s;
    })
    .subscribeOn(Schedulers.newThread())
    .subscribe(integer -> Log.d(TAG, "subscribe currentThread: " + Thread.currentThread().getName() + "   value:" + integer));

// 结果
hoyouly : create  currentThread: RxCachedThreadScheduler-1
hoyouly : map  前   currentThread :  main   value:1
hoyouly : map  后  currentThread :  main   value:map_1
hoyouly : subscribe currentThread: main   value:map_map_1

```
两次 map 都是在 main 线程中，看来 均分是不可能了，我全要。

所以可以得出结论<font color="#ff000" > subscribeOn 在 observeOn 之后，不起任何作用。</font>

subscribeOn 是 干不过 observeOn 了。    

## 试验六
很多情况下， subscribeOn 之后紧挨着 observeOn ，才能做到线程完美切换，可是如果在这两个中间有一个操作符，是按照上游，还是按照下游呢

```java
Observable
    .create((ObservableOnSubscribe<Integer>) e -> {
        Log.d(TAG, "create  currentThread: " + Thread.currentThread().getName());
        e.onNext(1);
    })
    //上游发送事件在 IO 线程中
    .subscribeOn(Schedulers.io())
    .map(integer -> {
        Log.d("hoyouly", "map :  currentThread " + Thread.currentThread().getName() + "   value:" + integer);
        return "map_"+integer;
    })
    //下游接受事件在主线程中
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(integer -> Log.d(TAG, "subscribe currentThread: " + Thread.currentThread().getName() + "   value:" + integer));

//结果如下
hoyouly : create  currentThread: RxCachedThreadScheduler-1
hoyouly : map :  currentThread RxCachedThreadScheduler-1   value:1
hoyouly : subscribe currentThread: main   value:map_1
```
map 竟然和 create 在同一个线程中执行，看来 observeOn 很高冷啊，没到我的范围内，我就不管。

那 observeOn 自己和自己较劲？如果执行两个 observeOn() 的话，那个会有效果呢？

## 试验七
```java
Observable
    .create((ObservableOnSubscribe<Integer>) e -> {
        Log.d(TAG, "create  currentThread: " + Thread.currentThread().getName());
        e.onNext(1);
    })
    //上游发送事件在 IO 线程中
    .subscribeOn(Schedulers.io())
    //下游接受事件在主线程中
    .observeOn(AndroidSchedulers.mainThread())
    .observeOn(Schedulers.newThread())
    .subscribe(integer -> Log.d(TAG, "subscribe currentThread: " + Thread.currentThread().getName() + "   value:" + integer));
//结果
hoyouly : create  currentThread: RxCachedThreadScheduler-1
hoyouly : subscribe currentThread: RxNewThreadScheduler-2   value:1
```
subscribe 竟然在 RxNewThreadScheduler 线程中，难道 observeOn() 有后发优势。    
还是因为连续执行的缘故啊，继续来下一个试验。

那如果不是连续执行呢，中间有其他操作符，会是怎么一个情形呢？
## 试验八

```java
Observable
    .create((ObservableOnSubscribe<Integer>) e -> {
        Log.d(TAG, "create  currentThread: " + Thread.currentThread().getName());
        e.onNext(1);
    })
    //上游发送事件在 IO 线程中
    .subscribeOn(Schedulers.io())
    //下游接受事件在主线程中
    .observeOn(AndroidSchedulers.mainThread())
    .map(integer -> {
        Log.d("hoyouly", "map   currentThread :  " + Thread.currentThread().getName() + "   value:" + integer);
        return "map_" + integer;
    })
    .observeOn(Schedulers.newThread())
    .subscribe(integer -> Log.d(TAG, "subscribe currentThread: " + Thread.currentThread().getName() + "   value:" + integer));
//结果
hoyouly : create  currentThread: RxCachedThreadScheduler-1
hoyouly : map   currentThread :  main   value:1
hoyouly : subscribe currentThread: RxNewThreadScheduler-2   value:map_1
```
map 是在 main 线程中执行的，而 subscribe 是在 RxNewThreadScheduler-2 中执行的。

所以呢，<font color="#ff000" > observeOn 指定一次，就会切换一次线程。</font>


## 结论
根据以上几个试验，可以得知
* 多次设置 subscribeOn ，只有第一次生效，先入为主原则
* 多次设置 observeOn 指定一次就会生效一次。连续的两次以后面那次生效。后发优势原则
* subscribeOn 在 observeOn 之后，不起任何作用

---
搬运地址：

[关于 RxJava 最友好的文章](https://juejin.im/post/580103f20e3dd90057fc3e6d)

[关于 RxJava 最友好的文章——背压（Backpressure）](https://juejin.im/post/582d413c8ac24700619cceed)

[关于 RxJava 最友好的文章—— RxJava 2.0 全新来袭](https://juejin.im/post/582b2c818ac24700618ff8f5)

[RxJava 2.x 使用详解系列](https://maxwell-nc.github.io/)

[给初学者的RxJava 2.0教程系列](https://www.jianshu.com/u/c50b715ccaeb)
