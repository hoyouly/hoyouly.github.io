---
layout: post
title: Kotlin核心编程 - lambda 和集合
category: 读书笔记
tags: Kotlin核心编程
---
<!-- * content -->
<!-- {:toc} -->

kotlin运行对java的类库做一些优化，任何函数接收了一个java的SAM（单一抽象方法）都可以用kotlin的函数进行替代。


# with 和 apply

最大的作用就是可以让我们在写lambda的时候，省略需要多次书写的对象。默认用this关键字来指向它

with 是一个 内联函数

```kotlin
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```

apply 是一个扩展函数
```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```

使用如下

```kotlin
fun main() {
    val withValue = with(2) {
        plus(3)
    }
    println(withValue) //5

    val applyValue = 2.apply {
        plus(3)
    }
    println(applyValue) //2

    val array: ArrayList<Int> = ArrayList()
    array.apply {
        add(4)
    }
    println(array.size)    //1
}
```
2.apply{}这个竟然输出结果还是2，这个倒是挺意外的。现在我还解释不清楚原因。

# 集合的高阶函数API

## 遍历：map
对一个集合遍历，生成一个新的集合，里面的元素是之前集合的2倍

```kotlin
val list = listOf(1, 2, 3, 4, 5, 6)
val newList = list.map{it * 2}
println(newList) //[2, 4, 6, 8, 10, 12]
```
map 实际上是一个高阶函数
```kotlin
public inline fun <T, R> Iterable<T>.map(transform: (T) -> R): List<R> {
    return mapTo(ArrayList<R>(collectionSizeOrDefault(10)), transform)
}

public inline fun <T, R, C : MutableCollection<in R>> Iterable<T>.mapTo(destination: C, transform: (T) -> R): C {
    for (item in this)
        destination.add(transform(item))
    return destination
}
```
通过源码可知:    
使用map方法只会，会产出一个新的集合，并且集合的大小与原来集合一样，
其实map方法就是接收一个函数，这个函数对集合中每个元素进行操作，然后将操作后的结果返回。最后产生一个由这些新结果组成的新的集合。

## 筛选：filter, count

### filter

```kotlin
data class Student(val name: String, val age: Int, val sex: String, val score: Int)

fun main() {
    val jilen = Student("jilen", 30, "m", 85)
    val shaw = Student("shaw", 18, "m", 90)
    val yison = Student("yison", 40, "f", 59)
    val jack = Student("jack", 30, "m", 70)
    val lisa = Student("lisa", 25, "f", 88)
    val pan = Student("pan", 36, "f", 55)

    val students = listOf(jilen, shaw, yison, jack, lisa, pan)

    val mStudents = students.filter { it.sex == "m" }
    println(mStudents) //[Student(name=jilen, age=30, sex=m, score=85), Student(name=shaw, age=18, sex=m, score=90), Student(name=jack, age=30, sex=m, score=70)]
}
```
过滤了所有sex 是 m 的学生。

接受

```kotlin
public inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {
    return filterTo(ArrayList<T>(), predicate)
}
public inline fun <T, C : MutableCollection<in T>> Iterable<T>.filterTo(destination: C, predicate: (T) -> Boolean): C {
    for (element in this) if (predicate(element)) destination.add(element)
    return destination
}
```

通过源码可知，filter与map类似，接收一个函数，只是改函数的返回值必须是Boolean，该函数的作用就是判断集合中的每一项是否满足某个条件，如果满足，filter方法就会将该项插入到新的列表中，最终得到的是一个满足给定条件的新列表。所以我们可得出结论：调用filter之后产生的新列表是原来列表的子集。

* filterNo: 用来过滤掉满足条件的元素，与filter方法的作用相反，当传入条件一样时候，会得到相反的结果。也就是说，filter+filterNo =原来列表
* filterNoNull: 过滤掉值为null的元素

###  count
统计满足条件的元素个数。
```kotlin
println(students.count{it.sex=="m"}) //3
```
结果就是3，
虽然可以通过students.filter { it.sex == "m" }.size 得到相同的结论，但是这个我们会通过filter得到一个满足条件的新列表，增加了额外的开销。
而count是直接变量原来的列表，满足函数后，count++得到的，通过源码我们也可以看出来。

```kotlin
public inline fun <T> Iterable<T>.count(predicate: (T) -> Boolean): Int {
    if (this is Collection && isEmpty()) return 0
    var count = 0
    for (element in this) if (predicate(element)) checkCountOverflow(++count)
    return count
}
```
## 求和： sumBy,sum,fold,reduce

### sumBy  和 sum

sum ：只能对数值类型的列表进行求和，而sumBy却可以对指定的类型进行求和

