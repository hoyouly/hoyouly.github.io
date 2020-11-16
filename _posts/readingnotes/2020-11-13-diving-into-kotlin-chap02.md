---
layout: post
title: Kotlin核心编程 - 基础语法
category: 读书笔记
tags: Kotlin核心编程
---
<!-- * content -->
<!-- {:toc} -->

## 类型声明


* 如果一个函数没有声明返回值的类型，函数会默认被当成返回Unit类型
`fun sum(x: Int, y: Int, z: Int) { return x + y + z} `
报错，因为默认是Unit ,却return Int类型

* Kotlin支持这种单行表达式与等号的语法来定义函数，叫做表达式函数体 。普通的函数声明则可叫做代码块函数体  

### 判断是否需要显示声明类型
* 如果它是一个函数的参数。必须使用
* 如果是一个非表达式定义的函数。除了返回Unit，其他情况必须使用
* 如果是一个递归函数。必须使用
* 如果它是一个共有方法的返回值。为了更好的代码可读性及输出类型的可控性，建议使用


### val 和 var
* var  普通变量

* val  只读变量。相当于java中给变量添加了final 关键字

val虽然声明是只读变量，引用不可更改，但是并不代表其引用对象的可变成员不能修改
```kotlin
class Book(var name: String) {
    fun printName() {
        println(this.name)
    }
}
fun main() {
    val book =Book("think in java")
    book.name="Diving into kotlin"
    book.printName() //  Diving into kotlin
```
虽然声明了一个val 类型的变量 book ,book的引用没变，但是里面的属性name的值可以改变的。

<span style="border-bottom:1px solid red;"> 尽可能的采用val,不可变对象及纯函数来设计程序。</span>

纯函数：没有副作用的函数，具备引用透明性。

#### 函数的副作用
副作用：就是修改了某处的某些东西，例如
* 修改的外部变量的值
* IO操作，如数据写入磁盘
* UI操作，如修改了一个按钮的可操作状态

副作用的产生往往和可变数据和共享状态有关。有时候它会使得结果变得难以预测。

在kotlin中，推荐优先使用val 来声明一个本身不可变的变量。可以避免避免副作用。
* 这是一种防御行的编码思维模式，更加安全可靠，因为变量的值永远不会在其他地方被修改（一些框架采用反射技术的情况除外）
* 不可变的变量意味这更加容易推理，越是复杂的业务逻辑，它的优势越大。


## 高阶函数

函数式语言一个典型的特性： 函数是头等公民。   
我们不仅可以像类一样在顶层定义一个函数，也可以在一个函数内部定义一个函数   
还可以将一个函数向普通变量一样传递给另外一个函数，或者在其他函数内被返回。

```kotlin
(Int)-> ((Int)-> Unit) //表示传入一个类型为Int的参数，然后返回另一个类型为(Int）-> Unit的函数。
//或者这种
(Int)-> (Int)-> Unit  //传入一个(Int)-> Int的函数类型的参数，返回一个Unit
```
函数： 就是一种定义过程的能力。   
过程：总结一些公关的行为，如对数字的加减法，求立方等，被称为过程。 它接收的数字是一种数据，然后也能产生另外一种数据。

高阶函数：以其他函数作为参数或返回值的函数


### 函数的类型
遵循以下几点
* 通过-> 符号来组织参数类型和返回值类型，左边是参数类型，右边是返回值类型
* 必现通过一个括号包裹参数类型   
如果是无参函数，括号内容为空。`()->Unit`
如果是多个参数，使用逗号隔开。`(Int,String)->Unit`
* 返回值即时是Unit,也必须显示声明

### 方法和成员引用
使用 `::` 对某个类的方法进行引用，


### 匿名函数
Kotlin支持在缺省函数名的情况下，直接定义一个函数，

只能存在某个函数体内，或者赋值给某个变量中。不能单独存在。

```kotlin
//赋值给变量
val foo=fun(content: String): Boolean {
    return content.equals("en")
}

//存在main()函数内部
fun main() {
    fun(content: String): Boolean {
        return content.equals("en")
    }
}
```

