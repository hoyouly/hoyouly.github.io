---
layout: post
title: Android 线程池
category: 读书笔记
tags: Android开发艺术探索 线程池
---

* content
{:toc}

## 线程池的好处：
1. 重用线程池中的线程，避免因为线程的创建和消耗带来的性能开销
2. 能有效的控制线程池的最大并发数，避免大量的线程之间因为互相抢占系统资源而导致的阻塞现象
3. 能够对线程进行简单的管理，并提供定时执行依据指定间隔循环执行等功能

## ThreadPoolExecutor
ExecutorService  是最初的线程池接口，ThreadPoolExecutor是线程池的实现类。ThreadPoolExecutor 继承AbstractExecutorService，而AbstractExecutorService实现了ExecutorService接口
```java
public class ThreadPoolExecutor extends AbstractExecutorService {}

public abstract class AbstractExecutorService implements ExecutorService {}
```
ThreadPoolExecutor 通过构造函数来配置线程池。下面是一个比较常用的创建ThreadPoolExecutor的构造方法

```java
public ThreadPoolExecutor(
int corePoolSize,
int maximumPoolSize,
long keepAliveTime,
TimeUnit unit,
BlockingQueue<Runnable> workQueue,
ThreadFactory threadFactory)
```
* corePoolSize： 核心线程数，默认会一直在线程池中存活，
  * 如果allowCoreThreadTimeOut 设置为true，那么闲置的核心线程在等待新任务的时候有超时策略，这个由keepAliveTime 决定，即keepAliveTime时间内核心线程没有回应则线程会被终止，
  * 如果allowCoreThreadTimeOut 为false，核心线程没有超时时间。
* maximumPoolSize： 线程池所能容纳的最大线程数，当活动线程达到这个数字后，后续任务会被阻塞。最大线程数=核心线程数+非核心线程，
  * 非核心线程只有核心线程不够用并且线程池有空余时才会创建，执行完任务后非核心线程会被销毁
* keepAliveTime： 非核心线程超时时长，当allowCoreThreadTimeOut 设置为true，同样作用于核心线程
* unit：keepAliveTime的时间单位，是一个枚举类型，常用的有TimeUnit.MILLISECONDS(毫秒),TimeUnit.SECONDS（秒）,TimeUnit.MINUTES（分钟）
* workQueue ：线程池中的任务队列，通过线程池中的execute方法提交的Runnable对象，会存储在这这个参数中
* threadFactory： 线程工厂，为线程池提供创建新线程的功能，是一个借口，只有一个方法，Thread newThread(Runnable r);

### RejectExecutionHandler handler
这是 ThreadPoolExecutor 不常用的参数，当线程池无法执行新任务，这可能由于任务队列已满或者无法成功执行任务，ThreadPoolExecutor会调用handler 的rejectEcecution()方法通知调用者，默认情况下rejectEcecution()会抛出一个RejectExecutionExeception

## 线程池遵循的规则
* 如果线程池中的线程的数量未达到核心线程的数量，那么直接启动一个核心线程来执行任务
* 如果线程池中线程的数量已经达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行
* 如果步骤 2 中无法将任务插入到任务队列中，往往是任务队列满了，这时候如果线程数量未达到线程池最大值，那么会立刻启动一个非核心线程执行任务。
* 如果步骤3中已经达到线程池中的最大值，那么就拒绝执行此任务，ThreadPoolExecutor会调用 RejectExecutionHandler 的rejectExecution 方法来通知调用者。

## ThreadPoolExecutor 的参数配置在AsyncTask中体现，
```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};

private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);

public static final Executor THREAD_POOL_EXECUTOR;

static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}

```
AsyncTask中对线程池的配置
1. 核心线程数 至少有2个线程和最多4个线程(SDK 26 )。 `int CORE_POOL_SIZE =Math.max(2, Math.min(CPU_COUNT - 1, 4))`
2. 线程池的最大线程数为CPU核心线程数的2倍+1。 `int CORE_POOL_SIZE =CPU_COUNT * 2 + 1`
3. 核心线程无超时机制。 `threadPoolExecutor.allowCoreThreadTimeOut(true);`
4. 非核心线程池在闲置时超时时间为30秒 。 `  int KEEP_ALIVE_SECONDS = 30`
4. 任务队列的容量为128 。`  BlockingQueue<Runnable> sPoolWorkQueue = new LinkedBlockingQueue<Runnable>(128)`

## 线程池的分类
Android中最常见的四类不同功能特性的线程池。他们都是直接或者间接通过配置ThreadPoolExecute来实现自己的功能特性的。从线程池的功能上来说，Android的线程池主要分为4类，这4类线程池可以通过Executors所提供的工厂方法来得到。分别是FixedThreadPool，CachedThreadPool，ScheduledThreadPool以及SingleThreadExecutor
### FixedThreadPool 线程数量固定的线程池
1. 通过 Executors 的 newFixedThreadPool(int nThreads) 创建
2. 线程数量固定的线程池，当线程处于空闲状态，不会被回收，除非线程池被关闭
3. 当所有线程都处于活动状态时，新任务会处于等待状态，知道有空闲出来，
4. 只有核心线程并且不会被回收，这意味着它能够更加快速的相应外界的请求
5. 任务队列没有大小限制。

```java
 public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

```
### CachedTheadPool 线程数量不定的线程池

1. 通过 Executors 的 newCachedThreadPool() 来创建，线程数量不定的线程池
1. 只有非核心线程，并且最大值为Integer.MAX_VALUE，由于Integer.MAX_VALUE很大，实际上可以理解为最大线程可以任意大
2. 当线程池中的线程都处于活动状态，线程池会创建新的线程处理新任务
3. 空闲线程都有超时机制，超过60秒，就会被回收
4. CachedTheadPool的任务队列是一个空集合，这将导致任何任务都会立刻执行
5. 适合执行搭理耗时较少的任务
6. 当整个线程处于空闲状态的时候，线程池中的线程都会因为超时被停止，这个时候CachedTheadPool时间是没有任何线程，几乎不占任何系统资源

```java
public static ExecutorService newCachedThreadPool() {
      return new ThreadPoolExecutor(0, Integer.MAX_VALUE,//
                                    60L, TimeUnit.SECONDS,//
                                    new SynchronousQueue<Runnable>());
}

```
### ScheduleThreadPool 核心线程固定，非核心线程没有限制
1. 通过Executors 的newScheduleThreadPool() 来创建，核心线程固定，非核心线程没有限制。
2. 当非核心线程闲置时会立即被回收
3. 主要用于执行定时任务和具有固定周期的重复任务

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
============================================================================
public ScheduledThreadPoolExecutor(int corePoolSize,ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE,//
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,//
          new DelayedWorkQueue(), threadFactory);
}
```

### SingleThreadExecutor 只有一个核心线程
1. 通过 Executors 的 newSingleThreadExecutor()来创建，只有一个核心线程
1. 所有任务都在一个线程中按照顺序执行。
2. 意义在于统一所有外界任务到一个线程中，使得这些任务之间不需要处理线程同步的问题

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService(
                                new ThreadPoolExecutor(1, 1,0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
