---
layout: post
title: Android 消息分发机制
category: 读书笔记
tags: Handler Message Looper ThreadLocal
---
* content
{:toc}

## 消息机制
就是Handler的运行机制以及Handler所带的MessageQueue 和Looper的工作过程。这三个是一个整体，只不过我们经常用到Handler，所以也称为Handler消息机制。Handler主要工作就是将一个任务切换到某个指定的线程中去执行。
主要解决 子线程无法直接访问UI线程的问题。
## 创建Handler的两种方式
1. 构造函数中传递一个Callback对象
2. 创建一个Handler子类，重写handleMessage()

```java
public Handler(Callback callback, boolean async) {
		//匿名类、内部类或本地类都必须申明为static，否则会警告可能出现内存泄露
		if (FIND_POTENTIAL_LEAKS) {
			final Class<? extends Handler> klass = getClass();
			if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) && (klass.getModifiers() & Modifier.STATIC) == 0) {
				Log.w(TAG, "The following Handler class should be static or leaks might occur: " + klass.getCanonicalName());
			}
		}
		//必须先执行Looper.prepare()，才能获取Looper对象，否则为null.
		mLooper = Looper.myLooper();//从当前线程的TLS中获取Looper对象
		if (mLooper == null) {
			throw new RuntimeException("Can't create handler inside thread that has not called Looper.prepare()");
		}
		mQueue = mLooper.mQueue;//消息队列，来自Looper对象
		mCallback = callback;//回调方法
		mAsynchronous = async;//设置消息是否为异步处理方式
	}

  public Handler(Looper looper, Callback callback, boolean async) {
		mLooper = looper;
		mQueue = looper.mQueue;
		mCallback = callback;
		mAsynchronous = async;
	}
```
在我们创建Handler的时候，会有并且必须一个Looper对象，要么是创建的时候传递过来，要么就是通过Looper.myLooper()获得，这个从Looper对象中取得的MessageQueue，
一个线程中Looper只有一个，相应的MessageQueue 也只有一个   
如下图
![添加图片](../../../../images/creat_handler.png)
这两种的区别后面会说道   


注意： 我们在创建 Handler的时候，经常被Android Studio提示`The following Handler class should be static or leaks might occur: `。

如下图所示   
![添加图片](../../../../images/handler_leank.png)
看if() 语句，现在知道原因了吧，匿名类、内部类或本地类都必须申明为static，否则会警告可能出现内存泄露

## 发送消息
虽然有很多中方法，但是主要可以归结两种
1. sendMessage 形式
2. post() Runnable 对象形式   

其实不管是sendMessage()还是post() 都会执行 sendMessageDelayed(),
```java
public final boolean sendMessageDelayed(Message msg, long delayMillis) {
	if (delayMillis < 0) {
		delayMillis = 0;
	}
	return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
//这里面会接受一个 uptimeMillis，这个如果没设置，会是当前时间 （SystemClock.uptimeMillis()），
//如果设置延迟时间，则是当前时间+延迟时间（SystemClock.uptimeMillis() + delayMillis）
//这个时间是很重要，是进入MessageQueue 中队列先后的依据
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
		MessageQueue queue = mQueue;//这个mQueue就是创建Handler的时候从Looper对象中取得的
		if (queue == null) {
			RuntimeException e = new RuntimeException(this + " sendMessageAtTime() called with no mQueue");
			Log.w("Looper", e.getMessage(), e);
			return false;
		}
		return enqueueMessage(queue, msg, uptimeMillis);
	}

  private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
		msg.target = this;
		if (mAsynchronous) {
			msg.setAsynchronous(true);
		}
		return queue.enqueueMessage(msg, uptimeMillis);
	}
```
由上可知 sendMessageDelayed() -> sendMessageAtTime()-> sendMessageAtTime() -> enqueueMessage()。enqueueMessage()主要做了两件事
1. 保存当前的handler对象到msg.target中
2. 执行到了 MessageQueue 的enqueueMessage()中。

