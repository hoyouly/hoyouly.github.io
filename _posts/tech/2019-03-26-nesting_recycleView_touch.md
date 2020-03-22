---
layout: post
title: 水平方向的 RecycleView 嵌套竖直方向的 RecycleView 滑动冲突解决办法
category: 技术
tags: RecycleView
---
* content
{:toc}

项目中用到了RecycleView嵌套，水平方向嵌套竖直方向的，结果发现一个问题，就是水平方向滑动后，直接竖直滑动，往往第一下竖直方向没响应，要等到第二下甚至第三下竖直方向的RecycleView才能响应滑动事件，可是我们交互要求如果正在水平滑动，突然竖直滑动，竖直方向的RecycleView是要立即响应的,这就有点意思了，为啥会出现这种情况呢，查看源码才知道，在RecycleView的onInterceptTouchEvent()中做了处理

```java
RecyclerView.java

public boolean onInterceptTouchEvent(MotionEvent e) {
   ...
   switch (action) {
       case MotionEvent.ACTION_DOWN:

       if (mScrollState == SCROLL_STATE_SETTLING) {
           getParent().requestDisallowInterceptTouchEvent(true);
           setScrollState(SCROLL_STATE_DRAGGING);
       }
       ...
    }
    return mScrollState == SCROLL_STATE_DRAGGING;
 }

```
当处于滑行状态，即快速滑动后到滑动静止之间的时候，会在DOWN事件的时候拦截掉这个事件

外部的RecycleView拦截后，那么内部的RecycleView就收不到Touch事件，也就没办法响应滑动事件了。

而且我们知道，一旦DOWN事件中被拦截，那么剩下的MOVE和UP等事件，也都会直接传递给该View。

也就是说一旦外部的RecycleView 在拦截了DOWN事件，那么就剩下一系列事件就别想传递到内部的RecycleView了，内部的RecycleView接收不到事件，就没办法滑动了。

所以第一步就是不能让外部RecycleView给拦截了，起码不能拦截竖直方向的。
然后我又做了一些处理，当滑动的时候，外部RecycleView只拦截水平方向的滑动事件，具体代码如下。

```java
public class OutRecyclerView extends RecyclerView {
    private static final String TAG = "OutRecyclerView";
    private int mInitialTouchX;
    private int mInitialTouchY;

    private static final int INVALID_POINTER = -1;
    private int mScrollPointerId = INVALID_POINTER;

    private int mTouchSlop;
    private ViewConfiguration mViewConfiguration;

    public OutRecyclerView(Context context) {
        this(context, null);
    }

    public OutRecyclerView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public OutRecyclerView(Context context, @Nullable AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        mViewConfiguration = ViewConfiguration.get(context);
        mTouchSlop = mViewConfiguration.getScaledTouchSlop();
    }

    @Override
    public void setScrollingTouchSlop(int slopConstant) {
        super.setScrollingTouchSlop(slopConstant);
        switch (slopConstant) {
            case TOUCH_SLOP_DEFAULT:
                mTouchSlop = mViewConfiguration.getScaledTouchSlop();
                break;
            case TOUCH_SLOP_PAGING:
                mTouchSlop = mViewConfiguration.getScaledPagingTouchSlop();
                break;
            default:
                break;
        }
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent e) {
        final int action = e.getAction();
        boolean intercepted = false;
        switch (action) {
            case MotionEvent.ACTION_MOVE: {
                final int index = MotionEventCompat.findPointerIndex(e, mScrollPointerId);
                if (index < 0) {
                    return false;
                }
                final int x = (int) (MotionEventCompat.getX(e, index) + 0.5f);
                final int y = (int) (MotionEventCompat.getY(e, index) + 0.5f);
                final int dx = x - mInitialTouchX;
                final int dy = y - mInitialTouchY;
                final boolean canScrollHorizontally = getLayoutManager().canScrollHorizontally();
                boolean startScroll = false;
                //可以水平方向滑动并且滑动距离大于mToushSLop,并且水平方向滑动距离大于竖直方向的。
                if (canScrollHorizontally && Math.abs(dx) > mTouchSlop && (Math.abs(dx) >= Math.abs(dy))) {
                    startScroll = true;//水平方向
                }
                intercepted = super.onInterceptTouchEvent(e);
                //如果竖直方向，那么startScroll为false，
                //那么不管super.onInterceptTouchEvent(e)true或者false，结果都是false，那么就不会拦截。
                return startScroll && intercepted;
            }
            default:
                int state = getScrollState();
                mScrollPointerId = e.getPointerId(0);
                mInitialTouchX = (int) (e.getX() + 0.5f);
                mInitialTouchY = (int) (e.getY() + 0.5f);
                intercepted = super.onInterceptTouchEvent(e);
                if (state == SCROLL_STATE_SETTLING || state == SCROLL_STATE_DRAGGING) {
                    intercepted = false;
                }
                return intercepted;
        }
    }

```
这样就保证在DOWN事件时候，外部RecycleView不会拦截掉，并且只拦截水平方向的事件，
这样就可以了吗，其实还不行，因为这样会有另外一个问题，尽管内部的RecycleView接收到了事件，可是当内部的RecycleView竖直滑动中，如果这个时候再水平滑动，那么外部RecycleView又接收不到事件了，因为事件已经传递给了内部RecycleView，这个时候，就需要进行判断，如果是水平滑动，那么应该把事件交由外部的RecycleView，然后就想起来requestDisallowInterceptTouchEvent()这个，在dispatchTouchEvent()中判断滑动速度，如果是水平方向大于竖直方向，那么就交给外部的RecycleView 处理。即

