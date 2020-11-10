---
layout: post
title: Android 性能优化 -- RecycleView 优化
category: 性能优化
tags: Android开发艺术探索 性能优化
description: Android 性能优化
---
## 数据处理与视图加载分离

简单来说就是在onBindViewHolder()只设置UI显示，不做任何逻辑判断，需要的业务逻辑在得到javabean之前处理好，

```java
class Task {
    Date dateDue;
    String title;
    String description;
    // getters and setters here
}
class MyRecyclerView.Adapter extends RecyclerView.Adapter {

  static final TODAYS_DATE = new Date();
  static final DATE_FORMAT = new SimpleDateFormat("MM dd, yyyy");

  public onBindViewHolder(Task.ViewHolder tvh, int position) {
      Task task = getItem(position);

      if (TODAYS_DATE.compareTo(task.dateDue) > 0) {
          tvh.backgroundView.setColor(Color.GREEN);
      } else {
          tvh.backgroundView.setColor(Color.RED);
      }

      String dueDateFormatted = DATE_FORMAT.format(task.getDateDue());
      tvh.dateTextView.setDate(dueDateFormatted);
    }
}
```
例如这个，在onBindViewHolder中既有逻辑判断，if else 操作，又有格式化时间的操作，这些都应该在getItem()的时候，已经处理好，因为onBindViewHolder()中会频繁执行，任何多余的计算在频繁的调用中都会放大的。
下面是优化后的展示

```java
public class TaskViewModel {
    int overdueColor;
    String dateDue;
    // getters and setters
}

public onBindViewHolder(Task.ViewHolder tvh, int position) {
    TaskViewModel taskViewModel = getItem(position);
    tvh.backgroundView.setColor(taskViewModel.getOverdueColor());
    tvh.dateTextView.setDate(taskViewModel.getDateDue());
}
```
这样在onBindViewHolder()就只set操作，用来更新UI，其他的操作，都在TaskViewModel 中处理好了。这样就大大减轻了onBindViewHolder()的耗时

## 避免创建过多对象

onCreateViewHolder 和 onBindViewHolder 对时间都比较敏感，尽量避免繁琐的操作和循环创建对象。例如创建 OnClickListener，可以全局创建一个。

同时onBindViewHolder调用次数会多于onCreateViewHolder的次数，如从RecyclerViewPool缓存池中取到的View都需要重新bindView，
所以我们可以把监听放到CreateView中进行。

优化前
```java
@Override
public void onBindViewHolder(ViewHolder holder, int position) {
    holder.setOnClickListener(new View.OnClickListener() {
       @Override
       public void onClick(View v) {
         //do something
       }
    });
}
```

优化后

```java
private class XXXHolder extends RecyclerView.ViewHolder {
        private EditText mEt;
        EditHolder(View itemView) {
            super(itemView);
            mEt = (EditText) itemView;
            mEt.setOnClickListener(mOnClickListener);
        }
    }
private View.OnClickListener mOnClickListener = new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        //do something
    }
}
```

## 数据优化
1. 分页加载远端数据，对拉取的远端数据进行缓存，提高二次加载速度
2. 对于新增的或者删除的数据通过DiffUtil来进行局部刷新，而不是一味的全部刷新

###  DiffUtil
support包下新增的一个工具类，用来判断新数据和旧数据的差别，从而进行局部刷新。

在原来调用mAdapter.notifyDataSetChanged()的地方，
```java
// mAdapter.notifyDataSetChanged()
DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(new DiffCallBack(oldDatas, newDatas), true);
diffResult.dispatchUpdatesTo(mAdapter);
```
DiffUtil最终是调用Adapter的下面几个方法来进行局部刷新：
```java
mAdapter.notifyItemRangeInserted(position, count);
mAdapter.notifyItemRangeRemoved(position, count);
mAdapter.notifyItemMoved(fromPosition, toPosition);
mAdapter.notifyItemRangeChanged(position, count, payload);
```
如果必须用 notifyDataSetChanged()，那么最好设置 mAdapter.setHasStableIds(true)