## lambda
和匿名函数一样，是一种函数字面量。具体语法如下：
* 一个lambda表达式必须通过{}来包裹   
val sum: (Int, Int) -> Int = { x: Int, y: Int -> x + y }
* 如果lambda声明了参数部分的类型，且返回类型支持类型推导，那么lambda变量就可以省略函数类型声明。   
val sum = { x: Int, y: Int -> x + y } ，x 和y 声明了Int类型，那么返回值 x+y就可以省略Int类型声明了
* 如果lambda变量声明的函数类型，那么lambda的参数部分的类型就可以省略     
val sum: (Int, Int) -> Int = { x, y -> x + y } ，声明了函数类型是 (Int, Int) -> Int，那么参数部分就可以省略Int

<font color="#ff000" >如果lambda表达式返回值不是Unit，那么最后一行表达式的值类型就是返回值类型。</font>



* fun 在没有等号，只有{}的情况下，是我们最常见的代码块函数体，如果返回非Unit,必须带有return   

```kotlin
fun foo(x:Int){print(x)}
fun foo(x:Int){ return x*x}
```
* fun 带有等号，是单表达式函数体，该情况下可以省略return

```kotlin
fun foo(x:Int)= x*x
```
<font color="#ff000" >不管是val 还是fun,如果是等号+{}的语法，那么构建的就是一个lambda表达式。lambda的参数在{}内部声明</font>

如果左侧是fun,那么lambda表达式函数体，也必须通过()或者invoke来调用lambda

```kotlin
val foo = { x: Int -> print(x) } //foo.invoke(1)或者foo(1)
fun foo(x: Int) = { y: Int -> x * y }//foo(1).invoke(2)或者foo(1,2)
```
不带参数的情况

```kotlin
val foo = { print("无参 val ") }  //foo() 或者foo.invoke()
fun foo1() = { println("无参 fun") } // foo1().invoke() 或者foo1()()
```
注意，如果带有参数，{}中必须使用-> 把参数和结果隔开。

```kotlin
fun foo(x: Int) = { y: Int -> x * y }//foo(1).invoke(2)或者foo(1,2)
//等价于
fun foo(x: Int): (Int)->Int{
    return { y: Int -> x * y }
} //foo(1)(2)
```

```kotlin
fun sum(x: Int, y: Int, z: Int) = x + y + z //sum(1, 3, 4)
//等价于
fun sum(x: Int) = { y: Int ->
    { z: Int -> x + y + z }
} //sum(1)(2)(3)
```
可以看到，第二种方式有点像是击鼓传花，整个过程如下

* 开始的暗号就是第一个参数
* 下个环节的演绎就是返回的函数
* 谜底就是科里化最终执行获得的结果


如果一个函数最后一个参数是函数类型，那么在调用该函数的时候，最后的函数类型可以写道括号外面，
如下所示。
```kotlin
fun curryingLike(content: String, block: (String) -> Unit) {
    block(content)
}

//下面四种调用方式是等价的。
curryingLike("look like curring style", { content ->
    println(content)
})

curryingLike("look like curring style", {
    println(it)
})

curryingLike("look like curring style") { content ->
    println(content)
}

curryingLike("look like curring style") {
    println(it)
}
```

如果只有一个参数并且是函数类型，那么括号也可以省略

```kotlin
fun omitParaentheses(block: () -> Unit) {
    block()
}

//下面两种方式等价的
omitParaentheses(){
    println("omitParaentheses")
}

omitParaentheses {
    println("omitParaentheses")
}
```

### 闭包
在Kotlin中，匿名函数体，lambda表达式，局部函数，object表达式，在语法上都存在{},由{}包裹的代码块如果访问了外部变量则成为一个闭包。

一个闭包可以被当做参数传递或者直接使用。简单的看成 访问外部环境变量的函数


## 面向表达式编程

kotlin 的流程控制不再是清一色的普通语句，而是可以返回值的。是一些崭新的表达式语句。例如if-else 表达式，when 表达式，函数体表达式，lambda 表达式等

* 程序的赋值，循环，打印等操作，都可以被称为语句
* 表达式可以是一个值，常量，变量，操作符，函数，或者他们之间的组合，编程语言对其进行解释和计算，以求产生另一个值。简单来说：<font color="#ff000" > 表达式就是可以返回值的语句</font>

表达式更倾向于自成一块，避免与上下文共享状态，相互依赖，因此具有更好的 隔离性。


从设计角度而言：
* 语句的作用就是服务于创建副作用的。
* 表达式的目的则是为了创造新值。

