---
layout: post
title: 扫盲系列之---RxJava2.0 注册登录顺序
category: 扫盲系列
tags: RxJava2.0
---
* content
{:toc}


在看 [给初学者的RxJava2.0教程(三)](https://www.jianshu.com/p/128e662906af) 的时候，发现最后关于注册后登陆的例子，作者写的有些不详细。

如果注册失败了，该怎么处理？这个时候怎么把事件停下来，看了后面的评论，也都是在问这个，也有哥们给出来了，自己就按照他们写的总结了一下。

可以说有三种方法处理这个问题。

### 使用 Observable.empty()
```java
private void flatMapTest(final int value) {
    Log.d(TAG, "flatMapTest() :   value " + value);
    Observable.create(new ObservableOnSubscribe<Integer>() {
                @Override
                public void subscribe(ObservableEmitter<Integer> emitter) {
                  Log.d(TAG, "subscribe: " + Thread.currentThread().getName());
                  emitter.onNext(value);
                }
            })
            .subscribeOn(Schedulers.io())//在子线程中执行上面的代码
            .observeOn(AndroidSchedulers.mainThread())//下面的代码切换到主线程中执行
            .doOnNext(new Consumer<Integer>() {
                @Override
                public void accept(Integer integer) throws Exception {
                    Log.d(TAG, "doOnNext accept: " + Thread.currentThread().getName() + "  value: " + integer);
                }
            })
            .observeOn(Schedulers.io())//下面的代码再切换到子线程中执行。
            .flatMap(new Function<Integer, ObservableSource<String>>() {
                @Override
                public ObservableSource<String> apply(Integer integer) {
                  //把每一个上游发送过来的事件转换成一个新的三个String事件，不保证事件顺序
                    Log.d(TAG, "flatMap apply: " + Thread.currentThread().getName() + "  " + integer);
                    if (integer % 2 == 0) {//执行下一步操作
                        final List<String> list = new ArrayList<>();
                        for (int i = 0; i < 2; i++) {
                            list.add("I am value :" + integer);
                        }
                        return Observable.fromIterable(list).delay(3, TimeUnit.SECONDS);
                    } else {
                        Log.d(TAG, "停止，不再分发: ");
                        return Observable.empty();
                    }
                }
            })//
            .observeOn(AndroidSchedulers.mainThread())//下面的代码再次切换到主线程中执行
            .subscribe(new Observer<String>() {
                @Override
                public void onSubscribe(Disposable d) {
                    Log.d(TAG, "onSubscribe: " + Thread.currentThread().getName() + "   " + d);
                }

                @Override
                public void onNext(String s) {
                    Log.d(TAG, "onNext: " + Thread.currentThread().getName() + "   " + s);
                }

                @Override
                public void onError(Throwable e) {
                    Log.d(TAG, "onError: " + Thread.currentThread().getName() + "   " + e.getMessage());
                }

                @Override
                public void onComplete() {
                    Log.d(TAG, "onComplete: "+Thread.currentThread().getName());
                }
            });
}
```
最后执行的结果是，如果value 为1的时候，
```java
 flatMapTest() :   value 1
 onSubscribe: main   0
 subscribe: RxCachedThreadScheduler-1
 doOnNext accept: main  value: 1
 flatMap apply: RxCachedThreadScheduler-2  1
 停止，不再分发:
```
后面的onNext()  onError()  onComplete() 三个方法都没执行   
但是如果传递的value是 2的时候，后面的onNext()就能顺利执行
```java
 flatMapTest() :   value 2
 onSubscribe: main   0
 subscribe: RxCachedThreadScheduler-1
 doOnNext accept: main  value: 2
 flatMap apply: RxCachedThreadScheduler-2  2
 onNext: main   I am value :2
 onNext: main   I am value :2
```
### 使用Observable.error()方法

这次只贴了部分代码，也就是不同的地方。
```java
private void flatMapTest(final int value) {
            ...
            .flatMap(new Function<Integer, ObservableSource<String>>() {
                @Override
                public ObservableSource<String> apply(Integer integer) {
                  //把每一个上游发送过来的事件转换成一个新的三个String事件，不保证事件顺序
                    Log.d(TAG, "flatMap apply: " + Thread.currentThread().getName() + "  " + integer);
                    if (integer % 2 == 0) {//执行下一步操作
                        final List<String> list = new ArrayList<>();
                        for (int i = 0; i < 2; i++) {
                            list.add("I am value :" + integer);
                        }
                        return Observable.fromIterable(list).delay(3, TimeUnit.SECONDS);
                    } else {
                        Log.d(TAG, "停止，不再分发: ");
                        return  Observable.error(new Exception("停止分发"));//这个会执行到onError()方法，
                    }
                }
            })//
            ...
}
```
这个唯一的不同就是执行了Observable.error()后，后面会执行onError()方法
```java
 flatMapTest() :   value 1
 onSubscribe: main   0
 subscribe: RxCachedThreadScheduler-1
 doOnNext accept: main  value: 1
 flatMap apply: RxCachedThreadScheduler-2  1
 停止，不再分发:
 onError: main   停止分发

```

### 使用CompositeDisposable来处理

CompositeDisposable  对我们订阅的请求统一管理。 使用步骤
1. 在UI层创建 CompositeDisposable 对象，比如在onCreat()
2. 把subiscribe() 订阅返回的Disposable对象加入管理器
3. 不需要的时候清空订阅对象。

也只贴了核心代码。
这次是在doOnNext()中处理compositeDisposable.clear();

```java
private void flatMapTest(final int value) {
  //创建CompositeDisposable 对象
  final CompositeDisposable compositeDisposable = new CompositeDisposable();
            ...
            .doOnNext(new Consumer<Integer>() {
               @Override
               public void accept(Integer integer) throws Exception {
                   Log.d(TAG, "doOnNext accept: " + Thread.currentThread().getName() + "  value: " + integer);
                   if (integer % 2 == 0) {//执行下一步操作

                   } else {
                       Log.d(TAG, "停止，不再分发: ");
                       compositeDisposable.clear();//不需要的时候清空订阅对象。
                   }
               }
            })//
            .observeOn(Schedulers.io())//下面的代码再切换到子线程中执行。
            .flatMap(new Function<Integer, ObservableSource<String>>() {
               @Override
               public ObservableSource<String> apply(Integer integer) {//把每一个上游发送过来的事件转换成一个新的三个String事件，不保证事件顺序
                   Log.d(TAG, "flatMap apply: " + Thread.currentThread().getName() + "  " + integer);
                   final List<String> list = new ArrayList<>();
                   for (int i = 0; i < 2; i++) {
                       list.add("I am value :" + integer);
                   }
                   return Observable.fromIterable(list).delay(3, TimeUnit.SECONDS);
               }
            })//
            .observeOn(AndroidSchedulers.mainThread())//下面的代码再次切换到主线程中执行
            .subscribe(new Observer<String>() {
               @Override
               public void onSubscribe(Disposable d) {
                   Log.d(TAG, "onSubscribe: " + Thread.currentThread().getName() + "   " + d);
                   compositeDisposable.add(d);//把Disposable对象加入管理器
               }
        ...
```
这个执行结果如下
```java
 flatMapTest() :   value 1
 onSubscribe: main   0
 subscribe: RxCachedThreadScheduler-1
 doOnNext accept: main  value: 1
 停止，不再分发:
```
这个连flatMap()方法都没执行到，推荐使用这种。