## 布局优化
* 减少xml文件inflate时间
xml文件包括：layout、drawable的xml，xml文件inflate出ItemView是通过耗时的IO操作。可以使用代码去生成布局，即new View()的方式。这种方式是比较麻烦，但是在布局太过复杂，或对性能要求比较高的时候可以使用。
* 减少View对象的创建
一个稍微复杂的 Item 会包含大量的 View，而大量的 View 的创建也会消耗大量时间，所以要尽可能简化 ItemView；设计 ItemType 时，对多 ViewType 能够共用的部分尽量设计成自定义 View，减少 View 的构造和嵌套。
* 减少过渡绘制
减少布局层级，可以考虑使用自定义 View 来减少层级，或者更合理地设置布局来减少层级     
不推荐在 RecyclerView 中使用 ConstraintLayout，

## 数据Prefatch预取
升级 RecycleView 版本到 25.1.0 (android sdk>=21)及以上使用 Prefetch 功能，支持渲染（Render）线程。

RecyclerView数据显示分两个阶段：
1. 在UI线程，处理输入事件、动画、布局、记录绘图操作，每一个条目在进入屏幕显示前都会被创建和绑定view；
2. 渲染（Render）线程把指令送往GPU。
数据预取的思想就是：将闲置的UI线程利用起来，提前加载计算下一帧的Frame Buffer

在新的条目进入视野前，会花大量时间来创建和绑定view，而在前一帧却可能很快完成了这些操作，导致前一帧的UI线程有一大片空闲时间。RecyclerView开发工程师将创建和绑定移到前一帧，使UI线程与渲染线程同时工作，在一个条目即将进入视野时预取数据。

1. setItemPrefatchEnable(false)可关闭预取功能
2. 嵌套时且使用的是LinearLayoutManager，子RecyclerView可通过setInitialPrefatchItemCount设置预取个数，例如如果竖直方向的RecyclerView至少展示三个条目，调用 setInitialItemPrefetchCount(4)。
3. 二级缓存时不需要重新绑定

## 加大RecycleView的缓存
通过 RecycleView.setItemViewCacheSize(size) 来加大 RecyclerView 的缓存，用空间换时间来提高滚动的流畅性。
主要是通过以下方式
```java
recyclerView.setItemViewCacheSize(20);
recyclerView.setDrawingCacheEnabled(true);
recyclerView.setDrawingCacheQuality(View.DRAWING_CACHE_QUALITY_HIGH);
```

## 处理刷新闪烁
用notifyDataSetChange时，适配器不知道整个数据集中的那些内容以及存在，再重新匹配ViewHolder时会花生闪烁。
设置adapter.setHasStableIds(true)，并重写getItemId()来给每个Item一个唯一的ID

## 其他
* 如果item高度是固定的话，可以使用RecyclerView.setHasFixedSize(true)来避免requestLayout浪费资源。尤其是当RecyclerView有条目插入、删除时性能提升更明显。RecyclerView在条目数量改变，会重新测量、布局各个item，如果设置了setHasFixedSize(true)，由于item的宽高都是固定的，adapter的内容改变时，RecyclerView不会整个布局都重绘。
* 通过重写 RecyclerView.onViewRecycled(holder) 来回收资源
* 如果多个 RecycledView 的 Adapter 是一样的，比如嵌套的 RecyclerView 中存在一样的 Adapter，可以通过设置 RecyclerView.setRecycledViewPool(pool); 来共用一个 RecycledViewPool。
* 增加RecyclerView预留的额外空间
额外空间：显示范围之外，应该额外缓存的空间

```java
new LinearLayoutManager(this) {
    @Override
    protected int getExtraLayoutSpace(RecyclerView.State state) {
        return size;
    }
};
```
* 设置RecyclerView.addOnScrollListener()来在滑动过程中停止加载的操作。
* 对 ItemView 设置监听器，不要对每个 Item 都调用 addXxListener，应该大家公用一个 XxListener，根据 ID 来进行不同的操作，优化了对象的频繁创建带来的资源消耗。
* 如果不要求动画，可以通过 ((SimpleItemAnimator) rv.getItemAnimator()).setSupportsChangeAnimations(false); 把默认动画关闭来提神效率。


---
搬运地址：    

[RecyclerView性能优化](https://www.cnblogs.com/zgz345/p/13436005.html)   

[RecyclerView 性能优化 | 安卓 offer 收割基](https://www.jianshu.com/p/aedb2842de30)   

[RecyclerView性能优化及高级使用](https://blog.csdn.net/smileiam/article/details/88396546)
