---
layout: post
title: Android的线程和线程池
category: 读书笔记
tags: Android开发艺术探索，线程池，线程
description: Android的线程和线程池
---

线程是操作系统调度的最小单元，又是一种受限制的系统资源，即线程不可能无限次地生产，并且线程的创建和消耗都有相应的开销，当系统中存在大量的线程是，系统会通过时间片轮转的方式调度每个线程，分为两种
1. 主线程： 处理和界面相关的事情
2. 子线程：用于好事的操作

如果线程中频繁创建和消耗线程，这种做法不是很高效，可以采用线程池，一个线程池会缓存一定数量的线程，通过线程池可以避免频繁创建和消耗线程所带来的系统开销。主要通过Executor来派生特定类型的线程池。

扮演线程的角色有很多，包括AsyncTask和IntentService,同时HandlerThread也是一种特殊的线程
## AsyncTaks
AsyncTask 底层使用的是线程池，分装了线程池和Handler，主要用于开发者可以使用在子线程中更新UI是一种轻量级的异步任务类。并不适合进行特别耗时的后台任务，
### AsyncTaks 的一些限制
1. AsyncTask的类必须在主线程中加载。
2. AsyncTask的对象必须在主线程中创建
3. execute方法必须在UI线程中调用
4. 不要在程序中直接调用onPreExecute().onPosteExecute(),doInBackground和onProgressUpdate方法
5. 一个AsyncTask对象只能执行一次，即只能调用一次execute()方法，否则会包运行时一次
6. 在1.6 之前是串行，在3.0 之前是采用线程池并行，在3.0 之后，为避免并发错误，有采用串行执行任务。但是可以同executeOnExecutor方法并行执行任务
### AsyncTask 的工作原理
从execute方法分析
``` java
  public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```

又以上源码可知
1. execute 方法调用的是executeOnExecutor 方法
2. 因为ececute方法是在主线程中调用的，所以onPreExecute方法确实是在主线程中执行的，然后才开始执行线程池
3. sDefaultExecutor看源码可知是一个串行的线程池，一个进程中的所有AsyncTack全部都放入到这个线程池中排队执行。

``` java
  public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
从SerialExecutor 的实现可以分析AsyncTask的排队执行过程
1. 把AsyncTask的Params参数封装为FutureTask对象，这个并发类，
2. 把FutureTask对象交给SerialExecutor的exctute方法执行
3. 如果这个时候没有活动中的AsyncTask人物，那么就执行scledulNext方法执行下一个AsyncTask人物，同事如果AsyncTask执行完，会执行其他的任务

AsyncTask有两个线程池，（sDefaultExecutor 和THREAD_POOL_EXECUTOR）和一个Handler（InternalHandler） 其中THREAD_POOL_EXECUTOR 是用来执行任务，而sDefaultExecutor 只是为了任务的排队。InternalHandler用于将执行环境从线程池切换到主线程

```java
  public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                return postResult(doInBackground(mParams));
            }
        };
        }
```
在AsyncTask的构造函数中，我们可以看到在mWorker的call方法中，首先讲啊mTaksInvoked 设置为true，表示当前任务已经被调用，然后执行doInbackgroud方法，接着将其返回值传递给postResult方法，

```java
 private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

private static class InternalHandler extends Handler {
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

