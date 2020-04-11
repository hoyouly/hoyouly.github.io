---
layout: post
title: Android 优化方案
category: 读书笔记
tags: Android  ListView  性能优化
---
* content
{:toc}

## ListView优化方案
1. 复用 contentView ,减少重新分配缓存造成的内存频繁分配回收
2. ViewHolder的使用，主要是因为 findViewById() 消耗时间太大，使用 setTag() , getTag() 得到view
3. 图片加载优化 监听 Listview 的滑动状态，滑动的时候（SCROLL_STATE_TOUCH_SCROLL）和惯性滑动的时候（SCROLL_STATE_FLING）不加载，停止滑动的时候再加载。listView.setOnScrollListener();
4. onClickListener的处理，通常是在 getView 方法时候处理，但是这样的话，每次调用 getView() 都会设置 onClickListener() 效率低，可以在 ViewHolder 中设置一个 Position ，然后实现 OnClickListener

```java
class  ViewHolder implements OnClickListener{  
    int position;  
    TextView name;  

    public void setPosition(int position){  
        this.position = position;  
    }  

    @Override  
    public void onClick(View v) {  
        switch (v.getId()){  
            //XXXX  
        }  
    }  
}  

public View getView(int position, View convertView , ViewGroup parent) {  
    ViewHolder holder = null;  
    if(convertView==null){  
        convertView = inflater.inflate(R.layout.list_item, parent , false);  
        holder = new ViewHolder();  
        holder.name = (TextView) convertView.findViewById(R.id.name);  
        holder.name.setOnClickListener(this);  
        convertView.setTag(holder);  
    }else{  
        holder = (ViewHolder) convertView.getTag();  
    }  
    //设置holder  
    holder.name.setText(list.get(position).partname);  
    //设置position  
    holder.setPosition(position);  
    return convertView;  
}  
```
5. 减小 item view 的布局层级关系，布局层级过深，直接导致 View 的测量和绘制时间加大
6. adapter中的 getView() 方法尽量少使用逻辑,可以想办法把逻辑抽离到其他地方
7. getView()中尽量少做耗时操作，避免创建大量对象
8. 善用自定义 View ，自定义 View 可以很好的控制层级，并且对绘制过程可以很好的控制。
9. 使用 RecycleView 代替， ListView 每次更新数据都要 notifyDataSetChanged() ，有些太暴力了。 RecycleView 在性能和可定制性上都有很大的改善，推荐使用。

## 性能优化方案
1. 避免创建不必要的对象
2. 合理使用 static 成员
3. 避免内部的 setters 和 getters 方法，
4. 使用增强 for 循环
5. 使用 package 代替 private 以便私有内部类高效访问外部类成员
6. 合理使用浮点类型  浮点型大概比整型数据处理速度慢两倍，所以如果整型可以解决的问题就不要用浮点型。
7. 采用<merge>优化布局层数。 采用<include＞来共享布局。
8. 延时加载View. 采用 ViewStub 避免一些不经常的视图长期被引用，占用内存.
9. 移除 Activity 默认背景，提升 activity 加载速度。
10. cursor 的使用。 不要每次打开关闭cursor.因为打开关闭 Cursor 非常耗时。 不再使用的 cursor 要记得关闭(一般在 finally 语句块执行)。
11. 广播 BroadCast 动态注册时，记得要在调用者生命周期结束时 unregisterReceiver ,防止内存泄漏。
12. 针对 ListView 的性能优化
13. 注意使用线程的同步机制（synchronized）
14. 合理使用 StringBuffer , StringBuilder ,String
15. 尽量使用局部变量 临时变量都保存在栈（Stack）中，速度较快。其他变量，如静态变量、实例变量等，都在堆（Heap）中创建，速度较慢。另外，依赖于具体的编译器/JVM，局部变量还可能得到进一步优化。
16. I/O流操作记得及时关闭流对象。
17. 使用 IntentService 代替Service
IntentService 和 Service 都是一个服务，区别在于 IntentService 使用队列的方式将请求的 Intent 加入队列，然后开启一个worker thread(线程)来处理队列中的Intent（在 onHandleIntent 方法中），对于异步的 startService 请求， IntentService 会处理完成一个之后再处理第二个，每一个请求都会在一个单独的 worker thread 中处理，不会阻塞应用程序的主线程，如果有耗时的操作与其在 Service 里面开启新线程还不如使用 IntentService 来处理耗时操作。
18. 使用 Application Context 代替 Activity 中的Context
19. 集合中的对象要及时清理
20. Bitmap的使用
21. 巧妙的运用软引用（SoftRefrence）
22. 尽量不要使用整张的大图作为资源文件，尽量使用 9path 图片

### 布局优化

* 布局复用，使用<include>标签重用layout；
* 提高显示速度，使用<ViewStub>延迟 View 加载；
* 减少层级，使用<merge>标签替换父级布局；
* 注意使用 wrap_content ，会增加 measure 计算成本；
* 删除控件中无用属性；

### 绘制优化
* 布局上的优化。移除 XML 中非必须的背景，移除 Window 默认的背景、按需显示占位背景图片
* 自定义 View 优化。使用 canvas.clipRect()来帮助系统识别那些可见的区域，只有在这个区域内才会被绘制。

### 启动优化
* 启动加载逻辑优化。可以采用分布加载、异步加载、延期加载策略来提高应用启动速度。
* 数据准备。数据初始化分析，加载数据可以考虑用线程初始化等策略。

### 刷新优化
* 减少刷新次数；
* 缩小刷新区域；

### 动画优化
在实现动画效果时，需要根据不同场景选择合适的动画框架来实现。有些情况下，可以用硬件加速方式来提供流畅度。

### 耗电优化
* 计算优化，避开浮点运算等。
* 避免 WaleLock 使用不当。
* 使用 Job Scheduler 。

### APK瘦身
应用安装包大小对应用使用没有影响，但应用的安装包越大，用户下载的门槛越高
减小安装包大小可以让更多用户愿意下载和体验产品。

* 代码混淆。使用 proGuard 代码混淆器工具，它包括压缩、优化、混淆等功能。
* 资源优化。比如使用 Android Lint 删除冗余资源，资源文件最少化等。
* 图片优化。比如利用 AAPT 工具对 PNG 格式的图片做压缩处理，降低图片色彩位数等。或者使用 WebP 图片格式
* 避免重复功能的库
* 插件化。比如功能模块放在服务器上，按需下载，可以减少安装包大小。
* 清理 assert 文件夹下无用的文件
* 根据实际情况进行多分辨率的支持，如果 APP 只是针对某种特定的机型，只需要一套图片即可
* 如果引用第三方 aar ,使用到了他们的资源，但是如果我们只想使用 mdpi ,那么可以根据情况剔除其他的，在
  ```
  defaultConfig{
    resConfigs "mdpi"
  }
  ```

---   
搬运地址：    
 

[Android优化】最强 ListView 优化方案](http://blog.csdn.net/gs12software/article/details/51173392)

[Android APP性能优化(最新总结)](http://blog.csdn.net/csdn_aiyang/article/details/74989318)

[Android开发性能优化总结(一)](http://blog.csdn.net/gs12software/article/details/51173392)
