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

如果线程中频繁创建和消耗线程，这种做法不是很高效，可以采用线程池，一个线程池会缓存一定数量的线程，通过线程池可以避免频繁创建和消耗线程所带来的系统开销。主要通过 Executor 来派生特定类型的线程池。

扮演线程的角色有很多，包括 AsyncTask 和 IntentService ,同时 HandlerThread 也是一种特殊的线程
## AsyncTaks
AsyncTask 底层使用的是线程池，分装了线程池和 Handler ，主要用于开发者可以使用在子线程中更新 UI 是一种轻量级的异步任务类。并不适合进行特别耗时的后台任务，
### AsyncTaks 的一些限制
1. AsyncTask的类必须在主线程中加载。
2. AsyncTask的对象必须在主线程中创建
3. execute方法必须在 UI 线程中调用
4. 不要在程序中直接调用onPreExecute().onPosteExecute(), doInBackground 和 onProgressUpdate 方法
5. 一个 AsyncTask 对象只能执行一次，即只能调用一次 execute() 方法，否则会报运行时异常
6. 在1.6 之前是串行，在3.0 之前是采用线程池并行，在3.0 之后，为避免并发错误，又采用串行执行任务。但是可以同 executeOnExecutor 方法并行执行任务

### AsyncTask 的工作原理
从 execute 方法分析
``` java
public final AsyncTask<Params, Progress , Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
public final AsyncTask<Params, Progress , Result> executeOnExecutor(Executor exec,Params... params) {
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
1. execute 方法调用的是 executeOnExecutor() 方法
2. 因为 ececute 方法是在主线程中调用的，所以 onPreExecute() 方法确实是在主线程中执行的，然后才开始执行线程池
3. sDefaultExecutor看源码可知是一个串行的线程池，一个进程中的所有 AsyncTack 全部都放入到这个线程池中排队执行。

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
                  r.run();//执行 Runnalbe 任务
              } finally {
                //执行下一个 AsyncTask 任务
                scheduleNext();
              }
          }
      });
      if (mActive == null) {//没有活动中的 AsyncTask 任务
          scheduleNext();
      }
  }

  protected synchronized void scheduleNext() {
      if ((mActive = mTasks.poll()) != null) {
          THREAD_POOL_EXECUTOR.execute(mActive);
      }
}
```
从 SerialExecutor 的实现可以分析 AsyncTask 的排队执行过程
1. 把 AsyncTask 的 Params 参数封装为 FutureTask 对象这个并发类
2. 把 FutureTask 对象交给 SerialExecutor 的 exctute() 方法执行
3. 如果这个时候没有活动中的 AsyncTask 任务，那么就调用 scledulNext() 方法执行下一个 AsyncTask 任务，
4. 如果有，则会执行完这个任务之后，执行 scledulNext()

* 两个线程池 sDefaultExecutor 和THREAD_POOL_EXECUTOR
  1. THREAD_POOL_EXECUTOR 是用来执行任务，
  2. sDefaultExecutor 只是为了任务的排队。
* 一个Handler   InternalHandler
InternalHandler 用于将执行环境从线程池切换到主线程

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
这个 callable 就是创建 FutureTask 的 mWorke 对象。于是就执行到了 WorkerRunnable 的 call() 中

