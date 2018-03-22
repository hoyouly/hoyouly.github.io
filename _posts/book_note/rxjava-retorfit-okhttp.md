---
layout: post
title: Rxjava,Retorfit和Okhttp简介
category: 读书笔记
tags: Rxjava ,Retorfit, Okhttp
---
## Rxjava
异步，实现异步操作的库。异步实现，是通过一种扩展的观察者模式来实现。

好处：简洁,随着程序逻辑越来越复杂，它依旧能保持简洁，简洁到无论多么什么复杂的逻辑都能串成一条链

四个基本概念
1. Observable 可观测者，即被观察者
2. Observer 观察者
3. subcribe 订阅事件

### Observer 观察者
决定事件触发的时候将有怎么样的行为,是一个泛型接口，实现方式
```Java
Observer<String> observable=new Observer<String>(){
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(String s) {

            }
        };

```
Observable 和 Observer 通过 subscribe() 方法实现订阅关系，从而 Observable 可以在需要的时候发出事件来通知 Observer。
与传统观察者模式不同， RxJava 的事件回调方法除了普通事件 onNext() （相当于 onClick() / onEvent()）之外，还定义了两个特殊的事件：onCompleted() 和 onError()。
* onCompleted(): 事件队列完结。RxJava 不仅把每个事件单独处理，还会把它们看做一个队列。RxJava 规定，当不会再有新的 onNext() 发出时，需要触发 onCompleted() 方法作为标志。
* onError(): 事件队列异常。在事件处理过程中出异常时，onError() 会被触发，同时队列自动终止，不允许再有事件发出。
* 在一个正确运行的事件序列中, onCompleted() 和 onError() 有且只有一个，并且是事件序列中的最后一个。需要注意的是，onCompleted() 和 onError() 二者也是互斥的，即在队列中调用了其中一个，就不应该再调用另一个。


Subscriber 是Observe的抽象类，对 Observer 接口进行了一些扩展，但他们的基本使用方式是完全一样的：
```Java
public abstract class Subscriber<T> implements Observer<T>, Subscription {
  protected Subscriber() {
         this(null, false);
     }
     protected Subscriber(Subscriber<?> subscriber) {
         this(subscriber, true);
     }
     protected Subscriber(Subscriber<?> subscriber, boolean shareSubscriptions) {
         this.subscriber = subscriber;
         this.subscriptions = shareSubscriptions && subscriber != null ? subscriber.subscriptions : new SubscriptionList();
     }
     public final void add(Subscription s) {
         subscriptions.add(s);
     }

     @Override
     public final void unsubscribe() {
         subscriptions.unsubscribe();
     }

     @Override
     public final boolean isUnsubscribed() {
         return subscriptions.isUnsubscribed();
     }
     public void onStart() {
         // do nothing by default
     }

}
public interface Subscription {
    void unsubscribe();
    boolean isUnsubscribed();
}
```
从源码可知，Subscriber 好像没干什么事情，只是简单的桥接了一下，创建了一个SubscriptionList对象，剩下的要么是把工作交给这个SubscriptionList对象subscriptions。例如unsubscribe()，isUnsubscribed()以及add(Subscription s)方法，要么就是空实现。比如onStart()，要么就是没实现。例如就是Observer中的三个方法。

实质上，在 RxJava 的 subscribe 过程中，Observer 也总是会先被转换成一个 Subscriber 再使用。主要区别有以下两点：
1. onStart(): 这是 Subscriber 增加的方法。它会在 subscribe 刚开始，而事件还未发送之前被调用，可以用于做一些准备工作，例如数据的清零或重置。这是一个可选方法，默认情况下它的实现为空。需要注意的是，如果对准备工作的线程有要求（例如弹出一个显示进度的对话框，这必须在主线程执行）， **onStart() 就不适用了，因为它总是在 subscribe 所发生的线程被调用，而不能指定线程。要在指定的线程来做准备工作，可以使用 doOnSubscribe() 方法，** 具体可以在后面的文中看到。
2. unsubscribe(): 这是 Subscriber 所实现的另一个接口 Subscription 的方法，用于取消订阅。在这个方法被调用后，Subscriber 将不再接收事件。一般在这个方法调用前，可以使用 isUnsubscribed() 先判断一下状态。 unsubscribe() 这个方法很重要，因为在 subscribe() 之后， Observable 会持有 Subscriber 的引用，这个引用如果不能及时被释放，将有内存泄露的风险。所以最好保持一个原则：要在不再使用的时候尽快在合适的地方（例如 onPause() onStop() 等方法中）调用 unsubscribe() 来解除引用关系，以避免内存泄露的发生。

 RxJava 的这个实现，是一条从上到下的链式调用，没有任何嵌套，这在逻辑的简洁性上是具有优势的。当需求变得复杂时，这种优势将更加明显

### Observable
使用Rxjava 的creat()方法来创建一个Observable对象，并为他定义事件的触发规则

