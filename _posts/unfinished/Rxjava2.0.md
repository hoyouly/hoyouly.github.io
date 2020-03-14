
1. Disposadle 一次性，用来切断上下游的联系 ，这个是Observer中设置的
2. ObservableEmitter 中的onComplete和onError()也可以用来切断上下游的联系，这个是Obserable中的设置的

这两种方法都是发送后，下游不再接受，但是上游还是会继续放松的

## zip
通过一个函数将多个Observable发送的事件接合到一起，然后发送这些组合到一起的时间，他按照严格的顺序应用这些函数。

## backpressure 背压
主要目的是控制流量的，水缸的控制能力毕竟有限，因此还得从源头控制




Observable      Observer
被观察者         观察者

View           OnClickListener     

用 setOnClickListener(),让被观察者View 持有观察者OnClickListener 的对象，   当View 监听到事件时边通知OnClickListener 进行处理。


Observable.just("").subscribe(new Observer<String>() {})  

用 subscribe() ,让被观察者Observable 持有观察者 Observer对象，当 Observable 监听到事件边通知 Observer 进行处理

这样就简单的实现了数据量从 被观察者 到 观察者的传递


ObservableFromIterable      

ObservableFlatMap  


subscribe         LambdaObserver


## FlatMap
简单来说就是把 被观察者的每次发射出来的事件，转换成一个子被观察者，然后通过合并（Merge）所有子被观察者的事件成总的一系列事件发送给观察者。

Rxjava 使用观察者模式，封装了很多观察者Obserever 和 被观察者 Observable,针对不同操作符，都有对应的Observer 和 Observable
例如
concat() 中的  ObservableConcatMap  和 SourceObserver
subscribeOn()  中的ObservableSubscribeOn  和 SubscribeOnObserver
observeOn()   ObservableObserveOn  和  ObserveOnObserver
flatMap()  中的 ObservableFlatMap 和 MergeObserver 以及InnerObserver
creat() 中的 ObservableCreate ，Observer 需要自定义才行


```