```java
public class InRecyclerView extends RecyclerView {
    private static final String TAG = "HomePageRecyclerView";
    //速度追踪
    private VelocityTracker mVelocityTracker;

    private int mInitialTouchX;
    private int mInitialTouchY;

    //是否正在快速滑动
    public boolean mIsFastScrolling = false;
    //判定为快速滑动的速度阀值
    private static final int SCROLL_THRESHOLD_SPEED = 6000;

    public InRecyclerView(Context context) {
        this(context, null);
    }

    public InRecyclerView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public InRecyclerView(Context context, @Nullable AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        mVelocityTracker = VelocityTracker.obtain();
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        int action = ev.getAction();
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                mInitialTouchX = (int) (ev.getX() + 0.5f);
                mInitialTouchY = (int) (ev.getY() + 0.5f);
                break;
            case MotionEvent.ACTION_MOVE:
                //向velocityTracker对象添加action
                mVelocityTracker.addMovement(ev);
                mVelocityTracker.computeCurrentVelocity(1000);
                float xVelocity = mVelocityTracker.getXVelocity();
                float yVelocity = mVelocityTracker.getYVelocity();
                final int x = (int) (ev.getX() + 0.5f);
                final int y = (int) (ev.getY() + 0.5f);
                final int dx = x - mInitialTouchX;
                final int dy = y - mInitialTouchY;
                //SCROLL_STATE_SETTLING：滑动后自然沉降的状态  SCROLL_STATE_DRAGGING 滑动状态
                if ((getScrollState() == SCROLL_STATE_SETTLING || getScrollState() == SCROLL_STATE_DRAGGING)
                        && (Math.abs(xVelocity) > Math.abs(yVelocity)) && (Math.abs(dx) > Math.abs(dy))) {
                    //水平方向滑动的速度和距离都大于竖直方向，告诉父View，这种情况下，我不需要，事件交给父View
                    getParent().requestDisallowInterceptTouchEvent(false);
                    return false;
                }
                if (Math.abs(yVelocity) > SCROLL_THRESHOLD_SPEED) {
                    mIsFastScrolling = true;
                }
                break;
            default:
        }
        return super.dispatchTouchEvent(ev);
    }    
}
```

---
搬运地址：    

