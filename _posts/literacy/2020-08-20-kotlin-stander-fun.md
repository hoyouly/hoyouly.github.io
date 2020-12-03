---
layout: post
title: 扫盲系列 - Kotlin 中的标准库函数 run、with、let、also、apply、use、takeif
category: 扫盲系列
tags:  Kotlin
---

<!-- * content -->
<!-- {:toc} -->

刚学习kotlin的时候，会发现一些奇怪的语法，例如
```
var dints2=ints.map{ it*2}

File("test.txt").let { }
```
等等，这些感觉好奇怪的，和java区别很大的，这些到底是啥呢，
在学习这些之前，我们先大概了解一下两个知识点，lambda表达式和 it


## lambda表达式

* 目的是为了更贴近函数式编程，把函数作为参数的思想
* 格式：`{(输入参数)-> (运算)输出}`

最外面使用`{}`,用`()`定义参数列表，箭头-> 以及一个表达式或者语句块

例如
```java
textView.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Log.d("hoyouly", " onClick "+v.getId());
    }
});
```
简化成为 lambda表达式后就成了下面这种形式
```kotlin
textView.setOnClickListener(view -> {Log.d("hoyouly", " onClick "+v.getId()});
//只有一行代码的，可以把{} 也省略掉
textView.setOnClickListener(view -> Log.d("hoyouly", " onClick "+v.getId());
```
再如

```java
executor.submit(
   new Runnable(){
     @Override
     public void run(){
         //todo
     }
  }
);
//简化为
executor.submit(() ->{/*todo*/})
```

## it
简单来说，it 就是为了简化代码而存在的。   
例如,对于这种单个参数的运算式。
```kotlin
val dints=ints.map{value->value*2}
```
可以进一步简化，把参数声明和->都简化掉，只保留运算输出,不过这要用it来统一代替参数

```kotlin
val dints2=ints.map{ it*2}
```

Kotlin里提供了一些有趣的函数，包括 let，apply，run，with，also,use、takeif等，他们都是建立在lamdba表达式的基础上的,并且用到了it.

## let
let能把更复杂的对象赋给it

```kotlin
File("test.txt").let {
       println(it::class.simpleName) //  File
       println(it.absolutePath) //  /Users/hoyouly/llll/HelloKotlin/test.txt
}
```
打印的it 类型就是 File,

这个特性可以稍微扩展一下，比如增加?检查

```kotlin
getVaraiable()?.let{
    it.length    // when not null
}
```
这样可以先检查返回值是否为空，不为空才继续进行

默认let{}返回中是带的it 类型，但是也可以进行修改成其他有含义的名称例如
```kotlin
File("test.txt").let {
     file-> //把默认的it 改成了有含义的file
     println(file::class.simpleName) //File
     println(file.absolutePath) //  /Users/hoyouly/llll/HelloKotlin/test.txt
 }
```
let 返回的是代码块中最后一行的对象

```kotlin
fun testlet() {
    val original = "abc"

    original.let {
        println("The original String is $it")  //adc
        it.reversed()
    }.let {//此时it 类型还是String
        println("The reverse String is $it") //cba
        it.length
    }.let {//此时it 类型就是Int
        println("The length of the String is $it") //3
    }
}
```

## also
kotlin 1.1 版本新加入的内容，像是let和apply的加强版。
有let的功能，可以直接操作对象本身，又与apply返回类似，是对象本身，并且不会发生变化
```kotlin
original.also {
    println("The original String is $it") //abc
    it.reversed()
}.also {//此时it 类型还是String
    println("The reverse String is $it") // abc
    it.length
}.let {//此时it 类型还是String
    println("The length of the String is $it") // abc
}
```
虽然 在 also 中执行了 it.reversed() ,可是再次输出的时候，还是abc ,

意义：
* 1、它可以对相同的对象提供非常清晰的分离过程，即创建更小的函数部分。
* 2、在使用之前，它可以非常强大的进行自我操作，从而实现整个链式代码的构建操作。

### let 和 also 组合
例如  创建一个文件夹，通用的写法
```kotlin
fun makeDir(path: String): File {
  val file=File(path)
  file.mkdirs()
  return  file
}
```
但是通过 also 和let 组合，就变得很简洁

```kotlin
fun makeDir(path: String): File {
  return path.let {
      File(it) //创建一个File对象，然后返回，
  }.also {//此时it 对象就是File，然后对it 进行 mkdirs() 的操作，
      it.mkdirs()
  } //最终return 的还是 also 的时候的it 对象，
}

```

