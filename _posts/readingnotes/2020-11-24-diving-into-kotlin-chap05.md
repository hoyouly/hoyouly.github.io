---
layout: post
title: Kotlin核心编程 - 类型系统
category: 读书笔记
tags: Kotlin核心编程
---
<!-- * content -->
<!-- {:toc} -->

# null 引用
null是一个不是值的值

null 存在歧义，因为可能包含多个含义
* 该值从未初始化过
* 该值不合法
* 该值不需要
* 该值不存在

## NullPointerException
对于java，编译时检查存在一个致命的缺陷-由于任何引用都可以为null，而调用一个为null的对象的方法，就会产生NPE（NullPointerException）

java 解决NPE问题的方案
* 函数内对于无效值，更倾向于抛出异常处理, 对于经常出现无效值的，有性能需求或者在代码中经常使用的函数并不适用。
* 采用 @NotNull/@Nullable标注。
* 使用专门的Optional对象对可能为null的对象进行装厢,这种方式耗时是普通盘控的十倍，

# 可空类型
kotlin 的思路，在类型层面提供一种可空类型，
kotlin 分为可空（nullable）和非空（not-null）类型


在kotlin中，我们可以在任何类型后面加上？,比如 Int？，等同于Int or null,通过合理的使用，不仅能够简化很多判空代码，还能够有效的避免NPE。

```kotlin
data class Seat(val student: Student?)
data class Student(val glasses: Glasses?)

data class Glasses(val degreeeOfMyopia: Double)


fun main() {
    val student = Student(Glasses(10.0))
    val seat = Seat(student)
    println(seat.student?.glasses?.degreeeOfMyopia)
}

```
例如这种，`?.`被称为安全调用，如果为空的话，就不会执行下一步。

## Elvis 操作符。
也称为合并运算符

例如 `val result = seat.student?.glasses?.degreeeOfMyopia ?: -1` 如果不戴眼镜，则返回-1，否则返回眼镜度数。

`!!. `  非空断言。。student!!.glasses  //当这个学生不带眼镜时候，程序就会抛出NPE。

通过IDEA反编译可以查看，kotlin的空类型实质上只是在java基础上进行了语法层的保证。方法上参数标注了@Nullable,依旧采用了if..else 来进行可空判断。



kotlin 可空类型优化与java Optional的地方体现在
* kotlin的可空类型兼容性好
* kotlin的可空类型性能更好，开销更低
* kotlin 可空类型语法简洁


## 类型检查 is

```kotlin
val obj: Any? = null
when (obj) {
    is String -> println(obj.length)
    !is String -> print(" Not a String")
}
```
不需要再强制转换了，is String 为true后，就可以直接把 原本是Any 类型的obj 当做String类型使用了。这就是智能转换（Smart Casts）帮我们省了的一些工作

智能转换（Smart Casts） 可以将一个变量的类型转换成另一种类型。

根据官方文档介绍，当且仅当kotlin的编译骑确定在类型检查后改变量不会再改变，才会产生Smart Casts,利用这一点，我们就能确保多线程的应用足够安全。

当需要强制类型转换时候，我们可以利用 as 操作符来实现

var stu: Student? = getStu() as? Student  //如果 stu 为空则不会抛出异常，而是返回转换结果为null

## reified 关键字：
利用它可以在方法内访问泛型指定的JVM类对象

注意： 这个关键字需要在内联函数 inline中使用。

配合泛型，封装一个更有效的类型转换方法
```kotlin
inline fun <reified T> cast(orignial: Any): T? = orignial as? T

```


java并不是一门纯面向对象的语言，因为他的原始类型 如int 的值与函数等并不能是作为对象

## Any : 非空类型的根类型
Object 是java类层级结构的顶层类似，Any 类型是kotlin中所有非空类型的超类


### 平台类型
平台类型本质上就是kotlin不知道可空性信息的类型，因为在kotlin中定义的类型，我们是能明确知道它到底是不是可空的，而如果kotlin中调用java的代码，那么对于kotlin来说，就不知道这个类型是否能为空了。

所有java引用类型在kotlin中都表现为平台类型，当kotlin中处理平台类型的值的时候，它即可以被当做可空类型来处理，也可以被当做非空类型来操作。

## Any? : 所有类型的根类型

* 继承 ： 强调的是实现上的复用。
* 子类型化 ： 核心是一种类型的替代关系， 是一种类型语义上的关系，与实现没关系