流程图如下
![添加图片](../../../../images/handlemessage_post.png)

post()的时候，会把Runnable对象转成一个Message，
```java
private static Message getPostMessage(Runnable r) {
		Message m = Message.obtain();
		m.callback = r; //这个在后面dispatchMessage()的时候还会用到
		return m;
	}
```
![添加图片](../../../../images/handler_post.png)


不知道有没有发现，为啥不用 SystemClock.currentTimeMillis()呢，而是用了 SystemClock.uptimeMillis()。区别在哪里呢？
### SystemClock.uptimeMillis() 和 SystemClock.currentTimeMillis() 区别

* System.currentTimeMillis() 方法产生一个标准的自1970年1月1号0时0分0秒所差的毫秒数。该时间可以通过调用setCurrentTimeMillis(long)方法来手动设置，也可以通过网络来自动获取。这个方法得到的毫秒数为“1970年1月1号0时0分0秒 到 当前手机系统的时间”的差。因此如果在执行时间间隔的值期间用户更改了手机系统的时间，那么得到的结果是不可预料的。因此它不适合用在需要时间间隔的地方，如Thread.sleep, Object.wait等，因为它的值可能会被改变。
* SystemClock.uptimeMillis()方法用来计算自开机启动到目前的毫秒数。如果系统进入了深度睡眠状态（CPU停止运行、显示器息屏、等待外部输入设备）该时钟会停止计时，但是该方法并不会受时钟刻度、时钟闲置时间亦或其它节能机制的影响。因此SystemClock.uptimeMillis()方法也成为了计算间隔的基本依据，比如Thread.sleep()、Object.wait()、System.nanoTime()以及Handler都是用SystemClock.uptimeMillis()方法。这个时钟是保证单调性,适用于计算不跨越设备的时间间隔。

简单一句话就是 System.currentTimeMillis()  可被手动修改而SystemClock.uptimeMillis() 不能手动修改。

继续看  MessageQueue #enqueueMessage()代码
## MessageQueue #enqueueMessage()
代码不算太长，中间都有注释
```java
boolean enqueueMessage(Message msg, long when) {
		// 每一个普通Message必须有一个target
		if (msg.target == null) {
			throw new IllegalArgumentException("Message must have a target.");
		}
		if (msg.isInUse()) {
			throw new IllegalStateException(msg + " This message is already in use.");
		}

		synchronized (this) {
			if (mQuitting) {//正在退出时
				IllegalStateException e = new IllegalStateException(msg.target + " sending message to a Handler on a dead thread");
				Log.w("MessageQueue", e.getMessage(), e);
				msg.recycle();//回收msg，加入到消息池
				return false;
			}

			msg.markInUse();
			msg.when = when;
			Message p = mMessages;
			boolean needWake;
			if (p == null || when == 0 || when < p.when) {
				//p为null(代表MessageQueue没有消息） 或者msg的触发时间是队列中最早的， 则进入该该分支
				msg.next = p;
				mMessages = msg;
				needWake = mBlocked;//当阻塞时需要唤醒
			} else {
				//插入队列中间。 通常，我们不必唤醒事件队列，除非队列头部存在障碍，并且消息是队列中最早的异步消息。
				needWake = mBlocked && p.target == null && msg.isAsynchronous();
				Message prev;
				for (; ; ) {
					prev = p;
					p = p.next;
					if (p == null || when < p.when) {
						break;
					}
					if (needWake && p.isAsynchronous()) {
						needWake = false;
					}
				}
				msg.next = p; // invariant: p == prev.next
				prev.next = msg;
			}

			//消息没有退出，我们认为此时mPtr != 0
			if (needWake) {
				//用于唤醒功能
				nativeWake(mPtr);
			}
		}
		return true;
	}
```
转换成流程图如下
![添加图片](../../../../images/enqueuemessage.png)
主要做了两件事
1. 插入到消息队列中
怎么插入队列中的，就是图中红框中的实现。主要包括两种
	1. 插入到队列头部 依据 p == null || when == 0 || when < p.when
  	* 如果当前队列为null   这个好理解，队列为空，那么来的第一个肯定是队头
  	* 该消息的处理时间为0   这个有点迷糊，什么时候 when 为0 呢，handler提供了一个sendMessageAtFrontOfQueue(),看名字也猜出来了，VIP消息，过来直接插入到队头。怎么能保证直接在队头呢，when =0即可，因为当前时间肯定大于0的，如果一个消息when=0,那么肯定排在最前面了
  	* 该消息的处理时间小于头消息的处理时间 这个也可以理解，虽然when 不为0，但是目前消息队列中第一个消息时间大于当前时间，那么就排在队头了。
	2. 插入到队列中间/后面
  遍历整个队列，然后找到第一个比该消息处理时间大的，然后排在他的前面即可。

