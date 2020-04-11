---
layout: post
title: Flutter 学习 - Dart 中的 Future
category: Flutter 学习
tags: Flutter  Dart  Future
---
* content
{:toc}

Dart 是单线程的。

单线程和异步并不冲突，因为 APP 在绝大多数情况下都是等待，比如等用户点击，等网络请求返回，等文件 IO 结果，而这些行为并不是阻塞的，比如说，网络请求， Socket 本身提供了 select 模型可以异步查询；而文件 IO ，操作系统也提供了基于事件的回调机制。

所以可以在等待过程中做别的事情，而真正需要相应结果的时候，再去做对应的处理。

因为等待过程并不是阻塞，所以给我的感觉是同时做了很多事情一样。但其实是一个线程在处理你的事情。

等待这个行为是通过 Event Loop 驱动的，一共有两个事件队列
* Event Queue  事件队列，
* Microtask Queue 微任务队列 ，优先级很高，短时间就完成的异步任务。由 scheduleMicroTask 建立，

![](../../../../images/event_loop.png)

循环的过程：每一次事件循环，会先去微任务队列中查询，如果队列不为 null ，则执行微任务队列中的事件，直到为 null ，才会执行事件队列里面的任务。  

不过很少有事件用在这个队列中。就连 Flutter 内部，也只有 7 处用到了而已（比如，手势识别、文本输入、滚动视图、保存页面效果等需要高优执行任务的场景）

大多数情况下，我们使用的是 Event Queue ，比如，I/O、绘制、定时器这些异步事件，都是通过事件队列驱动主线程执行的。

Flutter 为 Event Queue 的任务建立提供了一个封装，就是 Future

# Future
单线程，主线程由一个时间来执行，类似 Android 的主线程，对于异步代码，通过 Future 来获取结果

Future 对象表示异步操作结果，通常通过 then() 来处理返回的结果

把一个函数放到 Future 中，就完成了从同步任务到异步任务的封装

声明一个 Future 时， Dart 会把异步任务的函数体放入到事件队列中，然后立即返回。后续的代码继续执行，而当同步执行的代码执行完毕后，事件队列会按照（FIFO），依次取出事件进行执行，最后同步执行 Future 的函数体以及后续的 then()

## 结论
1. then 与 Future 函数体共用一个事件循环。如果 Future 有多个 then ，它们也会按照链式调用的先后顺序同步执行，同样也会共用一个事件循环。
2. 如果 Future 执行体已经执行完毕了，但你又拿着这个 Future 的引用，往里面加了一个 then ， Dart 会将后续加入的 then 方法体放入微任务队列，尽快执行。
3. 如果 then 中也是一个 Future ，那么这个 then ，以及后续的 then 都被放入到事件队列中了，
4. then 会在 Future 函数体执行完毕后立刻执行，无论是共用同一个事件循环还是进入下一个微任务。

看几个例子
## 例一
```java
//声明了一个匿名 Future ，并注册了两个 then 。第一个 then 是一个Future
Future(() => print('f6'))
  .then((_) => Future(() => print('f7')))
  .then((_) => print('f8'));

//声明了一个匿名Future
Future(() => print('f9'));
```
这段代码的结果就是
```
f6   f9   f7   f8
```
因为 f6 执行完后， f7 会添加到 事件队列中，队列中还有 f9 可以执行， f8 会再 f7 执行后再执行。

## 例二
```java
//f1比 f2 先执行
Future(() => print('f1'));
Future(() => print('f2'));

//f3执行后会立刻同步执行then 3
Future(() => print('f3')).then((_) => print('then 3'));

//then 4会加入微任务队列，尽快执行
Future(() => null).then((_) => print('then 4'));
print("main thread");
```
结构就是
```
main thread
f1
f2
f3
then 3
then 4

```

下面是一个综合案例。

