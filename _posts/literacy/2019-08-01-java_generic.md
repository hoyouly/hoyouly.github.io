---
layout: post
title: 扫盲系列 - Java 泛型
category: 扫盲系列
tags: 泛型 Java
---
* content
{:toc}
# 泛型
作用： 编译时期做类型检查以及自动转型。
## 泛型类
包括泛型类，泛型接口，泛型方法

## 泛型方法

```java
public <T> String getString(T parm) {
       return parm.toString();
}
```
## 泛型通配符
### <?>
可以接受任何类型的数据。
* 只能在声明的时候使用，不能在创建的时候使用，new ArrayList这样是不可以的，Lit<?> list=null,这样是允许的。
* List<?> 能给他赋值任何类型的List，但是不能给他赋值任何类型的数据，因为不知道他存的到底是什么类型。

```java
List<Object> objectList=new ArrayList<String>(); //错误，

List<?> list=new ArrayList<String>(); //正确   
//？表示可以是任何类型但是不能确定是什么类型，

list.add("hello");//错误   
//list.add(T t)这个方法的参数是T，声明list的时候传入的参数化类型是"?",
//而？表示不确定是什么类型，不知道是什么类型所以不能使用这个方法。

Object o = list.get(0);//正确，
//？表示不确定是什么类型，但是所有的类型都继承Object
```
### 带有下边界的通配符
`<? extends Number>  ? 只能是Number的类或者子类 `
```java  
List<? extends Number> list1;
list1=new ArrayList<Integer>(); //正确 Integer 继承 Number
list1=new ArrayList<Float>();  //正确  Float 继承Number
list1=new ArrayList<String>(); //错误，因为String 不是Number的子类
Number number = list1.get(0); //正确，因为返回的类型肯定是Number或者子类，使用父类接收是可以的
list1.add(new Float(0)); //错误，泛型上界，只能取(get)，不能添加(add)。
```
<font color="#ff000" >只能取(get)，不能添加(add)。</font>

因为Java是强引用，所以任何变量在使用的时候，必须知道其类型，虽然list1.add(new Float(0)) 看起来放入的是Float 类型，但编译器它却不知道这个list到底放的是啥类型，有可能是`ArrayList<Short>,ArrayList<Test>` 等，因为不确定所以就不允许你add一个long或者short。

### 带有上边界的通配符
`<? super Integer>   ? 只能是Number或者Number的父类 `
```java
List<? super Integer> list2;
list2=new ArrayList<Integer>();//正确
list2=new ArrayList<Number>(); //正确 Number是父类

list2.add(new Integer(0)); //正确 ， Integer 是子类，

Integer object = list2.get(0); //错误
//因为只能确定返回的是Integer或者父类，但是不能确定是哪个父类，所以使用Integer接受就错误，只能使用Object接受

Object object1 = list2.get(0); //正确  
```
<font color="#ff000" >不能取(get)，只能添加(add)</font>

### PECS原则总结
- 如果要从集合中读取类型T的数据，并且不能写入，可以使用 ? extends 通配符；(Producer Extends)
- 如果要从集合中写入类型T的数据，并且不需要读取，可以使用 ? super 通配符；(Consumer Super)
- 如果既要存又要取，那么就不要使用任何通配符。


## 泛型擦除
java中的泛型只存在编译时期，运行时，java虚拟机并不知道泛型的存在，那么带来的问题就是，多态失效
```java
  public void test(List<Integer> value){

   }

   public void test(List<String> value){

   }
```
这个就编译不过，原因就是泛型擦除。两个方法签名一样了。
## 常见问题
### List ,`List<Objec>` ,List<?> 的区别
* `List<Objec>` 的引用只能指向List对象，不能指向除了Object类型之外的其他对象，而List<?> 和List可以
* List 和`List<Objec>`中可以添加任何类型的对象，因为所有的类型都是Object的子类，但是List<?> 不可以，虽然他可以指向任何类型的对象，但是不能确定到底是哪种类型，所以不能往里面添加任何对象，但是可以往从里面取对象，取出来的对象都是Object类型
* 编译器不会对List 进行检查，但是会对List<?> 和`List<Objec>`进行类型检查
看下图，就能清楚看到   
![](../../../../images/list_object_comparae.png)