## apply
apply可以操作一个对象的任意函数，可以结合let返回该对象，例如
```kotlin
var array: ArrayList<Int> = ArrayList()
println(array.size)//  0
array.apply {
    add(0, 30); //执行了ArrayList 的add方法，
}.let {
    println(it.size)//1
    println(it) // [30]
}
```
## run
是一个扩展函数，最终返回该扩展函数的结果。

在run函数中，我们拥有了一个单独的作用域，例如定义一个 nickName，并且他的作用域只存在于run函数中。

```kotlin
fun main() {
    val nickName = "Prefet"
    run {
        val nickName = "YarenTang"
        println(nickName) //YarenTang
    }
    println(nickName) //Prefet
  }
```

可以和let组合使用。

```kotlin
array.run {
    add(20)
    var text ="abd"
    text
}.let {
    println(it::class.simpleName)  //String
    println(array.size) // 2
}
```
当把 text 这一行注释掉之后，返回的结果就变了

```kotlin
array.run {
    add(20)
    var text ="abd"
//        text
}.let {
    println(it::class.simpleName)  //Unit
    println(array.size) // 2
}
```
这是因为 run()返回的是最后一行代码的对象，因为最后一行代码没有对象，所以就返回了Unit

run的一个应用可以把将show()方法应用到两个View中，而不需要去调用两次show()方法
```kotlin
run {
  if (firstTimeView) introView else normalView
}.show()
```
### run 和 apply 区别
* apply是操作一个对象，run则是操作一块代码
* apply返回操作的对象，run的返回则是最后一行代码的对象,这一点和let 很相似的


## with
with有点儿像apply，也是操作一个对象，不过它是用函数方式，把对象作为参数传入with函数，然后在代码块中操作，例如

```kotlin
with(array) {
    add(10)
    var text = "abd"
    text
}.let {
    println(it::class.simpleName)  //String
    println(array.size) // 3
}
```
同样，把text 注释掉，结果就变成了 Unit

```kotlin
with(array) {
    add(10)
    var text = "abd"
//        text
}.let {
    println(it::class.simpleName)  //Unit
    println(array.size) // 3
}
```

### with 和 run 区别
run函数可以在使用它之前对可空性进行检查
```kotlin
// Yack!(比较丑陋的实现方式)
with(webview.settings) {
      this?.javaScriptEnabled = true
      this?.databaseEnabled = true
   }
}
// Nice.(比较好的实现方式)
webview.settings?.run {
    javaScriptEnabled = true
    databaseEnabled = true
}
```

## takeif
如果不仅仅想判空，还想加入判断条件，可以使用takeif
```kotlin
val result=student.takeif{it.age>19}.let{...}
```
年龄大于19的才会执行let操作。

和filter异曲同工，不过takeif 只操作单条数据，与takeif相反的还有takeUnless,即接收器不满足特定条件才会执行

## use
* 实现了Closeable接口的对象可调用use函数
* use函数会自动关闭调用者（无论中间是否出现异常）
* Kotlin的File对象和IO流操作变得行云流水
```kotlin
File("/home/test.txt").readLines()
        .forEach { println(it) }

```
readLines()内部间接使用了use函数，这样就省去了捕获异常和关闭流的烦恼

## 使用标准库函数的补充
1. 建议尽量不要使用多个标准库函数进行嵌套，不要为了简化而去做简化，否则整个代码可读性会大大降低，一会是it指代，一会又是this指代，估计隔一段时间后连你自己都不知道指代什么了。
2. let函数和run函数之所以能够返回其他类型的值，其原理在于lambda表达式内部返回最后一行表达式的值，所以只要最后一行表达式返回不同的对象，那么它们就返回不同类型，表现上就是返回其他类型
3. T.also和T.apply函数之所以能能返回自己本身，是因为在各自Lambda表达式内部最后一行都调用return this,返回它们自己本身，这个this能被指代调用者，是因为它们都是扩展函数特性


- - - -
搬运地址：    

[Kotlin Jetpack 实战｜00. 写给 Java 开发者的 Kotlin 入坑指南](https://juejin.im/post/6844904191098355719)

[写给Android开发者的Kotlin入门](https://www.cnblogs.com/it-tsz/p/10751332.html)

[Kotlin 教程](https://www.runoob.com/kotlin/kotlin-tutorial.html)

[Kotlin use函数的魔法](https://www.jianshu.com/p/2328262cd49d)

[[译]掌握Kotlin中的标准库函数: run、with、let、also和apply](https://blog.csdn.net/u013064109/article/details/80387322)
