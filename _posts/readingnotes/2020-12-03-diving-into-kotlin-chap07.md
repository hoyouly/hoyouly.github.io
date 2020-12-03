---
layout: post
title: Kotlin核心编程 - 多态和扩展
category: 读书笔记
tags: Kotlin核心编程
---
<!-- * content -->
<!-- {:toc} -->
## 多态类型

* 子类型多态： 一个子类继承一个父类情况
* 参数多态： 泛型就是最常见的形式
* 特设多态：可以理解为一个多态函数是有多个不同的实现，依赖于实参而调用相应版本的函数。是一种更加灵活的多态技术，在kotlin中，一些有趣的语言特性，如运算符重载，扩展都很好的支持这种多态。

## 运算符重载
operator关键字。将一个函数编辑为重载一个操作符或者实现一个约定

如下代码，Area 就具有了plus的能力

```Kotlin
data class Area(val value: Double)

operator fun Area.plus(that: Area): Area {
    return Area(this.value + that.value)
}

fun main() {
    println(Area(1.0) + Area(2.0)) //Area(value=3.0)
}
```
这里的plus(加法)是kotlin规定的函数名，还有minus（减法），times(乘法)，div(除法)，mod(取余)等函数可以实现运算符重载
kotlin严格区分了接受者是否可空，如果你的函数是可空的，你需要重写一个可空类型的扩展函数。

## 扩展函数
扩展函数被称为kotlin最有魅力的特性之一，扩展可以让代码更加通透。

扩展属性和函数的实现运行在类实例，他们的定义操作不会修改类本身。   
这样做就的好处就是被扩展的第三方类免于被污染，从而避免了一些因父类修改而可能导致子类出错的问题。

扩展函数的声明很简单:比普通的函数声明多了一个扩展类，格式如下
>fun 扩展类.函数名(){}


```kotlin
fun <T> MutableList<T>.exchange(fromIndex: Int, toIndex: Int) {
    val tmp = this[fromIndex]
    this[fromIndex] = this[toIndex]
    this[toIndex] = tmp
}
```
这就是一个扩展函数的例子。MutableList<Int>就是要扩展类，它是Kotlin标准库Collections中的一个List容器类，exchange是扩展函数名，其余的和普通函数并无区别

使用就如下
```kotlin
fun main() {
    val list = mutableListOf("a", "b", "c")
    list.exchange(1, 2)
    println(list)//[a, c, b]
}
```
看起来就像MutableList<Int> 自带的一个方法一样。

扩展函数可以理解为静态方法，它独立于任何对象，且不依赖类的特定实例，被该类的所有实例共享。

注意当扩展方法在一个类内部时，我们只能在该类和该类的子类中进行调用。

## 扩展属性

可以理解为扩展函数的一种特殊情况。

如下，给MutableList<Int>类型添加了一个和是否是偶数的属性。

```kotlin
//和是否是偶数的属性
val MutableList<Int>.sumIsEven: Boolean
    get() = this.sum() % 2 == 0

fun main() {
    val list1= mutableListOf(1,2,3,4)
    println(list1.sumIsEven)//true
}
```
注意：扩展属性不能有初始化值。因为扩展属性本质上也是对应的java中的静态方法。


## 扩展特殊情况

### 静态扩展函数

在kotlin中，如果你需要声明一个静态的扩展函数，必须将其定义在半生对象上，
```kotlin
cclass Son {
    companion object {
        val age = 19
    }
}
fun Son.Companion.foo1() {
    println("Companion age =$age")
}

fun Son.foo2() {
    println("normal age =${Son.age}")
}


fun main() {
    Son.foo1()//Companion age =19
    val son = Son()
    son.foo2() //normal age =19
  }
```

如上，在Son这个类上，声明了一个静态的扩展函数foo1()和一个普通的扩展函数foo2()。    
foo1()不需要Son的实例，就可以直接使用，但是foo2()则只能在Son的对象中才能使用。    
并且不能颠倒，即Son的对象 son 并不能直接调用 foo1(),Son 也不能直接调用foo2()。