2. 如果是即时消息并且线程是阻塞状态，唤醒Looper中等待的线程

## 创建Looper

虽然添加到队列，什么时候处理消息呢？我们的Looper就该出现了
前面创建Handler的时候需要一个Looper对象，如果没有Looper对象的话，Handler就会抛异常，可是这个Looper对象什么时候创建的呢?  在ActivityThead中 我们找到了创建的地方
```java
public static void main(String[] args) {
  ...
  //准备主线程的Looper
  Looper.prepareMainLooper();
  //将该进程绑定到AMS
  thread.attach(false);

  if (sMainThreadHandler == null) {
  	//保存进程对应的主线程Handler
  	sMainThreadHandler = thread.getHandler();
  }
  //进入主线程的消息循环
	Looper.loop();
}
```
记得之前一个搞C++的同学问我，Android 程序的最先执行的函数是哪个啊，怎么没找到main()函数啊，之前我一直不知道，现在可以告诉他了，在ActivityThread中。

然后我们就看看 Looper.prepareMainLooper()中干了啥吧，其实就是创建了一个Looper对象，
```Java
public static void prepareMainLooper() {
		//设置不允许退出的Looper
		prepare(false);
		synchronized (Looper.class) {
			//将当前的Looper保存为主Looper，每个线程只允许执行一次。
			if (sMainLooper != null) {
				throw new IllegalStateException("The main Looper has already been prepared.");
			}
			sMainLooper = myLooper();
		}
	}

  private static void prepare(boolean quitAllowed) {
		//每个线程只允许执行一次该方法，第二次执行时线程的TLS已有数据，则会抛出异常。
		if (sThreadLocal.get() != null) {
			throw new RuntimeException("Only one Looper may be created per thread");
		}
		//创建Looper对象，并保存到当前线程的TLS区域
		sThreadLocal.set(new Looper(quitAllowed));
	}

  private Looper(boolean quitAllowed) {
  		mQueue = new MessageQueue(quitAllowed);//创建MessageQueue
  		mThread = Thread.currentThread();
  }

  public static Looper myLooper() {
		return sThreadLocal.get();
	}
  static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
其实主要的就是prepare()，就是我们经常说的，在子线程中创建Handler之前，必须先Looper.prepare()才行   如下图
![添加图片](../../../../images/looper_prepare.png)
注意创建Looper的同时，也就创建了一个MessageQueue ，ThreadLocal 后面会说到   
Looper创建成功了，那么就开始干活吧，开启传送带。loop()方法。
## Looper # loop()
删掉了log之后，代码如下，关键地方也做了注释，相信不需要过多解释。
```java
public static void loop() {
		//获取TLS存储的Looper对象
		final Looper me = myLooper();
		if (me == null) {
			throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
		}
		//获取Looper对象中的消息队列
		final MessageQueue queue = me.mQueue;

		Binder.clearCallingIdentity();
		final long ident = Binder.clearCallingIdentity();

		for (; ; ) {//进入loop的主循环方法
			Message msg = queue.next(); // might block //可能会阻塞
			if (msg == null) {  //没有消息，则退出循环
				return;
			}
			//用于分发Message
			msg.target.dispatchMessage(msg);
			//确保分发过程中identity不会损坏
			final long newIdent = Binder.clearCallingIdentity();
			//将Message放入消息池  以便重复利用。
			msg.recycleUnchecked();
		}
	}
