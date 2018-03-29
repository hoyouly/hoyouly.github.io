---
layout: post
title: 扫盲系列之---Data Binding
category: 扫盲
tags: DataBinding
---
* content
{:toc}

# 开启DataBinding
在Moudle下面的build.gradle  中声明
```java
android {
     …………
     dataBinding{
        enabled = true
     }    
     …………
}
```
这样就开启DataBinding ,so easy
# layout 标签
作用： 作为DataBinding的标志，省去findViewById()方法
```java
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools">
    <data>
        <!--
            取一个name,并将她的type指向一个Bean，这样就可以绑定该对象了，
            使用的时候，通过@{}格将该对象的属性绑定到控件中式-->
        <variable
            name="student"
            type="top.hoyouly.framework.bean.Student"/>

        <variable
            name="controller"
            type="top.hoyouly.framework.mv.StudentActivity.Controller"/>

    </data>
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <EditText
            android:onTextChanged="@{controller.onChanged}"
            android:id="@+id/et_name"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

        <EditText
            android:id="@+id/et_nickName"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

        <!--@{student.name} 把Student中names属性的值设置给tv_name -->
        <TextView
            android:onClick="@{controller.onClicked}"
            android:id="@+id/tv_name"
            android:text="@{student.name}"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

        <!--@{student.nickName} 把Student中nickName属性的值设置给 tv_nickName -->
        <TextView
            android:id="@+id/tv_nickName"
            android:text="@{student.nickName}"
            android:onClick="@{()->controller.onClickListener(student)}"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

    </LinearLayout>

</layout>
```
# data 标签
在layou 标签下面，通过<data>节点来引入我们要使用的数据源

# variable 节点，
定义在data标签里面的，可以有多个，

```java
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools">
    <data>
        <!--
            取一个name,并将她的type指向一个Bean，这样就可以绑定该对象了，
            使用的时候，通过@{}格将该对象的属性绑定到控件中式-->
        <variable
            name="student"
            type="top.hoyouly.framework.bean.Student"/>

    </data>

  </layou>  
```
name属性表示变量的名称，type表示这个变量的类型 其实可以理解为创建了一个Student类型的实例student
也可以换一种写法：使用import 标签导入实体类，然后直接使用
```java
<data>
    <import type="top.hoyouly.framework.bean.Student"/>
    <variable
        name="student"
        type="Student"/>

</data>
```
如果有两个同名的类，那该怎么办呢，import 里有一个alias属性，表示给该类起一个别名，使用别名处理就可以了，这样就可以直接在variable中直接使用别名了。
```java
<data>
    <import type="top.hoyouly.framework.bean.Student" alias="Stu"/>
      <variable
          name="student"
          type="Stu"/>
</data>

```
设置完variable 标签后，怎么使用呢，其实很简单，只要记号，在xml中使用这些标签属性，格式就是 `@{}`中间填写的就是variable中name的名字，这个就是type类型的变量，直接使用他做操作就行了，例如``android:text="@{student.nickName}"就是把student变量中的name值设置到TextView中。

```java
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <!--@{student.name} 把Student中names属性的值设置给tv_name -->
    <TextView
        android:id="@+id/tv_name"
        android:text="@{student.name}"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <!--@{student.nickName} 把Student中nickName属性的值设置给 tv_nickName -->
    <TextView
        android:id="@+id/tv_nickName"
        android:text="@{student.nickName}"
        android:onClick="@{()->controller.onClickListener(student)}"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</LinearLayout>
```
设置完之后，那么在Activity上要做啥操作呢？

在我们定义的Activity中，把原来setContentView()这行代码换掉，换成 `ActivityStudentBinding mBinding = DataBindingUtil.setContentView(this, R.layout.activity_student);`