<span style="border-bottom:1px solid red;">
在函数式编程中。原则上表达式是不允许包含副作用的。   
一切皆表达式。让开发者在设计业务时候，促进了避免创造副作用的逻辑设计。从而让程序变得更加安全。
</span>




都有值。并且可以将一个表达式作为组成其自身的一部分。


### if-else 表达式
* 在if 作为表达式的时候，else 分支必须被考虑，
* 返回值类型是各个逻辑分支的相同类型或者公共父类型。


### void & Void & Unit

* void : java中一个关键字。如果一个函数没有返回值，则需要用void 修饰
* Void : java的一个类，为了对应void,它继承Object,类型是final,构造方法是私有的，所以不能具有实例
* Unit : kotlin 中的一种类型。一个单例，可写为()，除了不代表任何意义之外，与其他常规类型并没有什么差别


Kotlin 中的`?:`：，属于一种 elvis 运算符。类似三元运算符,`maybeInt ?: 1` 表示，如果变量maybeInt 不为空，返回maybeInt，否则返回1



### 枚举类

在枚举类中存在额外的方法或属性定义，必须强制加上分号。

```kotlin
enum class Day(val day: Int) {
    MON(1),
    TUR(2),
    WEN(3),
    THU(4),
    FRI(5),
    SAT(6),
    SUN(7)
    ;//下面有额外的方法或者属性定义，必须加上分号

    fun getDaNubmer(): Int {
        return day
    }
}
```

### when 表达式
具体语法
* 类似switch，由when 关键字开始，用{}包含多个逻辑分支，每个分支由->连接，不再需要break,由上到下匹配，一直匹配完为止。否则执行else 分支，类似switch的default
* 每个逻辑分支具有返回值。最终整个when的表达式返回类型就是所有分支相同的返回类型，或公共父类型。

```kotlin
fun testWhen(a: Int) = when (a) {
    1 -> println(2)
    9 -> println(3)
    2 -> 5
    else -> when {
        a > 7 -> 8
        else -> 10
    }
}

fun main() {
    val fo = testWhen(9); //3
    println(fo) //kotlin.Unit

    val fo2 = testWhen(8);
    println(fo2) //8
}
```

### for 循环和范围表达式
for 循环遍历如下。

```kotlin
for (i: Int in 1..10) {
    print(i) //12345678910
}
for (i in 1..10) {
    print(i) //12345678910
}

for (i: Int in 1..10) print(i) //12345678910

for (i in 1 until 10) {
    print(i) //123456789
}

for (i: Int in 1..10 step 2) {
    print(i)//13579
}

for (i in 10 downTo 1 step 2) {
    print(i) //108642
}
```

* `..` 范围表达式， 1..10 表示从1 到10，除了整型的基本数据类型，实现了的Comparable接口的都可以。例如 "adc".."xyz"
* step  步长，step 2 表示隔一个取一个
* downTo  倒序
* until ,实现半开区间。
* in ,检查一个元素是否在另一个区间或者集合中
```kotlin
"kot" in "abd".."xyz"  //true
//等价于
"kot">="adb"&&"kot"<="xyz"
```


### 可变参数

使用 varags  关键字表示

```kotlin
fun printLetters(vararg letters: String, count: Int): Unit {
    println("$count letter are ")
    for (letter in letters) print(letter)
}

printLetters("1","3","t","e","d",count = 8)
```

不需要放到最后一个，参数的任意位置都行，如果不是最后一个，则后面的参数需要显示指定，例如 count=8,这种形式

可以使用`*` 来传入外部的变量作为可变参数的变量，例如


```kotlin
val letter= arrayOf("1","3","t","e","d")
printLetters(*letter,count = 8)
```

## 字符串的定义和操作

* 原生字符串，使用 `"""    """`
包裹起来即可

```kotlin
val rawString="""
    \n kotlin is awesome.
    \n kotlin is a better java .
"""
print(rawString)

```
返回的结果如下
```kotlin
  \n kotlin is awesome.
  \n kotlin is a better java .
```
最终打印格式与在代码中所呈现的格式一致，并不会转义\n ,以及Unicode的转义字符

* 字符串模板

使用$ 直接进行拼接。例如` println("$count letter are ")`,count 是一个变量

提升了代码的紧凑性和可读性

* 字符串判等

* 结构相等 `==`，判断对象内容是否相等  
* 引用相等 `===` ，判断对象的引用是否相等。



---
搬运地址：    

Kotlin 核心编程 - 水滴技术团队