```
再来一份流程图
![添加图片](../../../../images/looper_loop.png)
这里面主要看两个地方
1. 取消息  queue.next()
2. 分发消息 msg.target.dispatchMessage()   


先看next()方法吧
## MessageQueue # next()
```java
Message next() {
		//如果消息循环已经退出并被处理，返回这里。如果应用程序试图在不支持退出后重新启动looper，就会发生这种情况。
		final long ptr = mPtr;
		//当消息循环已经退出，则直接返回
		if (ptr == 0) {
			return null;
		}

		int pendingIdleHandlerCount = -1; // -1 only during first iteration  // 循环迭代的首次为-1
		int nextPollTimeoutMillis = 0;//代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。
		//无限循环，如果队列中没有消息，那么next()方法就会一直阻塞在这里，当新消息到来的时候，next方法就会返回这条消息并且将其从链表中移除
		for (; ; ) {
			if (nextPollTimeoutMillis != 0) {
				Binder.flushPendingCommands();
			}
			//在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里
			//此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。
			//阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
			nativePollOnce(ptr, nextPollTimeoutMillis);

			synchronized (this) {
				// Try to retrieve the next message.  Return if found.
				final long now = SystemClock.uptimeMillis();
				Message prevMsg = null;
				Message msg = mMessages;
				if (msg != null && msg.target == null) {
					//msg.target为空是一类特殊消息（栅栏消息），用于阻塞所有同步消息，但是对异步消息没有影响，
          //在这个前提下，当头部是特殊消息时需要往后找是否有异步消息
					// Stalled by a barrier.  Find the next asynchronous message in the queue.
					//当消息Handler为空时，查询MessageQueue中的下一条异步消息msg，则退出循环。
					do {
						prevMsg = msg;
						msg = msg.next;
					} while (msg != null && !msg.isAsynchronous());
				}

				// 走到这一步, 有两种可能,
				// 一种是遍历到队尾没有发现异步消息,
				// 另一种是找到queue中的第一个异步消息

				if (msg != null) { // 找到queue中的第一个异步消息
					if (now < msg.when) { // 没有到消息的执行时间
						//当异步消息触发时间大于当前时间，则设置下一次轮询的超时时长
						nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
					} else {// 当前消息到达可以执行的时间, 直接返回这个msg
						// 获取一条消息，并返回
						mBlocked = false;
						if (prevMsg != null) {
							prevMsg.next = msg.next;
						} else {
							//更新队头指针 mMessages
							mMessages = msg.next;
						}
						//移除队头消息并返回
						msg.next = null;
						if (false) Log.v("MessageQueue", "Returning message: " + msg);
						return msg;
					}
				} else {
					//没有消息
					nextPollTimeoutMillis = -1;
				}

				//消息正在退出，返回null
				if (mQuitting) {
					dispose();
					return null;
				}

				// 如果queue中没有msg, 或者msg没到可执行的时间,那么现在线程就处于空闲时间了, 可以执行IdleHandler了
				if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
					// pendingIdleHandlerCount在进入for循环之前是被初始化为-1的  并且没有更多地消息要进行处理
					pendingIdleHandlerCount = mIdleHandlers.size();
				}
				if (pendingIdleHandlerCount <= 0) {
					//没有idle handlers 需要运行，则循环并等待。
					mBlocked = true;
					continue;
				}

				if (mPendingIdleHandlers == null) {
					mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
				}
				mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
			}

			//只有第一次循环时，会运行idle handlers，执行完成后，重置pendingIdleHandlerCount为0.
			for (int i = 0; i < pendingIdleHandlerCount; i++) {
				final IdleHandler idler = mPendingIdleHandlers[i];
				//只有第一次循环时，会运行idle handlers，执行完成后，重置pendingIdleHandlerCount为0.
				mPendingIdleHandlers[i] = null; // release the reference to the handler

				boolean keep = false;
				try {
					keep = idler.queueIdle();//idle时执行的方法
				} catch (Throwable t) {
					Log.wtf("MessageQueue", "IdleHandler threw exception", t);
				}

				if (!keep) {
					synchronized (this) {
						// 如果之前addIdleHandler中返回为false,
						// 就在执行完这个IdleHandler的callback之后, 将这个idler移除掉
						mIdleHandlers.remove(idler);
					}
				}
			}
			//重置idle handler个数为0，以保证不会再次重复运行
			pendingIdleHandlerCount = 0;
			//当调用一个空闲handler时，一个新message能够被分发，因此无需等待可以直接查询pending message.
			nextPollTimeoutMillis = 0;
		}
	}