在 AsyncTask 的构造函数中，我们可以看到在 mWorker 的 call 方法中，
1. 将 mTaksInvoked 设置为 true ，表示当前任务已经被调用，
2. 然后执行 doInbackgroud() 方法，
3. 接着将其返回值传递给 postResult() 方法，

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
1. postResult() 其实就是通过 sHandler 发送一个 MESSAGE_POST_RESULT 消息
2. sHandler 是一个静态的 Handler 对象，这样就能保证 sHandler 是在主线程中的，因为静态成员会在加载类的时候进行初始化
3. MESSAGE_POST_RESULT这个消息类型在 sHandler 中最后执行了result.mTask.finish(）方法，而result.mTask.其实就是一个 AsyncTask 对象，即最后执行到了 AsyncTask 的 finish 方法中，
4. 在 finish 方法中可以看到，如果取消了，就执行取消的方法，否则执行 onPostExecute() 方法，这样就又在主线程中了。

整个 AsyncTask 的过程也就分析完毕了

总结一下。调用链就是：
```
AsyncTask.execute()->AsyncTask.executeOnExecutor()->AsyncTask.onPreExecute()
->sDefaultExecutor.execute(FutureTask)->把 FutureTask 插入到 mTasks 队列中
->AsyncTask.SerialExecutor.scheduleNext(FutureTask)
->FutureTask.run()->WorkerRunnable.call()->AsyncTask.doInBackground()
->AsyncTask.postResult()->通过 Handler 发送消息到主线程
->FutureTask.finish() ->AsyncTask.onPostExecute()
```

## HandlerThread
HandlerThread 底层使用的是线程，是一种只用消息循环的线程，他的内部可以使用handler
是一种可以使用 Handler 的 Thread ，实现很简单，
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
* 是个 Thread 不假，但是在 run() 方法中，竟然Looper.prepare()，还Looper.loop()，新建一个 Looper 对象，并且已经启动 loop 开始循环。这就有点意思了。
 <span style="border-bottom:1px solid red;">所以使用上 HandlerThread 和普通的 Thread 不一样,无法执行后台常见的操作，只能处理新消息。因为Looper.loop()是一个死循环</span>
* 由于 HandlerThread 的 run 方法是一个无线循环，因此当明确不需要使用 HandlerThread 的时候，通过 quit 或者 quitSafety 方法来终止线程的执行。
* 普通 Thread 主要在 run() 中执行耗时操作。 HandleThread 内部建立消息机制。需要外接通过 handler 消息通知 HandlerThread 执行的具体任务。

## IntentService
* 一种特殊的 Service ，继承 Service 并且是一个抽象类 **public abstract class IntentService extends Service**
* 可用于执行后台耗时任务，当任务执行后会自动停止。
* 由于是一个 service ，所以优先级比单纯的线程要高，适合执行一些优先级高的后台任务，

使用方法
1. 创建一个 IntentService 的子类。复写其中的 onHandleIntent() 方法，在这里进行耗时的操作
2. startService(intent)
这样就行了，简单。可是为啥 IntentService 就能直接处理耗时操作呢，普通的 Service 还需要在 onStartCommand() 中再开线程才行。
<font color="#ff000" >IntentService可以看做是 Service 和 HanderThread 的结合体。</font>

还是看 IntentService 的源码吧

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
   public int onStartCommand(Intent intent, int flags , int startId) {
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
1. 在 onCreate() 的时候，就创建一个 HandlerThread 对象，然后启动，拿到这个 HandlerThread 的 Looper 对象，这才是关键。<span style="border-bottom:1px solid red;">创建一个继承 Handler 的类 ServiceHandler ，把得到的子线程的 Looper 对象传进去。</span>
2. 是 Service ， startService() 后，肯定就会执行到了 onStartCommand() 方法，里面执行了 onStart() 的。这个时候关键来了，<span style="border-bottom:1px solid red;">把 intent 封装成 Message 然后通过 ServiceHandler 送出去。</span> 之前我们讲过，<font color="#ff000" >创建 Handler 的时候需要一个 Looper 对象，这个 Looper 对象属于哪个线程，那么 Handler 就会把消息发送给那个线程。</font> ServiceHandler 创建的时候传递的是子线程的 Looper 对象，那么会把消息发送到这个子线程中去。所以 handleMessage() 就肯定是在 HanderThread 这个线程中执行的。
3. handleMessage()中调用了 onHandleIntent() ,将 Intent 对象传递给 onHandleIntent() 处理，这个 Intent 的内容和外界的startService（Intent）的内容是完全一致， onHandleIntent() 是需要我们实现的方法，可以进行执行耗时操作的方法。因为它就是执行在子线程的。并
4. handleMessage() 最后会 stopSelf(msg.arg1)，把自己停掉。这么智能啊。
5. <span style="border-bottom:1px solid red;">onBind()中返回 null ，所以不适合使用 bindService() 的方式使用IntentService()</span>

所以 IntentService 有以下特点：
* 继承 Service ，所以优先级比较高
* 内部创建了一个 HandlerThread 和 ServiceHandler ，可以做一些耗时操作
* 当任务执行完成后， IntentService 自动停止，不需要我们手动结束
* 如果多次启动 IntentService ，使用串行方式，依次执行 onHandlerIntent() ,执行完自动结束。那么每个耗时的操作都会消息的形式发生到消息队列中。
