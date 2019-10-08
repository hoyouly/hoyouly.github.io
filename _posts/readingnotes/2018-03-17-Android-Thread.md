---
layout: post
title: AsyncTaks , HandlerThread , IntentService 小结
category: 读书笔记
tags: Android开发艺术探索  HandlerThread AsyncTaks IntentService
---

* content
{:toc}

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
5. 一个AsyncTask对象只能执行一次，即只能调用一次execute()方法，否则会报运行时异常
6. 在1.6 之前是串行，在3.0 之前是采用线程池并行，在3.0 之后，为避免并发错误，又采用串行执行任务。但是可以同executeOnExecutor方法并行执行任务

### AsyncTask 的工作原理
从execute方法分析
``` java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,Params... params) {
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
1. execute 方法调用的是executeOnExecutor()方法
2. 因为ececute方法是在主线程中调用的，所以onPreExecute()方法确实是在主线程中执行的，然后才开始执行线程池
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
                  r.run();//执行 Runnalbe任务
              } finally {
                //执行下一个AsyncTask任务
                scheduleNext();
              }
          }
      });
      if (mActive == null) {//没有活动中的AsyncTask任务
          scheduleNext();
      }
  }

  protected synchronized void scheduleNext() {
      if ((mActive = mTasks.poll()) != null) {
          THREAD_POOL_EXECUTOR.execute(mActive);
      }
}
```
从SerialExecutor 的实现可以分析AsyncTask的排队执行过程
1. 把AsyncTask的Params参数封装为FutureTask对象这个并发类
2. 把FutureTask对象交给SerialExecutor的exctute()方法执行
3. 如果这个时候没有活动中的AsyncTask任务，那么就调用 scledulNext()方法执行下一个AsyncTask任务，
4. 如果有，则会执行完这个任务之后，执行 scledulNext()

* 两个线程池  sDefaultExecutor 和THREAD_POOL_EXECUTOR
  1. THREAD_POOL_EXECUTOR 是用来执行任务，
  2. sDefaultExecutor 只是为了任务的排队。
* 一个Handler   InternalHandler
  InternalHandler用于将执行环境从线程池切换到主线程

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
    mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                postResultIfNotInvoked(get());
                ...
            }
        };
}
```
THREAD_POOL_EXECUTOR.execute(mActive) 会执行到 FutureTask 的run()->  callable.call()
这个 callable 就是创建 FutureTask的mWorke对象。于是就执行到了 WorkerRunnable的 call()中

在AsyncTask的构造函数中，我们可以看到在mWorker的call方法中，
1. 将mTaksInvoked 设置为true，表示当前任务已经被调用，
2. 然后执行doInbackgroud()方法，
3. 接着将其返回值传递给postResult()方法，

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
1. postResult() 其实就是通过sHandler发送一个MESSAGE_POST_RESULT消息
2. sHandler 是一个静态的Handler对象，这样就能保证sHandler是在主线程中的，因为静态成员会在加载类的时候进行初始化
3. MESSAGE_POST_RESULT这个消息类型在sHandler 中最后执行了result.mTask.finish(）方法，而result.mTask.其实就是一个AsyncTask对象，即最后执行到了AsyncTask的finish方法中，
4. 在finish方法中可以看到，如果取消了，就执行取消的方法，否则执行onPostExecute()方法，这样就又在主线程中了。

整个AsyncTask的过程也就分析完毕了

总结一下。调用链就是：
```
AsyncTask.execute()->AsyncTask.executeOnExecutor()->AsyncTask.onPreExecute()
->sDefaultExecutor.execute(FutureTask)->把FutureTask插入到mTasks 队列中
->AsyncTask.SerialExecutor.scheduleNext(FutureTask)
->FutureTask.run()->WorkerRunnable.call()->AsyncTask.doInBackground()
->AsyncTask.postResult()->通过Handler发送消息到主线程
->FutureTask.finish() ->AsyncTask.onPostExecute()
```

## HandlerThread
HandlerThread底层使用的是线程，是一种只用消息循环的线程，他的内部可以使用handler
是一种可以使用Handler的Thread，实现很简单，
``` java
public class HandlerThread extends Thread {
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
}
```
* 是个Thread不假，但是在run()方法中，竟然Looper.prepare()，还Looper.loop()，新建一个Looper对象，并且已经启动loop开始循环。这就有点意思了。
 <span style="border-bottom:1px solid red;">所以使用上HandlerThread和普通的Thread不一样,无法执行后台常见的操作，只能处理新消息。因为Looper.loop()是一个死循环</span>
* 由于HandlerThread的run方法是一个无线循环，因此当明确不需要使用HandlerThread的时候，通过quit或者quitSafety方法来终止线程的执行。
* 普通Thread 主要在run()中执行耗时操作。HandleThread 内部建立消息机制。需要外接通过handler消息通知HandlerThread执行的具体任务。

## IntentService
* 一种特殊的Service，继承Service并且是一个抽象类 **public abstract class IntentService extends Service**
* 可用于执行后台耗时任务，当任务执行后会自动停止。
* 由于是一个service，所以优先级比单纯的线程要高，适合执行一些优先级高的后台任务，

使用方法
1. 创建一个IntentService的子类。复写其中的onHandleIntent()方法，在这里进行耗时的操作
2. startService(intent)
这样就行了，简单。可是为啥IntentService就能直接处理耗时操作呢，普通的Service还需要在onStartCommand()中再开线程才行。
<font color="#ff000" >IntentService可以看做是 Service和HanderThread的结合体。</font>

还是看IntentService的源码吧

```java
public abstract class IntentService extends Service {

