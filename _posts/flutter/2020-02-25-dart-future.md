---
layout: post
title: Flutter学习之Dart 中的 Future
category: Flutter 学习
tags: Flutter  Dart  Future
---
* content
{:toc}

Dart 是单线程的
单线程和异步并不冲突，因为APP在绝大多数情况下都是等待，比如等用户点击，等网络请求返回，等文件IO结果，而这些行为并不是阻塞的，比如说，网络请求，Socket 本身提供了 select 模型可以异步查询；而文件 IO，操作系统也提供了基于事件的回调机制。
所以可以在等待过程中做别的事情，而真正需要相应结果的时候，再去做对应的处理。
因为等待过程并不是阻塞，所以给我的感觉是同时做了很多事情一样。
但其实是一个线程在处理你的事情。

等待这个行为是通过Event Loop 驱动的，
两个事件队列
* Event Queue  事件队列，
* Microtask Queue 微任务队列 ，优先级很高，短时间就完成的异步任务。由scheduleMicroTask 建立，

![](../../../../images/event_loop.png)


每一次事件循环，会先去微任务队列中查询，如果队列不为null，则执行微任务队列中的事件，直到为null，才会执行事件队列里面的任务。  不过很少有事件用在这个队列中。
就连 Flutter 内部，也只有 7 处用到了而已（比如，手势识别、文本输入、滚动视图、保存页面效果等需要高优执行任务的场景）
大多数情况下，我们使用的是 Event Queue ，比如，I/O、绘制、定时器这些异步事件，都是通过事件队列驱动主线程执行的。

Flutter为Event Queue 的任务建立提供了一个封装，就是Future

## Future
单线程，主线程由一个时间来执行，类似Android的主线程，对于异步代码，通过Future来获取结果

Future对象表示异步操作结果，通常通过then()来处理返回的结果

把一个函数放到Future中，就完成了从同步任务到异步任务的封装

声明一个Future时，Dart会把异步任务的函数体放入到事件队列中，然后立即执行。
后续的代码继续执行，而当同步执行的代码执行完毕后，事件队列会按照（FIFO），依次取出事件进行执行，最后同步执行Future的函数体以及后续的 then()


then 与 Future 函数体共用一个事件循环。如果 Future 有多个 then，它们也会按照链式调用的先后顺序同步执行，同样也会共用一个事件循环。如果Future 执行体已经执行完毕了，但你又拿着这个 Future 的引用，往里面加了一个 then 方法体，这时 Dart 会如何处理呢？面对这种情况，Dart 会将后续加入的 then 方法体放入微任务队列，尽快执行。

如果 then 中也是一个Future，那么这个 then，以及后续的 then 都被放入到事件队列中了，
所以
```java
//声明了一个匿名Future，并注册了两个then。第一个then是一个Future
Future(() => print('f6'))
  .then((_) => Future(() => print('f7')))
  .then((_) => print('f8'));

//声明了一个匿名Future
Future(() => print('f9'));
```
这段代码的结构就是
```
f6
f9
f7
f8
```

then 会在 Future 函数体执行完毕后立刻执行，无论是共用同一个事件循环还是进入下一个微任务。

对于异步函数返回的 Future 对象，如果调用者决定同步等待，则需要在调用处使用 await 关键字，并且在调用处的函数体使用 async 关键字。


Dart 中的 await 并不是阻塞等待，而是异步等待 , Dart会把调用体的函数也当做异步函数，将等待异步任务的上下文也放入到Event Queue 中，一旦有了结果，Event Loop 就会把它从 Event Queue 中取出，等待代码继续执行。

```java
//声明了一个延迟2秒返回Hello的Future，并注册了一个then返回拼接后的Hello 2019
Future<String> fetchContent() =>
  Future<String>.delayed(Duration(seconds:2), () => "Hello")
    .then((x) => "$x 2019");
//异步函数会同步等待Hello 2019的返回，并打印
func() async => print(await fetchContent());

main() {
  print("func before");
  func();
  print("func after");
}
```
结果是
```
func before
func after
Hello 2019
```
调用func()函数，里面又调用了异步函数 fetchContent(),同时使用了 await进行等待，因此把fetchContent()和 await 语句的上下文函数 func() 放入事件队列中。
这只把func()函数放入事件队列，可是后面的print()并没有放入，所以会执行.
2 秒后，fetchContent 异步任务返回“Hello 2019”，于是 func 的 await 也被取出，打印“Hello 2019”

await 与 async 只对调用上下文的函数有效，并不向上传递


---
搬运地址：    
[Flutter和Dart系列四：Function](https://blog.csdn.net/xlh1191860939/article/details/87895616)      
[Flutter学习日记：Dart语言学习之typedef](https://blog.csdn.net/FreeAndWake/article/details/88979769)        
极客时间《Flutter核心技术与实战》      
