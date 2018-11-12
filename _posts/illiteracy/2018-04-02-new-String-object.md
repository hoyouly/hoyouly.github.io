---
layout: post
title: 街题系列---String s=new String("abc")创建了几个对象?
category: 街题系列
tags: jvm
---
* content
{:toc}

街题是我自己新创建的一个词，我相信只要说街机，大家都知道啥意思，就是使用人数较多的某种型号的手机，那么街题我给的定义就是面试（主要指Android）中经常被问到的问题。。

解释完这个街题，那么就开始第一个常见的街题，｀String s=new String("abc")创建了几个对象｀，我相信有很多人面试的时候被问道过，我也很没能幸免。在小米的时候被问道了，原话就是　String s=new String("abc")创建了几个对象，分别是啥，放在哪里？
这里面包含两个问题：
1. 创建几个对象 分别是啥
2. 放在哪里

# 1. 创建几个对象 分别是啥

首先我们得知道，java 创建对象的方式：
* new
* clone
* 反射
* 反序列化。

那么接下来看`String s=new String("abc")`　这行代码。先简单分解一下：

1. String s　：　只是定义了一个名为ｓ的字符串变量，并没有创建对象
2. =　：　　只是对变量ｓ的初始化，将某个丢下的引用赋值给它，显然它没有创建对象
3. new String("adc"　)  ： 肯定是创建 String 对象，因为new 在这里嘛，可是真的是一个吗？
上网经常听到的一句话：有问题，上百度，但是我要是： 有问题，看源码。看看 new String("adc"　)的源码

```java
public String(String toCopy) {
      value = (toCopy.value.length == toCopy.count)
              ? toCopy.value
              : Arrays.copyOfRange(toCopy.value, toCopy.offset, toCopy.offset + toCopy.length());
      offset = 0;
      count = value.length;
  }
```

查看源码我们发现，new String("adc"),接受的对象竟然还是String对象，那么就说明“abc”也是一个String对象，这个"adc"对象怎么来的呢，答案就是在JVM常量池，那么JVM字符串常量池是啥玩意呢

`JVM 字符串常量池`： 在Java虚拟机（JVM）中存在一个字符串池，其中保存了很多String对象，并且可以被共享使用，因此提高了效率，由于String是final，它的值一经创建就不可改变，因此不比担心String对象共享而带来的程序混乱。字符串池由String类维护，可以调用intern()方法来访问字符串池。

String 有两种赋值方式，
1. 通过字面量赋值，例如 `String str = "Hello";`
2. 通过关键字new 创建新对象 。例如 `String str = new String("Hello");`

接着回到`new String("adc"　)`上面， 所以可将： new String("adc") 再次分解
1. String str="abc"  通过字面量赋值
2. new String(str)   通过new 关键字赋值。

解释：
1. JVM 首先在字符串常量池中查找是否已经存在一个值为“adc”的对象，它的判断依据就是String类中的equals()方法的返回值，如果有，则不在创建新的对象，直接把该对象的引用返回，如果没有，则创建一个这个对象，然后把他添加到常量池里面。再返回这个对象的引用并赋值给str
2. str 传递给 new String(str)中。创建一个新的对象，并把引用赋值给s 变量。

这里解释一下new 对象的过程
## 关于 new
在java 语言里，<span style="border-bottom:1px solid red;">new 表达式是负责创建实例的，其中会调用构造函数去对实例做初始化，构造函数的返回值类型是void，并不是构造器返回了新创建的对象引用。而是new 表达式的值是新创建的对象的引用 </span>
对应的，在JVM里面，new 字节码指令只负责把实例创建出来（包括分配空间，设定类型，所有字段设置默认值等工作），并且把创建新对象的引用压入操作数栈顶。此时该引用还不能直接引用。处于未初始化状态（uninitialzed），如果某方法a视图通过未初始化状态的实例调用任何实例方法，那么方法a就通不过JVM的字节码校验，从而被JVM拒绝执行。
能对未初始化状态的引用做的唯一一件事就是通过调用它的构造函数，在class文件层面上表现为特殊的初始化方法"<init>" ,实际调用的是invokespecial，而实际调用前要把需要的参数按顺序压入到操作栈里，在构造函数返回之后，新创建的实例应用就可以正常使用 。

## 答案
所以最后的结论是： 如果JVM中字符串常量池中没“abc”,那么一共创建了两个对象，一个是字符串字面量 "abc" 所对应的，驻留（intern）在一个全局共享的字符串常量池中的实例，另一个就是通过new String(String)创建并初始化的、内容与"abc"相同的对象
如果JVM字符串常量池中有“abc”，那么就创建一个对象，即通过new String(String)创建并初始化的、内容与"abc"相同的对象

## 相似的问题
然后引发了一系列相同的问题：