```Java
//当 Observable 被订阅的时候，OnSubscribe 的 call() 方法会自动被调用，事件序列就会依照设定依次触发.
//由被观察者调用了观察者的回调方法，就实现了由被观察者向观察者的事件传递，即观察者模式。
Observable<String> observable=Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
              //观察者Subscriber 将会被调用三次 onNext() 和一次 onCompleted()
                subscriber.onNext("Hello");
                subscriber.onNext("Hi");
                subscriber.onNext("RxJava");
                subscriber.onCompleted();
            }
        });

      public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
                // cover for generics insanity
            }
      public interface Action1<T> extends Action {
                void call(T t);
            }
      public interface Action extends Function {
            //空接口
            }
      public interface Function {
        //空接口
        }
```
creat()是RxJava 最基本的创建事件序列的方法，基于这个方法，RxJava还创建了一系列方法用来快速创建事件序列
```java
// 方法一
Observable observable = Observable.just("Hello", "Hi", "Aloha");
//方法二
String[] words = {"Hello", "Hi", "Aloha"};
Observable observable = Observable.from(words);

```
上面两种方式和creat()方式效果一样的，都是一次执行onNext("Hello"),onNext("Hi"),onNext("RxJava")以及onComleted()
原因看源码
```Java
public class Observable<T> {
  ...
  // creat()方式
  public static <T> Observable<T> create(OnSubscribe<T> f) {
         return new Observable<T>(hook.onCreate(f));
     }

     // from（）方式
     public static <T> Observable<T> from(T[] array) {
        int n = array.length;
        if (n == 0) {
            return empty();
        } else
        if (n == 1) {
            return just(array[0]);
        }
        //public final class OnSubscribeFromArray<T> implements OnSubscribe<T> {
        return create(new OnSubscribeFromArray<T>(array));
    }

    // just()方式，
     public static <T> Observable<T> just(T t1, T t2, T t3) {
        return from((T[])new Object[] { t1, t2, t3 });
    }

```
其实就是just()方式，把参数集合封装成一个数组，然后执行from()方法，然后from（）方法又把这个数组转换成OnSubscribeFromArray对象，也就是OnSubcribe对象，执行creat()方法

### subscribe 订阅
创建了Observer和Observable之后，再用subscribe()方法，把他们连接起来，整条链子就可以工作了，代码形式很简单
```Java
    observable.subscribe(observer);
    //或者
    observable.subscribe(sub);
```
源码能告诉我们一切
```Java
// Observable<T>
  ···
public final Subscription subscribe(final Observer<? super T> observer) {
   if (observer instanceof Subscriber) {
       return subscribe((Subscriber<? super T>)observer);
   }
   return subscribe(new ObserverSubscriber<T>(observer));
}

```
当参数是Observer的时候，会判断是不是Subscribe对象，是的话就直接强转，然后执行subscribe(Subscriber<? super T> subscriber)方法。如果不是的话，就把它封装成一个Subscribe的子类对象，然后执行subcribe(Subscriber<? super T> subscriber)方法，也就是无论如何，Observer 也总是会先被转换成一个 Subscriber 再使用，接下来咱们看看 subscribe(Subscriber<? super T> subscriber) 方法是如何实现的

```Java
// Observable<T>
static final RxJavaObservableExecutionHook hook = RxJavaPlugins.getInstance().getObservableExecutionHook();

public final Subscription subscribe(Subscriber<? super T> subscriber) {
      return Observable.subscribe(subscriber, this);
  }

static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
      ···
       subscriber.onStart();
       ···
       try{
         //参照下面的源码可知，hook.onSubscribeStart(observable, observable.onSubscribe) 得到的结果就是observable.onSubscribe
       hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
          //参照下面的源码可知，得到的结果就是subscriber本身
          return hook.onSubscribeReturn(subscriber);
        }catch (Throwable e) {
          ···
          subscriber.onError(hook.onSubscribeError(e));
          ···
        }
}

//RxJavaObservableExecutionHook.Java
public <T> OnSubscribe<T> onSubscribeStart(Observable<? extends T> observableInstance, final OnSubscribe<T> onSubscribe) {
      // pass through by default
      return onSubscribe;
  }

public <T> Subscription onSubscribeReturn(Subscription subscription) {
         // pass through by default
         return subscription;
     }  

```
根据源码可知：subscriber()做了3件事
1. 执行了Subscribe的onStart()方法，这是一个可选的准备方法
2. 因为 hook.onSubscribeStart(observable, observable.onSubscribe)返回的是observable.onSubscribe对象，也就是说执行了Observable中onSubscibe的onCall()方法，从这里，事件发送的逻辑开始执行，从这里看出，在RxJava中,Observable并不是在一开始创建的时候就开始发送事件，而是在执行subscribe()方法执行的时候，
3. 根据RxJavaObservableExecutionHook.java 源码可知，hook.onSubscribeReturn(subscriber)得到的结果还是subscriber本身，也就是将Subscriber对象作为Subscription返回，这是为了方便unsubscrible()

### subscribe()不完整定义
除了 subscribe(Observer) 和 subscribe(Subscriber) ，subscribe() 还支持不完整定义的回调，RxJava 会自动根据定义创建出 Subscriber 。


搬运：
[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)

## Retorfit

retrofit封装了okhttp,okhttp使用的还是 httpurlconnection.

## Okhttp


摘抄：

[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)