### ？和T 的区别
？代表的是通配符，代表未知类型，
T 代表自定义类型，指一种特定的类型
### Class <T extends Comparable<? super T> >  意思
Class 中的类型是 Comparable或者Comparable的子类 T
Comparable 中是 T或者T的父类

```java
class Apple implements Comparable<Apple> {
      public int weight;

      @Override
      public int compareTo(Apple o) {
          return weight - o.weight;
      }
}

class RedApple extends Apple  {}

class MyData<T extends Comparable<T>> {}

public void test() {
    MyData<Apple> data = null;// ok
    MyData<RedApple> data1 = null; //编译失败
}
```
为啥 MyData<RedApple> data1 = null编译失败了呢

前面已经定义了 MyData<T extends Comparable<T>>，RedApple 就是T，那么替换就是

MyData<RedApple extends Comparable<RedApple>>

虽然RedApple 继承了Apple，Apple 又实现 Comparable 接口，也就相当于 RedApple 实现了Comparable 可是Comparable 中的泛型却是 Apple，而不是RedApple，所以编译失败
当我们在RedApple中重写compareTo()就知道了
```java
class RedApple extends Apple  {
    @Override
    public int compareTo(Apple o) { //类型是Apple而不是RedApple
        return super.compareTo(o);
    }
}
```

### 为啥不能创建泛型数组
因为java使用擦除实现的泛型，在运行的时候类型参数会被擦除。无法知道确切的类型，而对于java数组来说，必须知道它持有的所有对象的类型，因此不能创建泛型数组.

但是能不能曲线救国呢？
```java
String[] strings = new String[9]; // 正常
List[] arr0 = new ArrayList[9]; //正常

List<String>[] arr=new ArrayList<String>[9];//错误，

List<String>[] arr1 = (ArrayList<String>[]) new ArrayList[9]; //正确，
//通过强转，绕过编译器实例化检查

arr1[0]=new ArrayList<String>();//正确
arr1[1]=new ArrayList<Integer>(); //错误，
//类型不合法，因为通过强转，java已经获悉这个数组在运行时要应持有的具体数据类型，
```
参数化数组本身的类型，
不在参数化数组持有的类型，而是将数组应该存储的对象进行参数化。
```Java
public static  <T> T[] creatArr(T[] arg) {
    return arg;
}

public static void main(String[] args) {
    Integer[] arr={1,2,3,4};
    String[] arrStr={"a","b","C"};
    System.out.printf(Arrays.toString(creatArr(arr)));
    System.out.printf(Arrays.toString(creatArr(arrStr)));
}
```
无论我们传入Integer类型数组或String类型数组或自定义对象数组，该参数化方法总能正确的执行，即数组本身的类型被参数化了，但在运行时，被赋予了具体的类型。
虽然能通过参数化本身的类型，但是也不能直接创建泛型数组，只能通过强转
```java
public static <T> T[] creatArr(int size) {
   return (T[]) new Object[size];
}
```
### 静态变量不能是泛型类型
解释一: 泛型在创建的时候才知道是什么类型，而对象创建的代码执行先后顺序是 先static，后才是构造函数，所以在对象初始化之前，static部分已经执行了，如果在静态变量中设置泛型，java虚拟机就不知道是什么鬼东西了，因为这个时候对象还没创建呢，因此在静态方法、数据域或初始化语句中，为了类而引用泛型类型参数是非法的。

解释二：因为静态成员变量属于一个类，所有的对象持有一份，如果静态成员变量能够使用参数类型，那么不同的对象造成不同的类型，这样就造成了冲突

---
搬运地址：   
[关于java泛型大大小小的那些事](https://juejin.im/post/5ae2d31ef265da0b9a69b0da)   
[浅谈泛型数组](https://www.cnblogs.com/MrJR/p/10463479.html)
