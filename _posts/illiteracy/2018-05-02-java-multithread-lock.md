---
layout: post
title: 扫盲系列之---Java 多线程和锁机制
category: 扫盲系列
tags: 多线程 锁
---
* content
{:toc}

# 多线程
## 常识
1. Thread 实际上也是实现了Runnable接口，
2. run()方法是多线程的一个约定，所有的多线程的代码都放到的run()方法中。
3. 所有多线程的代码都是通过运行Thread的start()方法来运行的，但是start()方法调用后并不是立即执行多线程代码，而是使该线程变成可执行状态（Runnable），什么时候执行由操作系统决定
4. 多线程是乱序执行的，所以只有乱序执行的的代码才有必要进行多线程操作。所有多线程代码执行的顺序都不是固定的，每次执行结果都是随机的。
5. start()方法重复调用会出现java.lang.IllegalThreadStateException
6. 不管是扩展Thread还是实现Runnable来实现多线程，最终通过Thread的对象的API来控制线程的。熟悉Thread类的API是进行多线程编程的基础。
7. 在java程序中，每次程序运行至少启动2个线程，一个是main线程，一个是JVM线程，

## Runnable的优点

* 适合多个相同程序的代码的线程处理同一个资源
* 可以避免java中单继承的限制
* 增加程序的健壮性，代码可以被多个线程控制，代码和数据独立
* 线程池只能放入实现Runnable和callable类的线程，不能直接放入继承Thread类的线程

## Callable
除了实现Runnable接口和继承Thread类之外，在JDK1.5之后，又添加了另外一种创建多线程的方式，就是实现Callable接口
1. 该接口的call()方法可以在线程结束的时候产生一个返回值。
2. 必须用ExecutorService.submit()方法调用
3. submit()返回的类似是Future对象，他用Callable返回结果的特定类型进行参数化，
4. 可以使用isDon()检查Futrue是否完成
5. 也可以使用get(),get()将直接阻塞至结果准备就绪。


## 线程转换状态
线程分为以下几种状态：
1. 新建状态 New :创建一个新的线程
2. 就绪状态 Runnable ： 线程创建后，在其他线程调用了该对象的start()方法，该对象的线程位于可运行线程池中。变得可运行，等待获得CPU的使用权。
3. 运行状态 Running : 就绪状态的线程获得CPU使用权，执行程序代码
4. 阻塞状态 Blocked : 线程因为某种原因放弃了CPU使用权，暂时停止运行，直到线程进入就绪状态，才有机会转到运行状态，阻塞状态分为三种：
  * 等待阻塞：运行的线程执行的wait()方法，JVM会把该线程放到等待池中，wait会释放持有的锁。
  * 同步阻塞：运行的线程在获取对象的同步锁时，若该对象的同步锁被别的线程占用。则JVM会把该对象放入锁池中。
  * 其他阻塞：运行的线程执行了sleep()或者join()方法，或者发出了IO请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时，join()等待线程终止或者超时，或者IO处理完毕，线程重新进入就绪状态，注意：sleep()不会释放持有锁
5. 死亡状态 Dead： 线程执行结束或者因为一场退出了run()方法，该线程结束生命周期。      