例如对学生的成绩求和
```kotlin
 println(students.sumBy { it.score }) //447

 //students.sum() //报错，
 val values= listOf(1,2,3,4,5)
 println(values.sum()) // 15
```

### fold

fold: 折叠；合拢；抱住

先看看怎么使用吧

```kotlin
val foldValue = students.fold(0) { accumilator, student -> accumilator + student.score }
println(foldValue) //447
```
也能达到求和的目的
这个就感觉好复杂啊，先看看源码吧

```kotlin
public inline fun <T, R> Iterable<T>.fold(initial: R, operation: (acc: R, T) -> R): R {
    var accumulator = initial
    for (element in this) accumulator = operation(accumulator, element)
    return accumulator
}
```
需要两个参数，
initial： 通常称为初始值。
operation： 是一个函数，

在实现的时候，通过for循环来遍历集合中的每个元素，每次都调用operation函数，而该函数有两个参数，一个是上一次调用该集合的结果，另一个则是单曲遍历用到的集合元素，
简单来说就是。每次调用operation函数，然后将产生的结果作为参数提供给下一次调用。
其实等价于
```kotlin
var accumilator = 0
for (student in students) {
    accumilator = accumilator + student.score
}
```
fold 很好的利用了递归的思想。

### reduce

reduce 方法和fold非常相似，唯一区别就是reduce 没有初始值。

```kotlin
public inline fun <S, T : S> Iterable<T>.reduce(operation: (acc: S, T) -> S): S {
    val iterator = this.iterator()
    if (!iterator.hasNext()) throw UnsupportedOperationException("Empty collection can't be reduced.")
    var accumulator: S = iterator.next()
    while (iterator.hasNext()) {
        accumulator = operation(accumulator, iterator.next())
    }
    return accumulator
}
```
reduce 只接收一个函数参数，具体实现方式也与fold类似，不同的是，当要遍历的集合为空的时候，会抛出一个异常。
因为没有初始值，所以默认的初始值就是集合中的第一个元素。

## 分组 groupBy

```kotlin
val groupByValue = students.groupBy { it.sex }
println(groupByValue)
```
根据学生性别进行分组，返回的是一个Map集合，类型是Map<String,List<Student>

## 嵌套： flatMap,flatten

```kotlin
val list = listOf(listOf(jilen, shaw, lisa), listOf(yison, pan), listOf(jack))
val newList = list.flatten()
println(newList.map { it.name }) //[jilen, shaw, lisa, yison, pan, jack]
```
这样就把一个集合嵌套集合扁平化成一个集合了，
其实flatten实现原理很简单的，源码看一下

```kotlin
public fun <T> Iterable<Iterable<T>>.flatten(): List<T> {
    val result = ArrayList<T>()
    for (element in this) {
        result.addAll(element)
    }
    return result
}
```
声明一个数组result,然后嵌套遍历这个嵌套集合，将每个子集中的元素通过addAll方法添加到result中。最终就是一个扁平化的集合。

### flatMap
其实上面的例子，也可以通过flatmap进行处理，

```kotlin
println(list.flatMap { it.map { it.name } }) //[jilen, shaw, lisa, yison, pan, jack]

```
结果是一样的。flatMap接收一个函数，该函数的返回值是一个列表，一个由学生姓名组成的列表。

```kotlin
public inline fun <T, R> Iterable<T>.flatMap(transform: (T) -> Iterable<R>): List<R> {
    return flatMapTo(ArrayList<R>(), transform)
}
public inline fun <T, R, C : MutableCollection<in R>> Iterable<T>.flatMapTo(destination: C, transform: (T) -> Iterable<R>): C {
    for (element in this) {
        val list = transform(element)
        destination.addAll(list)
    }
    return destination
}
```
看源码可知，flatMap 接收一个函数transform,
通过看这个函数的定义可知： transform: (T) -> Iterable<R>

 transform 函数接收一个参数，该参数一般为嵌套列表中某个子列表，返回值为一个列表。

 flatMap 中调用一个叫作flatMapTo的方法，这个方法接受两个参数，一个参数是一个列表，该列表是一个空列表
 另一个参数是一个函数，该函数返回为一个序列。

 实现很简单，先遍历集合中的元素，然后将每个元素传入函数 transform中得到一个列表，然后将这个列表中的所有元素添加到空列表 destination 中，最终就得到了这个经过 transform 函数处理过的扁平化列表

# 集合库的设计
Iterable 是集合库的顶层接口。

集合分两种
* 可变集合：带Mutable前缀的
* 只读集合：不带的Mutable的

例如 MutableList 可变的List，而List 则表示只读的List

可变集合与只读集合的区别就是，
可变集合- （修改，添加，删除等方法）=只读集合

只读集合只有一些可以用来读的方法。比如集合的大小，遍历集合等。这样做的好处就是可以让代码看起来更容易理解


