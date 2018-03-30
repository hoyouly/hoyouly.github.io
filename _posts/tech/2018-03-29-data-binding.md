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

# 绑定ListView
首先看xml布局文件

```java
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <ListView
            android:id="@+id/lv"
            android:layout_width="match_parent"
            android:layout_height="match_parent"></ListView>
    </LinearLayout>

</layout>

```
这个布局文件没啥可说的，就是一个ListView。与我们之前常见的唯一的区别就是最外层是layout标题，然后看看item 的布局，
```java
<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    <data>
        <variable
            name="benefit"
            type="top.hoyouly.framework.bean.BenefitBean"/>
    </data>

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="96dp"
        android:orientation="vertical">

        <ImageView
            android:id="@+id/iv"
            android:layout_width="96dp"
            android:layout_height="96dp"
            android:padding="6dp"
            app:img="@{benefit.url}"/>

        <TextView
            android:id="@+id/description"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginLeft="8dp"
            android:layout_toRightOf="@id/iv"
            android:ellipsize="end"
            android:maxLines="3"
            android:text="@{benefit.desc}"/>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginLeft="8dp"
            android:layout_toRightOf="@id/iv"
            android:layout_alignParentBottom="true"
            android:layout_marginBottom="2dp"
            android:text="@{benefit.who}"
            android:textStyle="bold"/>
    </RelativeLayout>
</layout>
```
这个应该也看懂吧，定义的实体类是BenefitBean,实例名是benefit，然后就是一些设置。ImageView中设置图片的路径，TextView中设置文本，
加载图片的方法写到任何类中都可以的，例如我写了一个专门加载图片的Util类，只要符合规则就行。一个是静态，二是使用BindingAdapter注解，自定义属性 img 就可以了。
```java
public class ImageUtils {

	@BindingAdapter("bind:img")
	public static void loadInternetImage(ImageView iv, String img) {
		Picasso.get().load(img).into(iv);
	}
}
```
照样可以显示图片的。这样的话，我们就可以把所有处理图片的方法归纳到一个类里面了。

接下来看Adapter怎么处理的，这个可NB了，
```java
public class MyBaseAdapter<T> extends BaseAdapter {
	private Context context;
	private LayoutInflater inflater;
	private int layoutId; //layoutId这个表示item布局的资源id
	private int variableId;//variableId是系统自动生成的，根据我们的实体类，直接从外部传入即可
	private List<T> list;

	public MyBaseAdapter(Context context, int layoutId, List<T> list, int resId) {
		this.context = context;
		this.layoutId = layoutId;
		this.list = list;
		this.variableId = resId;
		inflater = LayoutInflater.from(context);
	}

	public void setList(List<T> list) {
		this.list = list;
		notifyDataSetChanged();
	}

	@Override
	public int getCount() {
		return list.size();
	}

	@Override
	public Object getItem(int position) {
		return list.get(position);
	}

	@Override
	public long getItemId(int position) {
		return position;
	}

	@Override
	public View getView(int position, View convertView, ViewGroup parent) {
		ViewDataBinding dataBinding;
		if (convertView == null) {
			dataBinding = DataBindingUtil.inflate(inflater, layoutId, parent, false);
		} else {
			dataBinding = DataBindingUtil.getBinding(convertView);
		}
		dataBinding.setVariable(variableId, list.get(position));

		return dataBinding.getRoot();
	}
}

```
看着和我们之前写的adapter很像，区别有以下几点：
1. 需要设置一个 variableId ，这个variableId 是系统自动生成的，根据我们的实体类，直接从外部传入即可，
例如我们需要对应的实体类 是top.hoyouly.framework.bean.BenefitBean，那么对应的variableId就是 top.hoyouly.framework.BR.benefit,
其实就是DataBinding自动的在我们的包名下面创建了一个BR包，里面有一个benefit。至于这个benefit 名字是怎么来的，没搞清楚
2. 注意getView()方法中，使用的是DataBindingUtil中的方法加载布局和服用布局的。
3. 这个几乎就是我们项目中的唯一一个adapter了，因为这个Adapter中没有一个变量和我们的ListView沾边，除非遇到非常奇葩的需求，你这个App中可能就只有这一个给ListView使用的Adapter了，

然后我们怎么使用呢？