```

代码很长，相应的地方也做了注释，但是可能还是看不懂，简单做一下解释。
1. 当首次进入或者所有消息都完成后，由于此刻没有消息（mMessage=null），这个时候nextPollTimeoutMillis = -1 ，然后会处理一些不紧急的任务（IdleHandler），之后线程会一直阻塞，直到被主动唤醒，插入消息后会根据消息类型决定是否唤醒。
2. 读取消息时，如果发现有消息屏障，则跳过后面的同步消息
3. 如果拿到的消息还没到时间，则重新赋值给nextPollTimeoutMills=延迟时间。for()循环再次进入的时候，执行 nativePollOnce(ptr, nextPollTimeoutMillis); 阻塞线程，直到线程被唤醒
4. 如果消息是即时消息，更新mMessage 为msg.next,mMessage 一直都是队头消息，移除该msg,(msg.next=null),并把msg 返回次消息给loop()处理，

一句话总结就是 ：next()有一个无限循环的方法 for (; ; ) {} ，如果消息队列中没有消息，那么next()方法会一直阻塞在这里，直到有新的消息到来，next()会返回该消息并把这个消息从链表中移除。




### 实现延迟消息
如果拿到的消息没有到时间，就会重新设置超时时间 nextPollTimeoutMillis ，然后下次for循环的时候，会调用 nativePollOnce(ptr, nextPollTimeoutMillis)进行阻塞，这是一个本地方法，通过C++实现的，最终会通过Linux的epoll监听 文件描述符的写入事件来实现延迟。

接下来说 msg.target.dispatchMessage()
之前就一直好奇，虽然Looper，MessageQueue 只有一个，但是我们在主线程中会创建很多Handler对象，可能一个Activity/Fragment中就会创建一个。怎么能保证在Activity中创建的Handler 没有传递到Fragment创建的Handler中处理呢，原因就在这里，在我们封装消息的时候，会把该handler作为msg的target，然后分发消息的时候，也是使用该msg.target来分发的，这就保证了不会乱窜的情况
既然 msg.target 是一个Handler，那么我们就看看Handler 的dispathcMessage()是怎么实现的吧
## Handler # dispatchMessage()

```Java
public void dispatchMessage(Message msg) {
		if (msg.callback != null) {//post()的时候执行到这里
			handleCallback(msg);
		} else {
			if (mCallback != null) {//创建Handler的时候传递的callback对象
				if (mCallback.handleMessage(msg)) {//返回true，就不执行handleMessage()了，
					return;
				}
			}
			handleMessage(msg);
		}
	}
	private static void handleCallback(Message message) {
		message.callback.run();
	}
