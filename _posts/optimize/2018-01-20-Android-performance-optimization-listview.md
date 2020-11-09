---
layout: post
title: Android 性能优化 -- ListView 优化
category: 性能优化
tags: Android开发艺术探索 性能优化
description: Android 性能优化
---

* 复用 contentView ,减少重新分配缓存造成的内存频繁分配回收
*  ViewHolder的使用，主要是因为 findViewById() 消耗时间太大，使用 setTag() , getTag() 得到view
*  图片加载优化 监听 Listview 的滑动状态，滑动的时候（SCROLL_STATE_TOUCH_SCROLL）和惯性滑动的时候（SCROLL_STATE_FLING）不加载，停止滑动的时候再加载。listView.setOnScrollListener();
* onClickListener的处理，通常是在 getView 方法时候处理，但是这样的话，每次调用 getView() 都会设置 onClickListener() 效率低，可以在 ViewHolder 中设置一个 Position ，然后实现 OnClickListener   

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
*  减小 item view 的布局层级关系，布局层级过深，直接导致 View 的测量和绘制时间加大
*  adapter中的 getView() 方法尽量少使用逻辑,可以想办法把逻辑抽离到其他地方
*  getView()中尽量少做耗时操作，避免创建大量对象
*  善用自定义 View ，自定义 View 可以很好的控制层级，并且对绘制过程可以很好的控制。
*  使用 RecycleView 代替， ListView 每次更新数据都要 notifyDataSetChanged() ，有些太暴力了。 RecycleView 在性能和可定制性上都有很大的改善，推荐使用。
*  尝试开启硬件加速来使 Listview 的滑动更加流畅

---
搬运地址：    

Android 开发艺术探索      

Android 群英传     

[Android性能优化系列之 Bitmap 图片优化](https://blog.csdn.net/u012124438/article/details/66087785)   

[Android中常见的内存泄漏及解决方案](https://blog.csdn.net/u014005316/article/details/63258107)   

[Android 内存泄漏分析心得](https://zhuanlan.zhihu.com/p/25213586)     

[Android 布局优化之 include 与merge](https://blog.csdn.net/a740169405/article/details/50473909)

[Android优化】最强 ListView 优化方案](http://blog.csdn.net/gs12software/article/details/51173392)

[Android APP性能优化(最新总结)](http://blog.csdn.net/csdn_aiyang/article/details/74989318)

[Android开发性能优化总结(一)](http://blog.csdn.net/gs12software/article/details/51173392)