注意： 并不是所有的类都有半生对象的，所以，也就意味着并不是所有的类都能进行静态的扩展函数的。


### 成员函数优先级高于扩展函数。
如果扩展一个已经存在的函数呢，例如
```kotlin
class Son {
    companion object {
        val age = 19
    }

    fun foo2() {
        println(" age =${age}")
    }
}

fun Son.foo2() {
    println("normal age =${Son.age}")
}

fun main() {
    val son = Son()
    son.foo2() // age =19
}
```
Son中自带了一个foo2()函数，又扩展了一个，那么执行
son.foo2() 的结果是 age =19，即成员函数执行了，扩展函数没执行

这个主要是为了避免造成混淆。试想一下如果是扩展函数高于成员函数，如果有人扩展了foo2()这个方法，其实在很大情况下，大家都是在用原生的方法，结果这一个扩展，把所有的都影响了，那不乱套了吗，还有一个，就是如果同时有几个同学都对扩展了foo2()这个方法，那么应该执行谁的呢。所以为了避免这些情况出现，同名的类成员函数优先级高于扩展函数。毕竟成员函数是亲生的，扩展函数只能是抱养过来的。



## 标注库中的扩展函数
详情查看 [扫盲系列 - Kotlin 中的标准库函数 run、with、let、also、apply、use、takeif](../../../../2020/08/20/kotlin-stander-fun/)   


## 扩展不是万能的

调度方式对扩展函数的影响

```kotlin
open class Base

class Extends : Base()

fun Base.foo() = "I am Base.foo !"

fun Extends.foo() = "I am Extengs foo !"

fun main() {

    val instance: Base = Extends()
    val instance2 = Extends()

    println(instance.foo()) //I am Base.foo !
    println(instance2.foo())//I am Extengs foo !
  }
```
虽然 instance 真正类型是Extends,可是输出的结果却还是 I am Base.foo !，这也就说明了扩展函数是一个静态调度，只考虑编译类型，instance的编译类型就是Base，运行时类型才是Extends

### 调度接收器和扩展接收器

* 扩展接收器： 与kotlin扩展密切相关的接收器，表示我们为其定义扩展的对象
* 调度接收器： 扩展被声明为成员是存在的一种特殊接收器，他表示声明扩展名的类的实例

```kotlin
class X {
    fun Y.foo()="I am Y.foo()"
  }
```
X 是调度接收器而Y则是扩展接收器。如果将函数声明为open，则它的调度接收器只能是动态的，而扩展接收器总是在编译时解析

```kotlin
open class Base
class Extends : Base()
open class X {
    open fun Base.foo() {
        println("I am Base.foo in X")
    }

    open fun Extends.foo() {
        println("I am Extengs foo in X")
    }

    fun deal(base: Base) {
        base.foo()
    }
}

class Y : X() {
    override fun Extends.foo() {
        println("I am Extengs foo in Y")
    }

    override fun Base.foo() {
        println("I am Base.foo in Y")
    }
}

fun main() {
    X().deal(Base()) //I am Base.foo in X
    Y().deal(Base()) //I am Base.foo in Y
    X().deal(Extends()) //I am Base.foo in X
    Y().deal(Extends()) //I am Base.foo in Y
  }
```
Y().deal(Base())  输出的结果，即调度接收器被动态处理

X().deal(Extends()) 输出的结果表明 扩展接收器被静态处理

因为deal()接受的参数是Base,所以，不管传递的是Base对象，还是Extends对象，最终都会执行到Base的扩展函数foo中。

## 扩展需要注意的地方
* 如果该扩展函数是顶级函数或者成员函数，则不能被覆盖
* 我们无法访问其接收器的非公共属性
* 扩展接收器总是被静态调度的

扩展函数固然方便，但是方便执行带来了滥用的情况，在新特性面前，我们不能过于喜新厌旧，应结合面向对象思想和设计模式来进行规范。

---
搬运地址：    

Kotlin 核心编程 - 水滴技术团队
