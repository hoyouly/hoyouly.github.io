---
layout: post
title: 扫盲系列 - Java 多线程和锁机制
category: 扫盲系列
tags: Java 多线程
---
* content
{:toc}

# 多线程
多线程不是为了提高执行速度，而是为了提高程序的应用效率。程序在运行的时候，都是在抢CPU执行权，如果多线程程序，那么抢到CPU的执行权比单线程的要大，也就是说CPU在多线程程序中执行的时间比在单线程中的要长，所以提高了程序的应用效率，但是就算是多线程程序，哪个能抢到CPU执行权，也是不确定的，所以多线程具有随机性。

## 线程的调度模型
* 分时调度模型 所有线程轮流使用CPU，平均分配每个线程占有的CPU时间片
* 抢占式调度模型  优先让优先级高的线程使用CPU，如果线程优先级相同，那么随机选一个，优先级高的线程获得CPU的时间片相对高一些，java中就是使用的抢占式调度模型。

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

![](../../../../images/thread_state.jpg)  
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

### setDaemon(boolean on)
将该线程标记为守护线程或者用户线程，当正在运行的线程是守护线程时，java虚拟机退出。   
该方法必须在启动线程之前调用。   
JVM会判断线程的类型，如果线程全部是守护线程，那么JVM就退出。

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

---
搬运地址：    

[Java中的多线程你只要看这一篇就够了](https://www.cnblogs.com/wxd0108/p/5479442.html)   

[Java多线程学习（吐血超详细总结）](https://blog.csdn.net/Evankaka/article/details/44153709)   