![](http://p5sfwb51p.bkt.clouddn.com/thread_state.jpg)  
## 线程调度
方式有以下几种
1. 调整线程优先级，Java线程有优先级，优先级高的线程会获得较多的运行机会。Java线程优先级用整数表示，取值范围是1~10，Thread类有三个静态常量
  * static int MAX_PRIORITY 线程可以具有最高优先级，取值是10
  * static int MIN_PRIORITY 线程可以具有最低优先级，取值是1
  * static int NORM_PRIORITY 分配线程的默认优先级，取值是5  

  Thread的setpriority()和getPriority()分别用来设置和获得优先级。
每个线程都有一个默认的优先级，主线程的默认优先级是 NORM_PRIORITY,线程的优先级具有继承权，比如A线程继承B线程，那么A和B具有相同的优先级

2. 线程睡眠  执行 sleep(),使线程进入阻塞状态，当睡眠结束，就进入就绪状态，sleep()平台移植性很好
3. 线程等待 Object类中有一个wait()方法，导致当前线程等待，直到其他线程调用了此对象的notify()或者notifyAll()唤醒，这两个方法也是Object类中的，等价于wait()方法
4. 线程让步,Thread.yield(),暂停当前正在执行的线程对象，把执行机会让给优先级相同或者更高的线程。
5. 线程加入 join() 等待其他线程终止，当前线程中调用另外一个线程的join()方法.则当前线程结束，直到另一个线程运行结束，当前现在再由阻塞状态转为就绪状态
6. 线程唤醒。Object类中有notify(),唤醒在此对象监视器上等待的单个线程，如果所有线程都在此对象上等待，则会选择唤醒其中一个，选择是任意的，并在现实做出决定时发生。线程通过调用其中一个wait(),在对象的监视器上等待。直到当前线程放弃此对象上的锁定，才能继续执行被唤醒的线程。被唤醒的线程将以常规方式与在该对象上主动同步的其他所有线程进行竞争。例如唤醒的线程在作为锁定此对象的下一个线程方面没有可靠的优势或者特权，类似的方法还有notfyAll(),唤醒在此对象上所有等待的线程

## 常用函数说明
### sleep()
在指定毫秒内让当前正在执行的线程休眠。暂停执行，   
有以下特点：  
* 使当前线程进入阻塞状态，让出CPU使用权，目的是不让当前线程独自霸占该CPU资源，以留一定的时间给其他线程机会
* Thread的静态方法，不能改变对象的的机锁，所以当一个sychronized块中调用了sleep(),线程虽然睡眠了。但是对象的锁并未释放，其他线程无法访问该对象，即使睡着了也持有该对象锁
* sleep()休眠后，线程并不一定能立即执行，这是因为可能其他线程正在执行并且没有被调度为放弃，除非该线程具有更高的优先级

### join()
Thread 的一个方法。等待该线程停止，也就是说该线程没停止之前，调用该线程的线程，一般是主线程只能等待。如果在main线程中创建了线程A，并且执行线程A 的start()和join()，那么主线程只等得到线程A执行完后才能停止。
```java
public static void main(String[] args) {
		System.out.println("main 线程开始");
		ThreadA threadA=new ThreadA("A");
		threadA.start();
		System.out.println("main 线程结束");
	}

	public static class ThreadA extends Thread {
		private String name;
		public ThreadA(String num){
			this.name=num;
		}

		@Override
		public void run(){
		System.out.println("线程 "+ name+" 开始执行");
			for(int i=0;i<5;i++){
				System.out.println(name+" 运行： "+i);
			}
			try{
				sleep((int)Math.random()*10);
			}catch(Exception e){

			}
			System.out.println("线程 "+ name+" 结束");
		}
	}
```
运行结果如下：
```java
main 线程开始
main 线程结束
线程 A 开始执行
A 运行： 0
A 运行： 1
A 运行： 2
A 运行： 3
A 运行： 4
线程 A 结束
```

如果线程A没有执行join方法，那么主线程是比线程A先执行完毕的，可是加上了join()方法后，结果就不同了
```java
public static void main(String[] args) {
		System.out.println("main 线程开始");
		ThreadA threadA=new ThreadA("A");
		threadA.start();
		try {
			threadA.join();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("main 线程结束");
	}

	public static class ThreadA extends Thread {
		private String name;
		public ThreadA(String num){
			this.name=num;
		}

		@Override
		public void run(){
		System.out.println("线程 "+ name+" 开始执行");
			for(int i=0;i<5;i++){
				System.out.println(name+" 运行： "+i);
			}
			try{
				sleep((int)Math.random()*10);
			}catch(Exception e){

			}
			System.out.println("线程 "+ name+" 结束");
		}
	}
```
运行结果：
```java
main 线程开始
线程 A 开始执行
A 运行： 0
A 运行： 1
A 运行： 2
A 运行： 3
A 运行： 4
线程 A 结束
main 线程结束
```
不管怎么运行，结果都是如此，线程A结束后，main线程才能结束。

### yield()
yield 有让步，屈服的意思，在这里就是暂停当前正在执行的线程对象，并让其他或者自己的线程执行，这就看谁先抢到CPU的执行权了，

### sleep()和yield()区别
1. sleep()是使得当前线程进入停滞状态，在执行sleep()这段时间，线程肯定不会执行，而yield()是使当前线程重新回到可执行状态，所以执行yield()的线程有可能进入到可执行状态后有马上执行
2. sleep()使得当前线程睡眠一段时间，进入不可运行状态，这段时间是程序控制的，而yield()是使得线程让出CPU控制权，但让出的时间是不可设置的，实际上yeild()操作如下：检测当前是否有优先级相同的处于可运行状态的线程，如果有，则把CPU控制权交给他，否则继续执行原来的线程，所以yield()是退让，把机会让给优先级相同的线程。
3. sleep()方法运行优先级较低的获得运行机会，而yield(),当前线程处于可运行状态，所以不可能把CPU控制权交给优先级较低的线程的。

### setpriority()
主要用来更改优先级
用法就是
```java
ThreadA threadA=new ThreadA("A");
ThreadA threadB=new ThreadA("B");
threadA.start();
threadB.start();
threadA.setPriority(Thread.MAX_PRIORITY);
threadB.setPriority(Thread.MIN_PRIORITY);
```
### interrupt()
不要以为是中断某个线程，他只是线程发出的一个中断信号，让线程在无限等待时，如死锁，能抛出抛出，从而结束线程，如果我们捕获这个异常，那么这个线程还是不会中断的。

### wait()
Object.wait(),与Object.notify()必须要在synchronized(obj)一起使用，也就是wait()和notify()是针对已获得obj锁操作的
* 从语法讲，wait()和notify()必须在sychronized(){}代码块里面，否则会运行报出异常java.lang.IllegalMonitorStateException
* 从功能讲，wait()就是说本线程在获得锁对象后，主动释放锁对象，同时本线程休眠，直到其他线程调用该对象的notify()唤醒该线程。才能继续获得该对象锁，继续执行。
* Object 类的方法，当一个对象执行了wait()方法后，它就进入到一个和该对象相关的等待池中，同时释放了对象锁，不过这只是暂时失去，等wait(long time)超时时间到以后，还需要返回该对象锁的，
* wait()后，其他线程可以访问
* notify()就是对对象锁的唤醒操作。但是notify()执行后，并不是马上释放锁对象，而是在相应的sychronized(){}代码快执行结束后，自动释放锁。   
* JVM在wait()对象锁的线程中随机选取一个线程，赋予锁对象，唤醒线程，继续执行。这样就在线程间提供了线程间同步，唤醒操作。

### sleep()和wait()区别
虽然二者都可以暂停当前线程，释放CPU控制权，但是wait()释放CPU控制权的时候，同时释放了对象锁的控制

#### 共同点
* 都是在多线程下，在程序调用出，阻塞指定的毫秒数，并且返回
* 都可以通过interrupt()打断暂停状态，从而是线程立刻抛出InterruptException,如果线程A希望立即结束线程B，那么可以在线程B对应的Thread实例调用interrupt(),如果此刻线程B正在sleep()/wait()/join(),则线程B会立即抛出InterruptException，在catch中直接return即可安全的介绍线程，但是要注意的，InterruptException是线程自己从内部抛出的，并不是interrupt()方法抛出的，对一个线程执行interrupt()，如果该线程正在执行普通的代码，根本就不会抛出InterruptException，但是线程一旦进入到wait()/sleep()/join()，则会立刻抛出InterruptException，

#### 不同点
* Thread 类 中 sleep()。join()，yield()等，Object类 中 wait(),notify()等
* 每一个对象都有一个锁来控制同步访问，synchronized 关键字可以和对象进行交互，来实现线程的同步，sleep()没有释放锁，而wait()释放了锁，使得其他线程可以同步控制块或者方法。
* wait(),notify(),notifyAll()只能在同步方法或者同步块中使用，而sleep()可以在任何地方使用，

总结： sleep()和wait()最大的区别是：
* sleep()睡眠时保持对象锁，仍然占有该锁
* wait()睡眠时，释放对象锁。
但是sleep()和wait()都可以通过Interrupt()方法打断线程的暂停状态，从而使线程立刻抛出InterruptException，

# 锁
乐观锁和悲观锁

## 乐观锁
认为竞争不会经常发生。任务读多写的少，因此拿数据不上锁。但是在更新的时候会判断在此期间有没有人去更新这个数据，采取在写时先读取版本号，然后加锁操作，比较上一次版本号，如果一样则更新，如果失败则重复比较写的操作。适用于多读场景   
java中的乐观锁都是通过CAS操作的，CAS是一种更新的原子操作，比较当前值跟穿入值是否一样，一样则更新，否则失败。
## 悲观锁
认为竞争经常发生，认为写的多，遇到并发的可能性高，所以每次拿数据都会上锁，这样别人读写这个暑假就的Block，直到拿到锁，   
java中的悲观锁就是synchroinized,AQS 框架下的锁则是先尝试CAS，去获取锁，获取不到才转换为悲观锁，如ReentrantLock

## Synchronized
会导致挣不到锁的线程进入阻塞状态，是java中的一个重量级锁，为了性能，JDK1.5，引入轻量级锁与偏向锁，默认启用了自旋锁，他们都是乐观锁
1. 如果一个对象多个synchronized方法，那么只要一个线程访问了其中一个synchronized方法，其他现在就不能访问这个对象的任意一个synchronized方法。
2. 可以修饰方法（同步方法），或者代码块（同步代码块），作用域是当前对象。
3. 不能继承，基类中的synchronized方法并不能能自动的转换到子类中
4. 细分，synchronized 可以作用于instance变量，object reference对象引用，static函数和class literals（类名称字面量，例如String.class ）身上
5. 无论是修饰方法或者代码快，取得的锁都是对象，而不是一个方法或者代码块，而且同步的方法还很可能被其他线程的对象访问
6. 每个对象只有一个锁lock与之关联
7. 实现同步是需要很大的系统开销的，可能造成死锁，所以尽量避免无所谓的同步控制

同步方法实质上是把synchronized作用在对象引用上  
死锁是线程间相互等待锁造成的，在实际中发生概率很小，真让写一个死锁，并不一定好使，d按时程序一旦发生死锁，必将死掉。

## 自旋锁
如果持有锁的线程在很短时间内能释放锁，那么等待锁的线程就不要进行内核态与用户态之间的切换进入阻塞状态，他们只需要等一等（自旋），持有锁的线程释放锁资源后就可以立即获得锁
，这样就避免用户线程和内核线程之间的切换了

## Lock
在java.util 包中，一共有三个实现类
ReentrantLock
ReentrantReadWriteLock.ReadLock
ReentrantReadWriteLock.WriteLock

主要目的和synchronized一样，都是解决同步问题，处理资源争端而产生的艺术。功能类似但是有一些区别
1. Lock 更灵活，可以自由定义多把锁的加锁解锁顺序，synchronized 要按照先加后解顺序。
2. 提供多种加锁方案，lock 阻塞式，tyrlock 无阻塞式，lockInterruptily 可打断时，还有trylock带超时时间版本，能力越大，责任越大，必须控制好加锁和解锁，否则会导致灾难。
3. Lock的性能更高

## ReentrantLock 可重入锁 使用方法
1. 创建一个实例   ReentrantLock r=new ReentrantLock()
2. 加锁  r.lock()或者r.lockInterruptily()
    * lockInterruptily() 可被打断，当线程alock 后，线程b阻塞，此时如果是lockInterruptily，那么调用b.interrupt(),b线程退出阻塞，并放弃对资源的争抢，进入catch块，如果使用lockInterruptily，必须throw interruptable exception 或catch
3. 释放锁 r.unlock(),必须要做，要放到finally里面，防止异常跳出正常流程，导致灾难。

## ReentrantReadWriteLock 可重入读写锁
ReentrantLock 是防止线程A在读数据，线程B写数据造成的数据不一致，那么两个线程都是读数据，就没必要加锁了，所以读写锁ReadWriteLock就产生了，而ReentrantReadWriteLock是 ReadWriteLock的具体实现，实现了读写分离，读锁是共享，写锁是独占，读和读之间不会排斥，读和写，写和读，写和写之间才会排斥。提升了读写性能

```java
ReentrantReadWriteLock lock = new ReentrantReadWriteLock()
　　ReadLock r = lock.readLock();
　　WriteLock w = lock.writeLock();
```
两个都有lock，unlock方法，

## Condition
使用 await(),signal(),signalAll() 替换Object中的wait(),notify()，notifyAll()，传统的线程的通信方式，Condition都可以实现，注意Condition绑定在Lock上的，要先创建一个Lock对象，然后创建Condition需要执行Lock的newCondition()方法，  
Condition的强大之处：可以为不同的线程创建多个Condition,而sychronized/wait()只有一个阻塞队列。notifyAll()唤醒所有阻塞队列的线程，而使用lock/Condition，可以实现多个阻塞队列，signalAll只会唤起某个阻塞队列下的线程。



[一道多线程面试题引起的自我救赎](https://segmentfault.com/a/1190000006671595)

---
搬运地址：   
[Java中的多线程你只要看这一篇就够了](https://www.cnblogs.com/wxd0108/p/5479442.html)   
[Java多线程学习（吐血超详细总结）
](https://blog.csdn.net/Evankaka/article/details/44153709)    
[java 中的锁 -- 偏向锁、轻量级锁、自旋锁、重量级锁](https://blog.csdn.net/zqz_zqz/article/details/70233767)