虽然Any 与Any? 看起来没有继承关系，然后在我们需要用到Any?类型的地方，先让可以传入一个类型为Any的值。这在编译上不会产生问题，反之却不然。

所以可以说Any? 是Any的父类型。是所有类型的根类型。

kotlin中的非空类型可以看做所谓的Union Type,近似于数学中的并集，Any? 可理解为 Any U Null

## Nothing 与 Nothing?

Nothing 是没有实例的类型，Nothing类型的表达式不会产生任何值

kotlin中的return，throw等（流程控制中与跳转相关的表达式）返回值都为Nothing

Nothing？是Nothing的父类型，所以Nothing处于kotlin类型的层级结构的最底层

Nothing 只能包含一个值，null，本质上与null没有区别，所以我们可以使用null作为任何可空类型的值

在kotlin中。Int 类型等同于int,Int?类型等同于Integer

# 数组
kotlin 中Array 并不是一种原生的数据结构，而是一种Array类，
IntArray ,CharArray等并不是Array子类，所以用两者创建的相同值的对象，并不是相同对象

# 泛型
利用泛型代码在编译期就能发现错误，防止在真正运行的时候就出现ClassCastException。
泛型除了帮助我们在编译时期进行类型检查外，还可以帮助我们自动类型转换

泛型的优势
1. 类型检查，能在编译时期就帮你检查出错误
2. 更加语义化，比如我声明一个List<String>,便可以知道里面存储的是String对象，而不是其他对象
3. 自动类型转换，获取数据时不需要进行类型强制转换
4. 能写出更加通用化的代码


kotlin中使用泛型和java一样，都是使用<>来表示泛型。比如<T>,<R>,<?>等

在kotlin中，必须要主动指明泛型类型的，
val smartList = ArrayList()  这种是不被允许的，



```kotlin
class FruitPlate<T : Fruit?>(val t: T)
// 泛型类型是 Fruit 以及子类还有null

val fruit =FruitPlate(Fruit(10.0))
val fruitPlate =FruitPlate(null)
```

<span style="border-bottom:1px solid red;"> 通过where关键字可以实现对泛型参数添加多个约束的条件。</span>比如下面这个，就必须是Fruit类型和 Groud类型的，才可以执行。
```kotlin
fun <T> cut(t:T) where T:Fruit,T:Ground{
    print("you can cut me")
}
```

# 泛型擦除
数组是协变的，而List是不变的，简单来说就是:   
`Object[]是所有对象数组的父类，而List<Object>却不是List<T>的父类`


数组可以在运行时候获取自身的类型，而List<Apple>在运行的时候只知道自己是一个List,而无法获取泛型参数的类型。

java 数组是协变的，也就是说任意类A和B，如果A是B的父类，则A[] 也是B[]的父类。

但是假如给数组加上泛型后，将无法满足协变的原则，因为运行是无法知道数组的类型。所以java中无法声明一个泛型数组。

Kotlin 中数组是支持泛型的，担任也就不再协变，也就是说不能将对象数组赋值给Array<Any>或者Array<Any?>


## 获得泛型参数的类型
泛型在运行的时候已经被擦除了，可是如果想在运行时候知道泛型参数的类型，比如序列化和反序列化的时候，该怎么办啊

### 主动指定
```kotlin
class Plate<T>(val t: T,val clazz: Class<T>){
    fun getType(){
        print(clazz)
    }
}

fun main() {
    val  plate=Plate(Apple(1.0),Apple::class.java)

    plate.getType() //class fun.hoyouly.kt.diving.Apple
  }
```
这种方式有一个限制，就是无法获取到一个泛型的类型。比如
```kotlin
val list = ArrayList<String>()
 val  plate=Plate(list,ArrayList::class.java)
 plate.getType() //class java.util.ArrayList
```
这种只能获得 ArrayList，可是我们更像知道的是 ArrayList<String>这种类型

### 通过匿名内部类

```kotlin
fun main() {
    val list = ArrayList<String>()
    println(list.javaClass.genericSuperclass)//java.util.AbstractList<E>

    val list1 = object : ArrayList<String>(){}
    println(list1.javaClass.genericSuperclass) //java.util.ArrayList<java.lang.String>
}
```
list2 声明的其实是一个匿名内部类，虽然它看起来和list一样，但是这种方式得到的泛型类型确实 java.util.ArrayList<java.lang.String>，
为什么会这样呢？
泛型类型擦除并不是真正的讲全部类型信息都擦除，还是会将类型信息放在对应classes的常量池中的。既然还存储这类型信息，那么想办法从这个常量池中获取到就行了，使用内部类就刚好可以实现