```java

public class GankActivity extends BaseBindingActivity<ActivityGankBinding> {
    private List<BenefitBean> benefitBeans=new ArrayList<BenefitBean>();
    private GankLoader loader;
    private MyBaseAdapter<BenefitBean> adapter;

    @Override
    protected void initView() {
        initData();
        getListData();
    }

    private void initData() {
        loader=new GankLoader();
        adapter = new MyBaseAdapter<>(GankActivity.this, R.layout.listview_item, benefitBeans, BR.benefit);//是BR中的，这个BR和我们项目中的R文件类似，都是系统自动生成的。
        mBinding.lv.setAdapter(adapter);
    }

    private void getListData() {
        SubscriberOnNextListener<GankBean> onNextListener = new SubscriberOnNextListener<GankBean>() {
            @Override
            public void onNext(GankBean gankBean) {
                adapter.setList(gankBean.results);
            }
        };
        loader.getGankBenefitList(10,1).subscribe(new ProgressDialogSubscriber<GankBean>(onNextListener,GankActivity.this));
    }

    @Override
    protected int getLayouId() {
        return R.layout.activity_gank;
    }
}

//BaseBindingActivity.java
public abstract class BaseBindingActivity<VB extends ViewDataBinding> extends Activity {
	protected  VB mBinding;

	@Override
	protected void onCreate(@Nullable Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
    mBinding=DataBindingUtil.setContentView(this,getLayouId());
		initView();
	}

	protected abstract void initView();

	protected abstract int getLayouId();

}

```
这个应该能看懂了吧，主要是在initData()中创建一个adapter对象，里面传递context,布局资源id,集合，和variableId，即BR.benefit，然后因为我们给listview设置了id为lv，所以可以直接通过mBinding.lv得到这个listVIew，并且设置Adapter，
在getListData中，我是使用的Retrofit + OkHttp+RxJava 封装的，直接在onNext()得到数据，设置进去就可以了，这样就展示数据了，

## 给Item设置点击事件，
这个也很简单，
1. 在BenefitBean中添加一个方法
```java
public void onItemClick(View view) {  
    Toast.makeText(view.getContext(), getDescription(), Toast.LENGTH_SHORT).show();  
}  
```
注意，这个方法名字可以随便起，比如aaa，这个完全可以，但是为了规范，建议命名有含义，还有必须要注意的是，参数是固定的，写死的。其实可以理解之前在Button中设置`android:onClick="change"`这种方式设置点击事件规则就可以了，

然后在item的布局文件的根节点，添加onClick属性。

```java
<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    <data>
        <import type="top.hoyouly.framework.utils.TextUtil"/>
        <variable
            name="benefit"
            type="top.hoyouly.framework.bean.BenefitBean"/>
    </data>

    <RelativeLayout
        android:onClick="@{benefit.onItemClick}"
        android:layout_width="match_parent"
        android:layout_height="96dp"
        android:orientation="vertical">

        。。。。。。
        </RelativeLayout>
</layout>

```

在RelativeLayout 标签中，看到了吗，`android:onClick="@{benefit.onItemClick}"`，意思就是点击这个相对布局的时候，调用BenefitBean中的onItemClick()方法，

## 使用类方法

在一个类中 添加静态方法中
例如我在TextUtil中添加了一个静态方法，就是把传递过来的文本，显示两遍。
```java
public class TextUtil {

    public static String doubleWord(final String word) {
        return word+"_"+word;
    }
}
```
使用的是，只需要在 data标签下面使用import导入这个类，就可以了，然后在要使用的地方`  android:text="@{TextUtil.doubleWord(benefit.desc)}"` 完美。。

```java
<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    <data>
        <import type="top.hoyouly.framework.utils.TextUtil"/>
        <variable
            name="benefit"
            type="top.hoyouly.framework.bean.BenefitBean"/>
    </data>

    <RelativeLayout
        android:onClick="@{benefit.onItemClick}"
        android:layout_width="match_parent"
        android:layout_height="96dp"
        android:orientation="vertical">
        <TextView
            android:id="@+id/description"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginLeft="8dp"
            android:layout_toRightOf="@id/iv"
            android:ellipsize="end"
            android:maxLines="3"
            android:text="@{TextUtil.doubleWord(benefit.desc)}"/>

    </RelativeLayout>
</layout>
```

wanwa
[Android DataBinding：再见Presenter，你好ViewModel！](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0727/3220.html)



---
搬运地址：

[告别findView和ButterKnife](https://www.jianshu.com/p/499c8e2b80c4)  
[DataBinding实用指南](https://www.jianshu.com/p/015ad08c2c75)  
[ Android基础——框架模式MVVM之DataBinding的实践](https://blog.csdn.net/qq_30379689/article/details/53037430)  
[玩转Android之MVVM开发模式实战，炫酷的DataBinding！](https://blog.csdn.net/u012702547/article/details/52077515)  
[完全掌握Android Data Binding](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0603/2992.html)
