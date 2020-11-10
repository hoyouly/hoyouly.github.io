---
layout: post
title: 源码分析 - RecycleView 的缓存复用机制
category: 源码分析
tags: RecycleView
---
<!-- * content -->
<!-- {:toc} -->
RecycleView 分为四级缓存

核心代码在Recycler中，它是RecycleView的一个内部类，用来缓存屏幕内ViewHolder以及屏幕外的Viewholder，部分代码如下

![添加图片](../../../../images/recycler_source.png)

 Recycler 的缓存机制就是通过上图中的这些数据容器来实现的，实际上Recycler的缓存也是分级处理的。

 根据访问优先级分为以下四级
 * 第一级缓存：mAttachedScrap和mChangedScrap，主要缓存屏幕内的Viewholder
 * 第二级缓存：mCachedViews，主要缓存屏幕外的Viewholder
 * 第三级缓存：ViewCacheExtension，用户自定义的缓存策略
 * 第四级缓存：RecycledViewPool，缓存屏幕外的 ViewHolder

![添加图片](../../../../images/recycler_cache.png)

前两级的数据都是干净的，可以直接拿过来使用的

第三层缓存是用户自定义的缓存策略

第四层的RecycledViewPool 的数据是脏的，不执行onCreateViewHolder，但是需要重新绑定,会执行 onBindViewHolder() 方法

## 第一级缓存 mAttachedScrap&mChangedScrap

这是两个名为Scrap的ArrayList<ViewHolder>，
* mAttachedScrap：
主要用在插入或是删除itemView时.
1. 先把屏幕内的ViewHolder保存至AttachedScrap中，作用在LayoutManager中，它仅仅把需要从ViewGroup中移除的子view设置父view为null，从而实现了从RecyclerView中移除操作detachView()。
2. 需要新插入的view从cacheView/Pool中找，没找到则createViewHolder()。
3. 而从ViewGroup中移除的子view会放到Pool缓存池中,如下图中的itemView b。

![添加图片](../../../../images/attached_scrap.png)

* mChangedScrap：
主要用到刷新屏幕上的itemView数据，它不需要重新layout，notifyItemChanged()或者notifyItemRangeChanged()


前面说了这级主要缓存屏幕内的Viewholder。可是为什么屏幕内的数据需要缓存？

主要是为了在下拉刷新的时候，只需要在原有的 ViewHolder 基础上进行重新绑定新的数据 data 即可，而这些旧的 ViewHolder 就是被保存在 mAttachedScrap 和 mChangedScrap 中。实际上当我们调用 RV 的 notifyXXX 方法时，就会向这两个列表进行填充，将旧 ViewHolder 缓存起来。

## 第二级缓存 mCachedViews

保存最近移出屏幕的ViewHolder，包含数据和position信息，复用时必须是相同位置的ViewHolder才能复用，应用场景在那些需要来回滑动的列表中，当往回滑动时，能直接复用ViewHolder数据，不需要重新bindView。

通常情况下刚被移出屏幕的 ViewHolder 有可能接下来马上就会使用到，所以 RecycleView 不会立即将其设置为无效 ViewHolder，而是会将它们保存到 cache 中，这个就是cache指的就是mCacherViews, 但又不能将所有移除屏幕的 ViewHolder 都视为有效 ViewHolder，它的默认容量只有 2 个。

所以这级缓存的特点如下：
* 主要缓存屏幕外的Viewholder
* 默认缓存个数是2个，可以通过 setViewCacheSize() 方法来改变缓存的容量大小。
* 如果 mCachedViews 的容量已满，则会根据 FIFO 的规则将旧 ViewHolder 抛弃，然后添加新的 ViewHolder，抛弃的会添加的第四级缓存 RecycledViewPool 中


## 第三级缓存 ViewCacheExtension
这是RecycleView 预留给开发人员的一个抽象类，在这个类中只有一个抽象方法
```java
public abstract static class ViewCacheExtension {
    public ViewCacheExtension() {
    }
    @Nullable
    public abstract View getViewForPositionAndType(@NonNull RecyclerView.Recycler var1, int var2, int var3);
}
```
开发人员可以通过继承 ViewCacheExtension，并复写抽象方法 getViewForPositionAndType 来实现自己的缓存机制。但是不建议自己去添加缓存逻辑，因为这个类的使用门槛较高，需要开发人员对 RV 的源码非常熟悉。

## 第四级缓存 RecycledViewPool
缓存屏幕外的 ViewHolder

当 mCachedViews 中的个数已满（默认为 2），则从 mCachedViews 中淘汰出来的 ViewHolder 会先缓存到 RecycledViewPool 中

ViewHolder 在被缓存到 RecycledViewPool 时，会将内部的数据清理，因此从 RecycledViewPool 中取出来的 ViewHolder 需要重新调用 onBindViewHolder 绑定数据。
这就同最早的 ListView 中的使用 ViewHolder 复用 convertView 的道理是一致的，

多个 RecycleView 之间可以共享一个 RecycledViewPool，这对于多 tab 界面的优化效果会很显著。需要注意的是

**RecycledViewPool 是根据 type 来获取 ViewHolder，每个 type 默认最大缓存 5 个**。

因此多个 RecyclerView 共享 RecycledViewPool 时，必须确保共享的 RecyclerView 使用的 Adapter 是同一个，或 view type 是不会冲突的。

---

搬运地址：    

Android 工程师进阶 34 讲 - 第16讲：为什么 RecyclerView 可以完美替代 Listview？

[RecyclerView性能优化及高级使用](https://blog.csdn.net/smileiam/article/details/88396546)    


[Recycler View 性能优化 （你必须知道的几点）](https://blog.csdn.net/weixin_37558974/article/details/108389821)    
