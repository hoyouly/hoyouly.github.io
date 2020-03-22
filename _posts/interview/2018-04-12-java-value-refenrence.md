---
layout: post
title: 街题系列 - Java 是值传递还是引用传递
category: 街题系列
tags: Java  值传递  引用传递
---
* content
{:toc}

## 前言
这个也是经常遇到的一个题目吧，可能是一道笔试题，给一段代码，问输出结果，也可能是直接被问到的，我就遇到了一个笔试题,问输入结果，虽然我知道答案，可是被问到为啥的时候，却不能打出来一个所以然，先说笔试题内容。

```java
public static void main(String[] args) {
    String str = new String("abc");
    char[] c = new char[] { 'a', 'b', 'c' };
    change(str, c);  
    System.out.println("str= "+str);
    System.out.println("c= "+String.valueOf(c));
}
public static void change(String str, char[] c) {
    str = "123";
    c[0] = 'd';
}
```
先公布这个结果吧  

```
 str= abc
 c= dbc
```
为毛线啊，不着急，且听我慢慢道来。。  
话说 Java 的数据类型分为两种
* 基本数据类型
  * 整型 ：byte,int，short,long
  * 浮点型： float,double
  * 字符型： char
  * 布尔型： boolean
* 引用数据类型
  * 数组
  * 类
  * 接口

* 基本数据类型，值就直接保存到变量中   
* 引用数据类型，变量中保存中的是实际对象的地址。一般称为这种变量为引用。引用指向实际类型，实际类型保存着内容，

局部变量和方法参数都是JVM在栈中开启空间存储的，随着方法进入开辟，退出回收。以32位JVM为例，boolean/byte/int/short/char/float 以及引用都是在内存中开辟4个字节的空间，long/double 则是8个字节的空间，对于每个方法来说，最多占据多少空间是一定的，这在编译时期都计算好了，

每一个线程都分配一个独享的栈，所有的线程共享一个堆，对于每个方法的局部变量，是觉得不可能被其他方法，甚至其他线程的同一个方法所访问的，更别说修改了。

我们在方法中声明一个int i=0;或者Object obj=null的时候，仅仅涉及到栈，不影响到堆。当我们new Object()的时候，实际上是在堆中开辟一个块内存并初始化Object对象，当我们将这个对象赋值给obj变量的时候，仅仅是栈中代表obj的四个字节变更为这个对象的地址。

方法的参数分为
* 形式参数  定义方法时写的参数
* 实际参数  调用方法时写的具体参数

一般情况下，在数据作为参数传递的时候，
* 基本数据类型是值传递，
* 引用数据类型是引用传递（地址传递）
* String,Integer,Double 等Immutable类型因为没有提供自身修改的函数，所以每次操作都是生成一个新的对象，所以需要特殊对待，可以认为是值传递  

接下来一个一个通过代码解释：
## 基本数据类型
```java
public static void main(String[] args) {
    int num1 = 10;
    int num2 = 20;

    swap(num1, num2);

    System.out.println("num1 = " + num1);
    System.out.println("num2 = " + num2);
}

public static void swap(int a, int b) {
    int temp = a;
    a = b;
    b = temp;

    System.out.println("a = " + a);
    System.out.println("b = " + b);
}
```
结果为
```java
a = 20
b = 10
num1 = 10
num2 = 20
```
## 解析
因为a,b 是从num1和num2 复制过来的，所以a,b 不管怎么改变，不会影响到num1,num2,这就相当于你拿着身份证复印件去办理一项业务，很多情况下，为了安全，我们都会在身份证复印件上写上“该复印件只能用来办理xxx业务”之类的话，写的这话显示到你的身份证复印件上，可是你的身份证上并没有因此改变一个道理。只要记住是复制，不是本身，就好理解。

## 引用数据类型
```java
public  static class Employee {
    public int age;
  }
  // 创建两个线程，交替打印数字

  public static void main(String[] args) {
    Employee employee = new Employee();
    employee.age = 10;
    changeEmployee(employee);
    System.out.println("age = "+employee.age);
  }
  public static void changeEmployee(Employee emp ) {
    emp = new Employee();
    emp.age = 50;
    System.out.println("changeEmployee  age = "+emp.age);
  }
```
运行结果如下：

```java
changeEmployee  age = 1000
age = 100
```
然后我们再添加一些log,查看运行结果
```java
public  static class Employee {
    public int age;
  }
  // 创建两个线程，交替打印数字

public static void main(String[] args) {
    Employee employee = new Employee();
    employee.age = 10;
    System.out.println("employee : "+employee);
    changeEmployee(employee);
    System.out.println("employee: "+employee+"    age = "+employee.age);
  }
public static void changeEmployee(Employee emp) {
    System.out.println("changeEmployee before"+emp);

    emp = new Employee();
    System.out.println("changeEmployee end "+emp);

    emp.age = 50;
    System.out.println("changeEmployee emp = "+emp+"  age: "+emp.age);
  }

```
运行结果如下：
```java
employee : top.hoyouly.sina.JavaReferenceTest$Employee@6a998c1
changeEmployee beforetop.hoyouly.sina.JavaReferenceTest$Employee@6a998c1
changeEmployee end top.hoyouly.sina.JavaReferenceTest$Employee@686baa51
changeEmployee emp = top.hoyouly.sina.JavaReferenceTest$Employee@686baa51  age: 50
employee: top.hoyouly.sina.JavaReferenceTest$Employee@6a998c1    age = 10
```
这应该能看出来点眉目吧
## 解析
* 因为 Employee 是一个类，创建该对象，这个对象会存入到堆中一个地址里面，而这个地址就是 6a998c1，然后把该对象赋值给变量employee，于是employee这个变量指向这个对象的地址，即6a998c1，  
* 引用数据类型在参数传递的时候传递的是引用地址，也就是在执行changeEmployee()方法时，变量employee会把引用的地址复制一份给emp，于是changeEmployee()方法第一行输出的emp地址就和变量employee变量一样
* 然而changeEmployee()的第二行确实重新再创建一个对象，在堆内存的地址是6a998c1，然后把这个对象赋值给emp,也就是从这一刻开始，emp指向的地址变量由原来的6a998c1变成了6a998c1，那么以后对emp的操作，都是对686baa51 地址的操作。所以尽管emp设置了age为50，但是这个设置是对686baa51地址的对象操作的，而非6a998c1地址的对象，
* changeEmployee() 执行完成后，尽管686baa51对象的age值变化了，可是6a998c1 对象的age依旧没改变，还是10  