```
这个代码很短，可以直接看流程图
![添加图片](../../../../images/handler_dispatchmessage.png)
这几个if else 判断，就和创建handler 和handler发送消息有关
1. msg.callback  就是 post()中创建的Runnable对象，最后执行到了run()方法中
2. mCallback 就是创建Handler的时候传递的callback对象，如果mCallback.handleMessage(msg) 返回true，就不会执行handleMessage了，可是如果返回false，还是可以执行handleMessage()的。   
这就给我们提供了一种思路，<font color="#ff000" > 在mCallback.handleMessage(msg) 中修改发送过来的消息，然后返回false，这样还会会继续执行handleMessage()，达到偷梁换柱的目的</font>

## 解惑
尽管分析完了，可是还有几个困惑的地方。比如为啥不会出现ANR，在子线程中发送消息，怎么就会到主线程中执行？ThreadLocal 的作用是啥。现在一一解惑。

### 死循环为啥没出现ANR
不知道大家又没有疑问，
Looper 的loop()中有一个循环for (; ; ) {} 用来一直循环分发消息
MessageQueue 的 next() 也有一个循环for (; ; ) {} ，用来查找下一个消息。
这两个都是死循环，next()还会阻塞，为啥没出现ANR呢？
我们首先知道出现ANR的原因：
1. 当前事件没有机会得到处理，即主线程正在处理前一个事件，没有即时的完成或者Looper被某种原因阻塞了
2. 当前事件正在处理，但是没有即时完成

我们再看看ActivityThread 中的main()
```java
public static void main(String[] args) {
  ...
  //准备主线程的Looper，
  Looper.prepareMainLooper();
  //将该进程绑定到AMS
  thread.attach(false);

  if (sMainThreadHandler == null) {
  	//保存进程对应的主线程Handler
  	sMainThreadHandler = thread.getHandler();
  }
  //进入主线程的消息循环
	Looper.loop();

	throw new RuntimeException("Main thread loop unexpectedly exited");
```
会发现一个很奇怪的线程，loop()执行完毕后，直接抛异常了，很诡异啊，但是我们很少能遇到这个异常，这是因为 loop()是一个死循环，永远不会结束，也就执行不到了throw new RuntimeException()中了，程序也就不会崩溃了。可是这和ANR有啥关系呢？

**那还是没解释清楚，为啥这个死循环不会造成ANR呢**     
我们知道Android是由事件控制的，而Looper.loop() 不断的接收事件，处理事件，每一个点击，触摸或者Activity的生命周期都是运行在Looper控制上的，如果Looper停止了，那么应用就停止了。所以Looper不能停止，这就是loop()设计成死循环的原因。
<font color="#ff000" >我们只能说某个消息或者说对消息的处理阻塞了Looper.loop(),而不是Looper.loop()阻塞了它.</font> 主从关系不能颠倒。这也是为啥我们不能在UI线程中执行耗时操作的原因,因为耗时操作会阻塞loop()运行。
主线程Looper从消息队列中取出消息，当读取消息后，主线程阻塞。子线程往主线程中发送消息，唤醒主线程，主线程唤醒只是为了读取消息，当消息读取完毕，就又进入了睡眠，因此loop()的循环并不会对CPU性能有过多的消耗。也就不会产生ANR。


### 在子线程中发送消息，怎么就会到主线程中执行呢
这个问题一直都没搞明白，什么时候切换到主线程了呢？再次查看的时候，猛地想明白了。
其实并不没有切换到主线程了，而是一个典型的生产者-消费者模式。

**消费者：MessageQueue 的next()， 在主线程中执行，一直从队列中取消息，所以是消费者;    
生产者：消费者：MessageQueue 的 enqueueMessage()， 在子线程中执行,一直往队列中插入消息，所以是生产者。**    
为了证明这个事实，需要先确定几件事：
1. 主线程中 创建 Handler 对象 ，肯定在主线程中
2. Handler # handleMessage() 执行在主线程中。
Looper.loop() 在ActivityThread的main()中执行的，肯定是主线程，上面也说到。Looper.loop()会执行MessageQueue.next(),取得消息后执行Handler.handleMessage(),这两个也是在主线程中的。
3. 在子线程中我们会执行 Handler# sendMessage()，所以Handler# sendMessage()在子线程，相应的，  MessageQueue# enqueueMessage()也就在子线程中。   
如下图所示。  
![添加图片](../../../../images/handler_thread.png)
因为涉及到多线程，所以在enqueueMessage() 和next() 中我们都看到了 `synchronized (this) {}`这样的代码。this 就是MessageQueue对象，Looper只有一个，那么相应的MessageQueue也只有一个，锁对象只有一个，就能防止多线程对同一队列的同时操作。这样就保证了多线程数据的安全。


### ThreadLocal 的作用
看了一大堆资料，也没搞想明白ThreadLocal的作用。然后突然灵光一现，明白了。
我们知道，一个线程中只能有一个Looper对象，但是可以创建多个Handler对象，而创建Handler的时候，必须有一个Looper对象。这该咋办呢？怎么处理才好呢，就用到了ThreadLocal   
ThreadLocal 是线程内部的数据存储类，通过他可以在指定的线程中存储数据，数据存储后，只能在指定的线程中可以获取到存储数据。
我们可以在创建Looper的时候，把该对象存入到ThreadLocal中，然后下次想要得到该线程的Looper，直接通过该ThreadLocal 取得即可。Looper 里面就是这么做
```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static void prepare(boolean quitAllowed) {
		//每个线程只允许执行一次该方法，第二次执行时线程的TLS已有数据，则会抛出异常。
		if (sThreadLocal.get() != null) {
			throw new RuntimeException("Only one Looper may be created per thread");
		}
		//创建Looper对象，并保存到当前线程的TLS区域
		sThreadLocal.set(new Looper(quitAllowed));
	}

	public static Looper myLooper() {
			return sThreadLocal.get();
	}
```
在Looper中创建一个静态的ThreadLocal变量sThreadLocal，在prepare()中创建Looper对象，然后保存到ThreadLocal中。sThreadLocal.set(new Looper(quitAllowed));，同时对外提供得到该Looper的静态方法。当我们在某个线程中想要创建Looper的时候，如果是第一次，需要执行Looper.prepare(),创建一个Looper对象并且保存到ThreadLocal,再次创建的时候，只需要使用Looper.myLooper()得到该Looper对象即可。

但我们所以在主线程中创建Handler的时候，没有指明需要的Looper对象，也没有执行Looper.prepare(),可是却能创建成功，并且正常执行，为啥呢。
因为Looper.prepare() 在ActivityThread 的main()中已经执行了。而不需要传递Looper对象，是因为在Handler的构造函数自己通过Looper.myLooper()拿到了。
```java
public Handler(Callback callback, boolean async) {
	  ...
		//必须先执行Looper.prepare()，才能获取Looper对象，否则为null.
		mLooper = Looper.myLooper();//从当前线程的TLS中获取Looper对象
		if (mLooper == null) {
			throw new RuntimeException("Can't create handler inside thread that has not called Looper.prepare()");
		}
		mQueue = mLooper.mQueue;//消息队列，来自Looper对象å
		mCallback = callback;//回调方法
		mAsynchronous = async;//设置消息是否为异步处理方式
	}
```
其实不用ThreadLocal 也可以，只需要弄一个全局的HashMap供Handler查找指定线程的Looper对象即可。那样就需要一个类似LooperManager的类进行管理。既然ThreadLocal 有这样的功能，干嘛重复造轮子呢？
### 为啥一个线程中只能有一个Looper
这个很好理解，如果有两个甚至多个Looper，那么就有多个MessageQueue,那么sendMessage()过来的消息该放到哪个MessageQueue呢？怎么能保证消息的顺序执行呢？所以只能有一个Looper对象，相应的，也就只能执行一次Looper.prepare().

---
搬运地址：    

Android 开发艺术探索    

[深入理解MessageQueue](https://www.jianshu.com/p/8c829dc15950)