匿名内部类在初始化的时候就会绑定父类接口的相应信息，这样就能通过获取父类活父类接口的泛型类型信息来实现我们的需求。



获取所有类型信息的泛型类

```kotlin
open class GenericesToken<T> {
    var type: Type = Any::class.java

    init {
        val superClass = this.javaClass.genericSuperclass
        type = (superClass as ParameterizedType).actualTypeArguments[0]
    }
}
fun main() {
    val gt=object :GenericesToken<Map<String,Int>>(){}
    println(gt.type) //java.util.Map<java.lang.String, ? extends java.lang.Integer>
  }
```

Gson 就是使用相同的设计。

<!-- 3. 内联函数获取泛型

kotlin中内联函数在编译的时候编译器便会将相应的函数的字节码插入到调用的地方。也就是说参数类型也会被插入字节码中。
 -->

# 泛型 协变、逆变

##  out 关键字
如果在定义的泛型类和泛型方法的泛型参数前面加上out 关键字，就说明这个泛型类以及泛型方法是协变，

简单来说.`类型A是类型B的子类，那么Generic<A> 也是Generic<B>的子类`

支持协变的List只可以读取，不可以添加。
类似于java中的 <? extends Object>

通常情况下，一个泛型类Generic<out T> 支持协变，那么他里面的方法的参数类型不能使用T类型，
但可以添加 @UnsafeVariance 注解来解除这个限制。

例如List中的public fun indexOf(element: @UnsafeVariance E): Int

## in 关键字

逆变:`假如类型A是类型B的子类型，那么Generic<B> 反过来就是Generic<A>的子类型`

类似于java中的<? super T>


in和out是一个对立面，其中in代表泛型参数类型是逆变，out代表泛型参数协变


```kotlin
fun <T> copy(dest: Array<T>, src: Array<T>) {
    if (dest.size < src.size) {
        throw IndexOutOfBoundsException()
    } else {
        src.forEachIndexed { index: Int, value: T ->
            dest[index] = src[index]
        }
    }
}

fun <T> copyIn(dest: Array<in T>, src: Array<T>) {
    if (dest.size < src.size) {
        throw IndexOutOfBoundsException()
    } else {
        src.forEachIndexed { index: Int, value: T ->
            dest[index] = src[index]
        }
    }
}

fun <T> copyOut(dest: Array<T>, src: Array<out T>) {
    if (dest.size < src.size) {
        throw IndexOutOfBoundsException()
    } else {
        src.forEachIndexed { index: Int, value: T ->
            dest[index] = src[index]
        }
    }
}

fun main() {
    var dest = arrayOf<Number>(3)
    val src = arrayOf(1.0, 2.2, 3.2);
    //copy(dest,src) //不允许，

    copyIn(dest, src) //ok
    copyOut(dest, src)//ok
  }
```
copy()方法只能copy 类型相同的，但是copyIn和copyOut却可以copy类型不同的，例如Number和Double，这就是in 和out 的强大之处

* copyIn()中，T是Double类型，所以dest可以接受Double类型的父类型Array，例如Array<Number>
* copyOut()中，T是Number类型，所以src可以接收Number类型的子类型Array ,例如Array<Double>



## 通配符
如果对泛型参数类型不感兴趣，可以使用通配符代替泛型参数，java中的泛型通配符是？，kotlin中则是 `*`来表示类型通配符

```kotlin
val list3: MutableList<*> = mutableListOf(1, "kotlin", 2.0)

list3.add("hello")//出错


val list4: MutableList<Any?> = mutableListOf(1, "kotlin", 2.0)

list4.add("hello") //正常

```
`MutableList<Any?> 和 MutableList<*> ` 不是同一个类型的，虽然在初始化的时候，都可以放Int,String ,Double等任意类型，

前者可以添加任意元素，但是后者只是匹配某一种类型的，因为编译器不知道通配符是一种什么类型，所以就不允许这个列表add元素

其实通配符只是一种语法糖，背后也是用协变来实现的，所以 `MutableList<*>`本质上是 `MutableList< out Any?> `

---
搬运地址：    

Kotlin 核心编程 - 水滴技术团队