这里解释一下，ActivityStudentBinding 这个类是自动生成的，生成的规则和我们设置的布局文件名字有关，因为我们的布局文件叫`activity_student`,所以生成的类名字就是取出下划线，然后根据驼峰命名法，后面再加上Binding，就是新类名，这个类就是ViewModel，
有了这个ViewModle,我们就可以处理数据了
```java
    student = new Student();
    student.setName("何磊");
    student.setNickName("hoyouly");
    mBinding.setStudent(student);
```

这里面就奇怪了，这个setStudent()方法是从哪里来的呢，其实还记得我们之前定义了一个variable，里面有一个name属性，值是student，这个setStudent()就是根据这个student变量，设置的set方法，
其实这个mBinding功能还不仅仅是这些，他可以直接取得我们设置的控件，只要添加有id的控件，都能通过mBinding得到，例如设置了一个`  <EditText
      android:id="@+id/et_nickName"`,那么直接通过 `mBinding.etNickName`就得到这个Edittext，规则也是根据id的名字，然后驼峰命名法则取得。

# @{}进行简单的计算

## 基本的三目运算
```java
<TextView
           android:onClick="@{controller.onClicked}"
           android:id="@+id/tv_name"
           android:text="@{student.name??student.nickName}"
           android:layout_width="match_parent"
           android:layout_height="wrap_content" />
```
`@{student.name??student.nickName}`,这里面有两个问号，注意，这表示，如果username属性为null则显示nickname属性，否则显示username属性。
## 字符串拼接
```java
<TextView
            android:onClick="@{controller.onClicked}"
            android:id="@+id/tv_name"
            android:text="@{`name: `+student.name??student.nickName}"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />
```
拼接字符不是单引号，而是反单引号，也叫重音符，是在ESC上面那个键，使用Markdown语法插入代码的人肯定会经常用到这个符号，之前可能不支持中文，但是现在我测试了一下，是可以支持中文的。

## 根据数据显示样式
```java
<TextView
         android:onClick="@{controller.onClicked}"
         android:id="@+id/tv_name"
         android:text="@{`中文: `+student.name??student.nickName}"
         android:background="@{student.age>30?0xFF0000FF:0xFFFF0000}"
         android:layout_width="match_parent"
         android:layout_height="wrap_content" />
```
这里给TextView 设置背景，做了小判断，如果大于30，设置0xFF0000FF，否则背景设置成0xFFFF0000 这个颜色。

另外，DataBinding对于基本的四则运算、逻辑与、逻辑或、取反位移等都是支持的，我这里不再举例。

# 绑定ImageView
## DataBinding 自定义属性。 @BindingAdapter
通过@BindingAdapter 注解创建一个自定义属性。同时还需要有一个注解方法，当我们在布局文件中使用自定义属性的时候，会触发这个被我们注解的方法，

```java
public class Student {
	public String name;
	public String nickName;
	public List<Courses> courses;
	public int age;
  public String userface;

	@BindingAdapter("bind:userface")
	public static void getInternetImage(ImageView imageView,String userface){
        Picasso.get().load(userface).into(imageView);
    }
}

```

主要看getInternetImage（）方法。
1. 使用了注解 BindingAdapter，该注解表示用户在ImageView使用userface属性的时候，会触发这个方法
2. 该方法必须是静态方法

```java
<ImageView
           android:id="@+id/iv"
           app:userface="@{student.userface}"
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"/>

```
ImageView控件中使用userface属性的时候，使用的前缀不是android而是app哦

然后在Activity中怎么设置呢
```java
student.userface="http://img2.cache.netease.com/auto/2016/7/28/201607282215432cd8a.jpg";
mBinding.setStudent(student);

```




[告别findView和ButterKnife](https://www.jianshu.com/p/499c8e2b80c4)  
[DataBinding实用指南](https://www.jianshu.com/p/015ad08c2c75)  
[ Android基础——框架模式MVVM之DataBinding的实践](https://blog.csdn.net/qq_30379689/article/details/53037430)  
[玩转Android之MVVM开发模式实战，炫酷的DataBinding！](https://blog.csdn.net/u012702547/article/details/52077515)