## 案例
```java
Future(() => print('f1'));//声明一个匿名Future
Future fx = Future(() =>  null);//声明 Future fx ，其执行体为null

//声明一个匿名 Future ，并注册了两个 then 。在第一个 then 回调里启动了一个微任务
Future(() => print('f2')).then((_) {
  print('f3');
  scheduleMicrotask(() => print('f4'));
}).then((_) => print('f5'));

//声明了一个匿名 Future ，并注册了两个 then 。第一个 then 是一个Future
Future(() => print('f6'))
  .then((_) => Future(() => print('f7')))
  .then((_) => print('f8'));

//声明了一个匿名Future
Future(() => print('f9'));

//往执行体为 null 的 fx 注册了了一个then
fx.then((_) => print('f10'));

//启动一个微任务
scheduleMicrotask(() => print('f11'));
print('f12');
```
 输出的结果是

```java
f12 f11 f1 f10   f2 f3 f5  f4  f6 f9  f7  f8
```
* 第一轮循环，只输出了 f12 ，其他的要么添加到事件队列中，要么添加到微任务队列中
* 第二轮循环，输出 f11 ，因为他在微任务队列中
* 第三轮循环，输出 f1 ， fx 执行，然后
* 第四次循环，输出 f10 ，因为 f10 的引用 fx 在上一轮执行了，那么再用这个 Future 引用，会把他们添加到微任务队列中，首先执行，所以会 输出 f10
* 第五轮循环， 输出 f2 f3 f5 ， f4 会添加到微任务队列中，
* 第六轮循环，输出 f4
* 第七轮循环，输出 f6 ， f9 ，同时 f7 添加到事件队列中
* 第八轮循环，输出 f7  f8

# 异步函数

对于异步函数返回的 Future 对象，在 Future 上注册一个 then ,进行异步处理。

如果调用者决定同步等待，则需要在调用处使用 await 关键字，并且在调用处的函数体使用 async 关键字。

Dart 中的 await 并不是阻塞等待，而是异步等待 。 Dart 会把调用体的函数也当做异步函数，将等待异步任务的上下文也放入到 Event Queue 中，一旦有了结果， Event Loop 就会把它从 Event Queue 中取出，等待代码继续执行。

在使用 await 进行等待的时候，调用上下文函数需要加上 async 关键字
## 例一
```java
//声明了一个延迟 2 秒返回 Hello 的 Future ，并注册了一个 then 返回拼接后的Hello 2019
Future<String> fetchContent() =>
  Future<String>.delayed(Duration(seconds:2), () => "Hello")
    .then((x) => "$x 2019");
//异步函数会同步等待 Hello 2019 的返回，并打印
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
调用 func() 函数，里面又调用了异步函数 fetchContent() ,同时使用了 await 进行等待，因此把 fetchContent() 和 await 语句的上下文函数 func() 放入事件队列中。
这只把 func() 函数放入事件队列，可是后面的 print() 并没有放入，所以会执行.
2 秒后， fetchContent 异步任务返回“Hello 2019”，于是 func 的 await 也被取出，打印“Hello 2019”

如果想要输出
```java
func before
Hello 2019
func after
```
那么就需要在 main() 上添加 async 关键字，即
```java
main() async{
  print("func before");
  print(await fetchContent());//等待 Hello 2019 的返回
  print("func after");
}
```
## 例二
```java
Future(() => print('f1'))
  .then((_) async => await Future(() => print('f2')))
  .then((_) => print('f3'));
Future(() => print('f4'));
```
结果是
```
f1  f4   f2   f3
```
并没有如我们想的那样，输出 f1  f2   f3  f4 ，因为 await 不会阻塞 f4 的，
如果要得到我们想要的结构，就需要这样处理

```java
main() async{
 await Future(() => print('f1'))
  .then((_)  =>  Future(() => print('f2')))
  .then((_) => print('f3'));
Future(() => print('f4'));
}
```

---
搬运地址：    

极客时间《Flutter核心技术与实战》      