# 序列 Sequence

通过asSequeue()方法将一个列表转换成序列
通过toList()方法将一个序列转换列表

将list转换成序列，很多大程度上就是提高了上面操作集合的效率。
这是因为在使用序列的时候，filter方法和map()方法的操作都没有创建额外的集合。这样当集合中的元素数量巨大的时候，就减少了大部分开销。


序列中的元素的求值是惰性的，

惰性求值： 一种在需要时候才进行求值的计算方式。感觉有点像是懒加载的意思

惰性求指的好处
* 一个是性能优化
* 能够构造出无限的数据类型

```kotlin
val values = listOf(1, 2, 3, 4, 5)

values.asSequence().filter { it > 2 }.map { it * 2 }.toList()
```
`filter { it > 2 }.map { it * 2 }`
filter和map返回的还是一个序列，我们称这部分操作是中间操作
toList()返回的是一个List,这类操作是末端操作，它的返回值不能是序列，必须是一个明确的结果。比如列表，数字，对象等表意明确的结果。

末端操作一般都放在链式操作的尾部。在执行末端操作的时候，才会去触发中间操作的延迟计算。

```kotlin
val values = listOf(1, 2, 3, 4, 5)

println("原始list使用 filter map")
val list = values.filter {
    println("fileter $it")
    it > 2
}.map {
    println("map $it")
    it * 2
}
println(list)

println("使用 Sequence 后 filter map")
val sequencelist = values.asSequence().filter {
    println("fileter $it")
    it > 2
}.map {
    println("map $it")
    it * 2
}.toList()
println(sequencelist)
```
输出的结果是

```kotlin
原始list使用 filter map
fileter 1
fileter 2
fileter 3
fileter 4
fileter 5
map 3
map 4
map 5
[6, 8, 10]
使用 Sequence 后 filter map
fileter 1
fileter 2
fileter 3
map 3
fileter 4
map 4
fileter 5
map 5
[6, 8, 10]
```

通过对比我们可以发现
普通的集合进行链式操作的时候，会先再list上调用filter，然后产生一个结果列表，接下来的map就是在这个结果列表上进行操作的，而序列则不一样，序列在执行链式操作的时候，会将所有的操作都应用在一个元素上，也就是说第一个元素指向玩filter,map之后，第二个元素才会执行。

当我们使用序列的时候，如果filter和map的位置可以互相调换，应该优先使用filter，这样会减少一部分的开销

# 内联函数

 使用inline关键字来修饰函数，这些函数就成为了内联函数，
 他们的函数体在编译器被嵌入每一个被调用的地方，以减少额外生成的匿名类类，以及函数执行的时间开销

 内联函数典型的一个应用场景就是kotlin的集合类，
 例如前面说的map,filter等，都被定义为内联函数。


内联函数不是万能的
以下情况避免使用内联函数
* 普通的函数，因为JVM对普通函数已经能够根据时间情况只能的判断是否进行内联优化了，所以不需要对其使用inline语法，那只会让字节码变得更加复杂
* 尽量避免对具有大量的函数体的函数进行内联，这样会冬至过多的字节码数量
* 一旦一个函数被定义为内联函数，边不能获取闭包类的私有成员，除非你把他们声明为internal

## noinline  :避免参数被内联

把oninline加在不想要内联的函数开头，改参数变不会具有内联的效果了

```kotlin
inline fun foo(block: () -> Unit, noinline block2: () -> Unit) {
    println("before block")
    block()
    block2()
    println("after block")
}

fun main() {
    foo(
        {
            println("dive into kotlin...")
        },
        {
            println("I am not inline")
        }
    )
}
```
block2参数带上了online之后，反编译后的java代码中并没有将其函数体代码在调用处进行替换

## 非局部返回

内联函数处理优化lambda开销之外，还带来了其他方面的特效。典型的就是非局部返回和具体化参数类型（reified）


正常情况下lambda 表达式是不允许存在return关键字的，

```kotlin
fun foo(block: () -> Unit) {
    println("before block")
    block()
    println("after block")
}

fun main() {
    foo {
        return@foo //直接使用return是不行的，但是可以使用非局部返回，即return@
    }
  }
```

输出结果
```kotlin
before block
after block
```
但是通过内联函数就可以了。

```kotlin
inline fun foo(block: () -> Unit) {
    println("before block")
    block()
    println("after block")
}

fun main() {
    foo {
        return
    }
}
```
输出的结果就是
```kotlin
before block
```
# crossinline

为了避免带有return的lambda参数产生破坏，可以使用crossinline关键字修饰该参数，从而杜绝此类问题的发生。




---
搬运地址：    

Kotlin 核心编程 - 水滴技术团队