举个栗子：   
你说你会清理抽烟机，我刚好需要，就告诉你我家地址。而你知道我家地址，可是竟然没来我家，去了另外一家，然后把油烟机清理干净了，但是我家的油烟机还是脏兮兮的啊，其实就是这个道理。
1. 知道我家的地址，即Employee employee = new Employee();   
2. 发现我家油烟机很脏   employee.age = 10;   
3. changeEmployee() 可以理解为清理油烟机(),接受一个家庭地址   
4. 我把我家地址给你了，执行 changeEmployee(employee);  
5. 你知道了我家地址，    System.out.println("changeEmployee before"+emp);  
6. 可是你竟然私自做主张，更换地址，emp = new Employee();      
7. 那么就算你把油烟机清理的再干净，emp.age = 50; 那也不是我家的啊，我家的还是脏兮兮的，age = 10   

接下来我们再改变一下代码
```java
public  static class Employee {
    public int age;
  }
  // 创建两个线程，交替打印数字

  public static void main(String[] args) {
    Employee employee = new Employee();
    employee.age = 10;
    System.out.println("employee : "+employee);
    changeEmployee(employee);
    System.out.println("employee: "+employee+"    age = "+employee.age);
  }
public static void changeEmployee(Employee emp) {
    System.out.println("changeEmployee before"+emp);

    // emp = new Employee();
    System.out.println("changeEmployee end "+emp);

    emp.age = 50;
    System.out.println("changeEmployee emp = "+emp+"  age: "+emp.age);
  }
```
我只是注释了一行代码，  // emp = new Employee();，其他一样，可是结果却大不相同了，先看这次运行结果
```java
employee : top.hoyouly.sina.JavaReferenceTest$Employee@2f63e9a1
changeEmployee beforetop.hoyouly.sina.JavaReferenceTest$Employee@2f63e9a1
changeEmployee end top.hoyouly.sina.JavaReferenceTest$Employee@2f63e9a1
changeEmployee emp = top.hoyouly.sina.JavaReferenceTest$Employee@2f63e9a1  age: 50
employee: top.hoyouly.sina.JavaReferenceTest$Employee@2f63e9a1    age = 50
```
其实这就好理解了，还以刚才清理油烟机的例子，这次请你清理油烟机，你学乖了，不敢私自改地址了，整个运行过程中，对象的地址一直都是2f63e9a1， 乖乖来我家了，把油烟机清理干净了，那样理所当然的我家的油烟机就干净了啊。

引用数据类型包括类，数组和接口，类刚才说过了，接口也属于类的一种，就不多解释了，
然后说一下数组
### 数组类型
```java
public static void main(String[] args) {
    char[] ch = new char[] { 'a', 'b', 'c' };
    change(ch);
    System.out.println("ch= "+String.valueOf(ch)+"  ch的地址： "+ch);
  }
  public static void change( char[] c) {
    c[0] = 'd';
    System.out.println("c[0]: "+c[0]+"  c :"+c);
  }
```
运行结果:
```java
c[0]: d  c :[C@528f2588
ch= dbc  ch的地址： [C@528f2588
```
看到了没，change()中c的地址和main()中ch地址一样，所以在change()中对c的修改直接影响main()中ch,所以ch的值就变成了dbc ,

然后看第三种，特殊的类，没有提供自身修改的函数的类，例如 String,Integer,Double 等Immutable类型

## 没有提供自身修改函数的类
说最简单的吧，String类型，
```java
public static void main(String[] args) {
    String str = new String("abc");
    System.out.println("before  str: "+str.hashCode()+"   str: "+str);
    change(str);
    System.out.println("end str: "+str.hashCode()+"   str: "+str);
}

public static void change(String s) {
    System.out.println("before s "+s.hashCode()+"  s: "+s);
    s = "123";
    System.out.println("end  s "+s.hashCode()+"  s: "+s);
  }
```
运行结果：
```java
before  str: 96354   str: abc
before s 96354  s: abc
end  s 48690  s: 123
end str: 96354   str: abc
```
同样，进入到change()后，s 的hashcode变了，可以理解为不是同一个对象了，change()中s值该为“123”，但是str 并没有改变。

## 解析
String 中API中有这样一句话: "their values cannot be changed after they are created",意思是 String类的值创建后就不能被改变了。也就是说对String对象s的任何修改都等同于创建一个新对象，并将新的地址赋给s。String 对象一旦创建，内容不可更改，每一次内容更改都是重新创建新的对象。

---
搬运地址：    

[JAVA中值传递和引用传递的三种情况](https://blog.csdn.net/zhzhao999/article/details/53449504)  

[Java 到底是值传递还是引用传递？](https://www.zhihu.com/question/31203609)