1. String str="abc";   创建了一个对象，
2. String a="abc";  String b="abc";   那这里呢？ 还是一个对象，那就是"adc"，后面会解释。
3. String a="ab"+"cd"; 这样呢，答案是三个  
只有使用引号包含文本的方式创建String对象之间使用“+”链接的产生的新对象才会被加入到字符串池中，对于所有包含new方式新建对象（包括null）的“+”方式表达式，它所产生的新对象是不会添加到字符串池中的。因此提倡用引号包含文本的方式创建String对象以提高效率，实际上这也是我们在编程中常用的方式，
接下来回答第二个问题。放在哪里？

# 2. 放在哪里
下面是 [java用这样的方式生成字符串：String str = "Hello"，到底有没有在堆中创建对象？ - 胖胖的回答](https://www.zhihu.com/question/29884421/answer/113785601) 原话，之所以再写下来，一是从这里面我们能找到一些答案。二是为了自己以后看起来方便。

## JVM 结构图
先看看下面这张虚拟机的结构图  
![虚拟机的结构图](https://github.com/hoyouly/BlogResource/raw/master/imgesjvm.jpg)
其他先不管，主要看中间五彩叫运行时数据区（runtime data area） ,就是虚拟内存管理，也就是所谓的内存。一般讲虚拟机内存的主要是三块。
* Heap Mermory: 堆  最大一块空间，存放对象的实例和数组。全局共享  "abc"在这个里面  new String()也在这里面，
* Java Stacks : 栈  全称 JVM Stacks 虚拟机栈。存放基本数据类型，以及对象引用，线程私有  
* Method Area： 方法区  类被加载后的信息，常亮，静态变量存放在这，全局共享。在hotSpot里面也叫“永生代”，
## Stacks 区
![](https://github.com/hoyouly/BlogResource/raw/master/imges51f4a70e8f951c8c3d9c0eee21e2157c_r.jpg)
Stacks 区中的局部变量表（Local Variables）和操作数栈(Operand Stack),因为栈是私有的，每个方法被执行的时候都会创建栈帧（Stack Frame）,而每一个栈帧都维护着一个局部变量表和操作数栈，<span style="border-bottom:1px solid red;">我们经常说的基本数据类型和对象引用存在栈里，其实就是指的存在局部变量表中，而操作数栈是线程实际的操作台 </span>。看下图，做一个100+98的加法，局部变量表就是存数据的，一直不变，直到结果出来把和添加进去，而操作数栈就很忙了，先把两个数压进去，然后再求和，最后再弹出结果。
![](https://github.com/hoyouly/BlogResource/raw/master/imges100add80.jpg)

## 非堆区  
中间这个非堆（Non-heap） 可以粗略的理解非堆里面的包含了永生代，而永生代又包含了方法区。每个类加载完后，类的信息就存到了方法区，和String最相关的是 运行时常量池(Run-Time Constant Pool)，他是每个类私有的，每个class文件的常量池被类加载器加载进来后，就会映射在这个区域，另一个就是字符串常量池（String Pool）,和运行时常量池不是一个概念，字符串常量池是一个全局共享的，位置就在Non-heap 的Interned String 位置。可以理解在永生代上，方法区外，String.intern（）方法字符串驻留之后，引用就放到这里面。

比如下面这个Test.java ，在主线程方法main中，声明了一个字面量hello的字符串str
```java
Class Test{
  public void f(String s){...}

  public static void main(String[] args){
    String str="hello";
    ...
  }
}

```

编译成Test.class 文件之后，如下图，除了版本，字段，方法，接口等描述信息之外，还有一个也叫常量池（Constant Pool Table）的东西，但是这个常量池和内存中的常量池不是一个。class文件中的常量池主要存两个东西，字面量（Literal）和符号引用量（Symoblic Reference）， 其中字面量就包括类定义的一些常量，因为Sting是final，所以代码里面的hello字符串，就是作为字面量写在class的常量池里面  
![](https://github.com/hoyouly/BlogResource/raw/master/imgesefe43f0a055aa7f36931ab417fb05811_hd.jpg)

运行程序到Test类的时候，Test.class 文件的信息就被解析到了内存的方法区，class文件常量池大部分数据就被直接加载到了"运行时常量池"，但是String不是， 例子中的hello一个引用会被存到同样在Non Heap去的字符串常量池（Sting Pool）里，<span style="border-bottom:1px solid red;"> 而hello本身还是和所有的对象一样，创建在heap堆区</span>，R 大的文章里，测试的结果是在新生代的Eden区，但是因为一直有一个引用驻留在字符串常量池里面，所以不会被GC清理掉。这个hell对象从生存到整个线程结束，如下图所示，字符串常量池的具体位置是在过去说的永生代里，方法去外面，  
![](https://github.com/hoyouly/BlogResource/raw/master/imgeshello_object.jpg)
**注意：** <span style="border-bottom:1px solid red;"> 这只是Test类被类加载器加载时候的情形，主线程的str变量这个时候还没有创建，但是hello的实例已经在Heap里了，对它的引用也已经在字符串常量池里了</span>

等主线程开始创建str变量的时候，JVM就会到字符串常量池里面去找，看有没有能equals("hello")的String，如果找打了，就在栈区当前栈帧的局部变量表里面创建str变量，然后把字符串常量池里对hello对像的引用复制给str变量，找不到的话，会在Heap堆重新创建一个对象，然后把引用驻留到字符串常量区，然后再把引用复制给栈帧区的局部变量表
![](https://github.com/hoyouly/BlogResource/raw/master/imges20568a6ad0ef2860746533595e400716_r.jpg)

## 定义了多个值为Hello的字符串
如果当我们定义了多个值为Hello的String，比如下面的代码，有三个变脸str1，str2,str3,也不会在堆上增加String实例，局部变量表中三个变量统一指向同一个堆内存地址。
```java
Class Test{
  public void f(String s){...}

  public static void main(String[] args){
    String str1="hello";
    String str2="hello";
    String str3="hello";
    ...
  }
}

```
![](https://github.com/hoyouly/BlogResource/raw/master/imges8e743518809bd37723a4b0e8bf35f332_r.jpg)
上图str1,str2,str3可以用== 链接的，但是如果用new 关键字创建字符串，情况就不一样了。
## new 关键字创建字符串
```java
Class Test{
  public void f(String s){...}

  public static void main(String[] args){
    String str1="hello";
    String str2="hello";
    String str3=new String("hello");
    ...
  }
}

```
虽然str1 和str2 还是和之前一样，但是<span style="border-bottom:1px solid red;"> str3 因为new关键字会在Heap堆申请一块全新内存，来创建对象，</span> 虽然字面还是hello，却是完全不同的对象，有不同的内存地址。
![](https://github.com/hoyouly/BlogResource/raw/master/imgesfe6b27f35b5491eb562138eda573c238_r.jpg)
当然，String #intern()方法让我们手动检查字符常量池，把有新字面量的字符串地址驻留到常量池里。
## 答案
看上面划红线的地方，相信可以知道问题二的答案了： "abc" 和 new String() 都是放在Heap 堆里面，而 变量 s 是存放在 栈中的栈帧里面的局部变量表里面的。


## 后续补充
今天看关于java 是值传递还是引用传递的时候，无意中在知乎中看到了这个问题，[Java中，关于String类型的变量和常量做“+”运算时发生了什么？](https://www.zhihu.com/question/35014775)   和String 有关，觉得有点意思，就搬运过来，
问题如下
```
㈠
   String s1="a"+"bc";
   String s2="ab"+"c";
   s1==s2的结果是true

㈡
   String a="a";
   String bc ="bc";
   String s1="a"+"bc";
   String s2=a + bc;
   s1==s2的结果就是false

请问对于第㈠和第㈡种情况，在内存中发生了什么.
第一种情况下，常量池是不是创建了a，bc，ab，c和abc五个常量？s1和s2共享了abc这个常量？
第二种情况下，和第一种的不同在哪儿？
```
首先我在实验了，确实是这个结果，原因呢，可以查看知乎原贴。

原因是这里面有一个新的名字，**常量折叠**  是一种编译器优化技术，就是对于String s1="1"+"2",编译器会给你优化成String s1="12";在生成字节码中，根本看不到“1”，”2“。
常量折叠的条件
1. 必须是编译时期常量之间进行运算才能进行常量折叠。
编译时期常量就是在编译期间可以确定的常量。而且这个认定非常严格，
* 首先，字面量是编译期常量，数字字面量，字符串字面量等
* 其次，编译期常量进行简单运算的结果也是编译期常量，如1+2，"a"+"b"
* 最后，被编译器常量赋值的final的基本类型和字符串变量也是编译期常量

第一种情况，s1和s2 都是字符串常量进行相加，是编译期常量，会被编译器进行常量折叠，所以只会有一个常量，abc,且位于字符串常量池中
第二种情况 s1 是字符串常量相加，会被常量折叠，而s2 却是两个非final变量进行相加，不会进行常量折叠，而是根据String 类特有的“+”运算符重载，变成类似这样的代码
`String s2=new StringBuffer(a).append(b).toString()`  
这里的toString()会生成新的 String常量，而不是常量池中“abc”的引用，显然用“==”运算符会返回false



---
搬运地址：  
[请别再拿“String s = new String("xyz");创建了多少个String实例”来面试了吧](http://rednaxelafx.iteye.com/blog/774673)  
[String s=new String("abc")创建了几个对象?](https://www.cnblogs.com/ydpvictor/archive/2012/09/09/2677260.html)  
[java用这样的方式生成字符串：String str = "Hello"，到底有没有在堆中创建对象？ - 胖胖的回答](https://www.zhihu.com/question/29884421/answer/113785601)    
