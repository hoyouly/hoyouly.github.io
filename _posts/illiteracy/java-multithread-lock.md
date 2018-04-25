---
layout: post
title: 扫盲系列之---Java 多线程和锁机制
category: 扫盲系列
tags: RecycleView
---
* content
{:toc}

# 多线程
start()方法调用后并不是立即执行多线程代码，而是使该线程变成可执行状态（Runnable），什么时候执行由操作系统决定
多线程是乱序执行的，所以只有乱序执行的的代码才有必要进行多线程操作。所有多线程代码执行的顺序都不是固定的，每次执行结果都是随机的。

start()方法重复调用会出现java.lang.IllegalThreadStateException

Thread 实际上也是实现了Runnable接口，run()方法是多线程的一个约定，所有的多线程的代码都放到的run()方法中。所有多线程的代码都是通过运行Thread的start()方法来运行的，因此，不管是扩展Thread还是实现Runnable接口来实现多线程，最终通过Thread的对象的API来控制线程的。熟悉Thread类的API是进行多线程编程的基础。

## 实现Runnable接口比继承Thread类的好处

* 适合多个相同程序的代码的线程处理同一个资源
* 可以避免java中单继承的限制
* 增加程序的健壮性，代码可以被多个线程控制，代码和数据独立
* 线程池只能放入实现Runnable和callable类的线程，不能直接放入继承Thread类的线程


在java程序中，每次程序运行至少启动2个线程，一个是main线程，一个是JVM线程，

## 线程转换状态
![](http://p5sfwb51p.bkt.clouddn.com/thread_state.jpg)
1. 新建状态 New :创建一个新的线程
2. 就绪状态 Runnable ： 线程创建后，在其他线程调用了该对象的start()方法，该对象的线程位于可运行线程池中。变得可运行，等待获得CPU的使用权。
3. 运行状态 Running : 就绪状态的线程获得CPU使用权，执行程序代码
4. 阻塞状态 Blocked : 线程因为某种原因放弃了CPU使用权，暂时停止运行，直到线程进入就绪状态，才有机会转到运行状态，阻塞状态分为三种：
  * 等待阻塞：运行的线程执行的wait()方法，JVM会把该线程放到等待池中，wait会释放持有的锁。
  * 同步阻塞：运行的线程在获取对象的同步锁时，若该对象的同步锁被别的线程占用。则JVM会把该对象放入锁池中。
  * 其他阻塞：运行的线程执行了sleep()或者join()方法，或者发出了IO请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时，join()等待线程终止或者超时，或者IO处理完毕，线程重新进入就绪状态，注意：sleep()不会释放持有锁
5. 死亡状态 Dead： 线程执行结束或者因为一场退出了run()方法，该线程结束生命周期。

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
在指定毫秒内让当前正在执行的线程休眠。暂停执行
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

### setpriority(),更改优先级
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






# 锁
乐观锁和悲观锁
---
搬运地址：   
[Java中的多线程你只要看这一篇就够了](https://www.cnblogs.com/wxd0108/p/5479442.html)[Java多线程学习（吐血超详细总结）
](https://blog.csdn.net/Evankaka/article/details/44153709)  