	 private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```
从上面三段代码可以看出
1. postResult 其实就是通过sHandler发送一个MESSAGE_POST_RESULT消息
2. sHandler 是一个静态的Handler对象，这样就能保证sHandler是在主线程中的，因为静态成员会在加载类的时候进行初始化
3. MESSAGE_POST_RESULT这个消息类型在sHandler 中最后执行了result.mTask.finish(）方法，而result.mTask.其实就是一个AsyncTask对象，即最后执行到了AsyncTask的finish方法中，
4. 在finish方法中可以看到，如果取消了，就执行取消的方法，否则执行onPostExecute方法，这样就有在主线程中了。整个AsyncTask的过程也就分析完毕了

## HandlerThread
HandlerThread底层使用的是线程，是一种拒用消息循环的线程，他的内部可以使用handler
是一种可以使用Handler的Thread，实现很简单，
``` java
 @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```
在run方法中创建Looper.perpare()来创建消息队列，通过loop()开启消息循环，
由于HandlerThread的run方法是一个无线循环，因此当明确不需要使用HandlerThread的时候，通过quit或者quitSafety方法来终止线程的执行。
## IntentService
一种特殊的Service，继承Service并且是一个抽象类 public abstract class IntentService extends Service ，可用于执行后台耗时任务，当任务执行后会自动停止，由于是一个service，所以优先级比单纯的线程要高，适合执行一些优先级高的后台任务，内部封装了HandlerThread和Handler，可在onCreate方法中看处理
```java
 public void onCreate() {
      super.onCreate();
      HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
      thread.start();
      mServiceLooper = thread.getLooper();
      mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```
当第一启动的适合，onCreate被调用，创建一个HandlerThread，然后使用它的looper来构造一个Handler对象mServiceHandler,这样mServiceHandler发送消息最终都会在handlerThread中执行，他的onStartCommand每次调用，都会处理后台任务中的Intent，
```java
 public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

  public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
```
由onStart方法可知，IntentService仅仅通过mServiceHandler发送一个消息，这个消息会在handlerThread中处理，mServiceHandler收到消息后，会将Intent对象传递给onHandleIntent 方法处理，这个Intent的内容和外界的startService（Intent）的内容是完全一致，


## Android中的线程池
线程池的好处：
1. 重用线程池中的线程，避免因为线程的创建和消耗带来的性能开销
2. 能有效的控制线程池的最大并发数，避免大量的线程之间因为互相抢占系统资源而导致的阻塞现象
3. 能够对线程进行简单的管理，并提供定时执行依据指定间隔循环执行等功能

### ThreadPoolExecutor
线程池的实现类。Android中的线程池都是直接或者间接通过配置ThreadPoolExecutor 来实现的，从线程池的功能上来说，Android的线程池主要分为4类，这4类线程池可以通过Executors所提供的工厂方法来得到。

```java
一个常用的创建线程池的构造方法
public ThreadPoolExecutor(
int corePoolSize,
int maximumPoolSize,
long keepAliveTime,
TimeUnit unit,
BlockingQueue<Runnable> workQueue,
ThreadFactory threadFactory)

corePoolSize： 核心线程数，默认会一直在线程池中存活，
如果allowCoreThreadTimeOut 设置为true，那么闲置的核心线程在等待新任务的时候有超时策略，这个由keepAliveTime 决定，
maximumPoolSize： 线程池所能容纳的最大线程数，当活动线程达到这个数字后，后续任务会被阻塞
keepAliveTime： 非核心线程超时时长，当allowCoreThreadTimeOut 设置为true，同样作用于核心线程
unit：keepAliveTime的时间单位，是一个枚举类型，常用的有TimeUnit.MILLISECONDS(毫秒),TimeUnit.SECONDS（秒）,TimeUnit.MINUTES（分钟）
workQueue ：线程池中的任务队列，通过线程池中的execute方法提交的Runnable对象，会存储在这这个参数中
threadFactory： 线程工厂，为线程池提供创建新线程的功能，是一个借口，只有一个方法，Thread newThread(Runnable r);
```
ThreadPoolExecutor 的参数配置在AsyncTask中有明显的体现，
```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);
        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

```
AsyncTask中对线程池的配置
1. 核心线程数等于CPU核心数+1；
2. 线程池的最大线程数为CPU核心线程数的2倍+1；
3. 核心线程无超时机制，非核心线程池在闲置时超时时间为1秒
4. 任务队列的容量为128

### 线程池的分类
Android中最常见的四类不同功能特性的线程池。他们都是直接或者间接通过配置ThreadPoolExecute来实现自己的功能特性的。分别是FixedThreadPool，CachedThreadPool，ScheduledThreadPool以及SingleThreadExecute
#### FixedThreadPool 线程数量固定的线程池
1. 通过Executors的newFixedThreadPool(int nThreads) 创建
2. 线程数量固定的线程池，当线程处于空闲状态，不会被回收，除非线程池被关闭
3. 只有核心线程并且不会被回收，这意味着它能够更加快速的相应外界的请求

```java
 public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

```
#### CachedTheadPool 线程数量不定的线程池

1. 只有非核心线程，并且最大值为Integer.MAX_VALUE，由于Integer.MAX_VALUE很大，实际上可以理解为最大线程可以任意大
2. 当线程池中的线程都处于活动状态，线程池会创建新的线程处理新任务
3. 空闲线程都有超时机制，超过60秒，就会被回收
4. CachedTheadPool的任务队列是一个空集合，
5. 适合执行搭理耗时较少的任务
6. 当整个线程处于空闲状态的时候，线程池中的线程都会因为超时被停止，这个时候CachedTheadPool时间是没有任何线程，几乎不占任何系统资源

```java
  public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

```
#### ScheduleThreadPool 核心线程固定，非核心线程没有限制
1. 当非核心线程闲置时会理解被回收
2. 主要用于执行定时任务和具有固定周期的重复任务

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

 public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }
===================
 public ScheduledThreadPoolExecutor(int corePoolSize,ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,new DelayedWorkQueue(), threadFactory);
    }
```

#### SingleThreadPool 只有一个核心线程
1. 所有任务都在一个线程中按照顺序执行。
2. 意义在于统一所有外界任务到一个线程中，使得这些任务之间不需要处理线程同步的问题

```java
   public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