	private final class ServiceHandler extends Handler {
         public ServiceHandler(Looper looper) {
             super(looper);
         }

         @Override
         public void handleMessage(Message msg) {
             onHandleIntent((Intent)msg.obj);
             stopSelf(msg.arg1);
         }
     }
	@Override
  public void onCreate() {
     super.onCreate();
     HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
     thread.start();
     mServiceLooper = thread.getLooper();
     mServiceHandler = new ServiceHandler(mServiceLooper);
  }

	@Override
	public void onStart(Intent intent, int startId) {
		 Message msg = mServiceHandler.obtainMessage();
		 msg.arg1 = startId;
		 msg.obj = intent;
		 mServiceHandler.sendMessage(msg);
	}

	 @Override
	 public int onStartCommand(Intent intent, int flags, int startId) {
			 onStart(intent, startId);
			 return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
	 }
	 protected abstract void onHandleIntent(Intent intent);

   @Nullable
   public IBinder onBind(Intent intent) {
       return null;
   }
}
```
1. 在onCreate()的时候，就创建一个HandlerThread对象，然后启动，拿到这个HandlerThread的Looper对象，这才是关键。<span style="border-bottom:1px solid red;">创建一个继承Handler的类 ServiceHandler，把得到的子线程的Looper对象传进去。</span>
2. 是Service，startService()后，肯定就会执行到了onStartCommand()方法，里面执行了onStart()的。这个时候关键来了，<span style="border-bottom:1px solid red;">把intent封装成Message然后通过ServiceHandler 送出去。</span> 之前我们讲过，<font color="#ff000" >创建Handler的时候需要一个Looper对象，这个Looper对象属于哪个线程，那么Handler就会把消息发送给那个线程。</font> ServiceHandler 创建的时候传递的是子线程的Looper对象，那么会把消息发送到这个子线程中去。所以handleMessage()就肯定是在HanderThread这个线程中执行的。
3. handleMessage()中调用了 onHandleIntent(),将Intent对象传递给onHandleIntent()处理，这个Intent的内容和外界的startService（Intent）的内容是完全一致，onHandleIntent()是需要我们实现的方法，可以进行执行耗时操作的方法。因为它就是执行在子线程的。并
4. handleMessage() 最后会 stopSelf(msg.arg1)，把自己停掉。这么智能啊。
5. <span style="border-bottom:1px solid red;">onBind()中返回null，所以不适合使用bindService()的方式使用IntentService()</span>

所以IntentService有以下特点：
* 继承Service，所以优先级比较高
* 内部创建了一个HandlerThread 和ServiceHandler，可以做一些耗时操作
* 当任务执行完成后，IntentService自动停止，不需要我们手动结束
* 如果多次启动IntentService，使用串行方式，依次执行onHandlerIntent(),执行完自动结束。那么每个耗时的操作都会消息的形式发生到消息队列中。
