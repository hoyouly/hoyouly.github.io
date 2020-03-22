---
layout: post
title: 街题系列 - Java 两个线程交替打印
category: 街题系列
tags: Java Synchronized Thread
---
* content
{:toc}
两个线程交替打印，不管是一个线程打印奇数，一个线程打印偶数，或者一个线程打印数字，一个线程打印字母，这种题目应该面试的时候没少遇到吧。看网上有好几种实现方式。今天我就列举一下。以一个线程打印数字，一个线程打印字母为例吧

## 第一种 Synchronized + await()+ notify()
这样应该是我们常想到的吧
```java
public class PrintNumAndChar5 {

    public static void main(String[] args) {
        Integer[] nums = {1, 2, 3, 4, 5};
        Character[] chars = {'a', 'b', 'c', 'd', 'e'};

        Object lock = new Object();
        new PrintThread<>("PrintNumThread ", nums, lock).start();
        new PrintThread<>("PrintCharThread", chars, lock).start();
    }

    public static class PrintThread<T> extends Thread {
        private T[] nums;
        private Object object;

        public PrintThread(String threadName, T[] nums, Object object) {
            super(threadName);
            this.nums = nums;
            this.object = object;
        }

        public void run() {
            try {
                for (int i = 0; i < nums.length; i++) {
                    synchronized (object) {
                        System.out.println(Thread.currentThread().getName() + "  : " + nums[i]);
                        //唤醒其他线程
                        object.notify();
                        //使当前线程阻塞，当线程执行wait()方法时候，会释放当前的锁，然后让出CPU，进入等待状态。
                        object.wait();
                        // object.notify();
                    }
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                synchronized (object) {//结束线程，防止出现某个线程一直wait()的情况。
                    object.notify();
                }
            }
        }
    }
}
```
1. 两个对象交替打印，肯定有一个线程打印，另外一个线程等待，然后打印完后，处于等待状态，同事通知等待的线程。这样循环就可以了
所以两个线程需要同一把锁。即 Object lock = new Object();
2. 核心在 run()方法中。
很多资料上一般都是这样写： 打印操作->notify() ->wait() -> notify()

```java
synchronized (object) {
  System.out.println(Thread.currentThread().getName() + "  : " + nums[i]);
  //唤醒其他线程
  object.notify();
  //使当前线程阻塞，当线程执行wait()方法时候，会释放当前的锁，然后让出CPU，进入等待状态。
  object.wait();
  object.notify();
}
```
怎么有些蒙圈的，notify(),wait(),notify(),玩绕口令呢？
啥意思啊，这里需要解释一下这两个函数的意思
### wait() & notify()
* wait()、notify()方法属于Object中的方法；对于Object中的方法，每个对象都拥有。
* wait()方法：该方法用来使得当前线程进入等待状态，直到接到通知或者被中断打断为止。在调用wait()方法之前，线程必须要获得该对象的对象级锁；换句话说就是该方法只能在同步方法或者同步块中调用，如果没有持有合适的锁的话，线程将会抛出异常IllegalArgumentException。调用wait()方法之后，当前线程则释放锁。
* notify()方法：该方法用来唤醒处于等待状态获取对象锁的其他线程。如果有多个线程则线程规划器任意选出一个线程进行唤醒，使其去竞争获取对象锁，但线程并不会马上就释放该对象锁，wait()所在的线程也不能马上获取该对象锁，要程序退出同步块或者同步方法之后，当前线程才会释放锁，wait()所在的线程才可以获取该对象锁。
* wait()方法是释放锁的；notify()方法不释放锁，必须等到所在线程把代码执行完。


还是看不太明白，那就简单点
1. notify()和 wait() 必须在 同步代码块，也就是synchronized()中执行
2. 当前线程调用 notify(),并不能立即唤醒换线当前线程，只有退出同步代码块之后才可以。却能唤醒其他处于wait()线程。
3. wait()方法是释放锁的；notify()方法不释放锁，必须等到所在线程把代码执行完。

这就有点像小时候和小伙伴们小游戏机，这个只能一个人玩，而很多小伙伴都要玩，那咋办呢，一人一局，轮流玩。一个人在玩的时候，其他小伙伴没事干，就睡觉。一局玩完，叫醒其他小伙伴
所以上面的代码就可以这样解释了
```java
synchronized (object) {//一个小伙伴拿到了游戏机
  //开始玩游戏
  System.out.println(Thread.currentThread().getName() + "  : " + nums[i]);
  //游戏玩完，叫醒其他小伙伴，虽然小伙伴都醒了，可是游戏机只有一个，还在他手里，这个时候玩不了
  object.notify();
  //自己开始睡觉，同时把游戏机扔出去，让其他小伙伴抢吧。谁抢到谁玩
  object.wait();
}//同步代码块结束，
```
但是这就有一个问题，不知道你们有没有发现，当最后一个人玩完后，自己会睡觉，再没有其他小伙伴叫醒自己了。
这就是一个线程阻塞到这里了。所以很多人就在 wait()后面又加了一个notify()用来再次唤醒，因为执行完notify()之后，就退出同步代码块了，锁也就释放了，wait() 所在的线程才可以获取到锁。其实没必要，在for()循环结束后就可以了执行notify()就可以了，如下

```java
public void run() {
    try {
        for (int i = 0; i < nums.length; i++) {
            synchronized (object) {
                System.out.println(Thread.currentThread().getName() + "  : " + nums[i]);
                //唤醒其他线程
                object.notify();
                //使当前线程阻塞，当线程执行wait()方法时候，会释放当前的锁，然后让出CPU，进入等待状态。
                object.wait();
                // object.notify();
            }
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        synchronized (object) {//结束线程，防止出现某个线程一直wait()的情况。
            object.notify();
        }
    }
}
```
## 第二种 自旋锁 AtomicBoolean+ yield()

```java
public class PrintNumAndChar {

    public static void main(String[] arg) {
        AtomicBoolean atom = new AtomicBoolean(true);
        int[] nums = {1, 2, 3, 4, 5};
        char[] chars = {'a', 'b', 'c', 'd', 'e'};
        new PrintNumThread(nums, atom).start();
        new PrintCharThread(chars, atom).start();
    }

    public static class PrintNumThread extends Thread {
        private int[] nums;
        private AtomicBoolean mAtom;

        public PrintNumThread(int[] nums, AtomicBoolean atom) {
            this.nums = nums;
            this.mAtom = atom;
        }

        @Override
        public void run() {
            for (int i = 0; i < nums.length; i++) {
                while (!mAtom.get()) { //mAtom 默认值是true，所以第一次进来不会走到这里。
                    Thread.yield();// 让步，暂停当前正在执行的线程对象
                }
                System.out.print(nums[i] + " ");
                mAtom.set(false);//把值设置为false，这样再次进入for 循环的时候，就会在while循环中一直等待。
            }
        }
    }

    public static class PrintCharThread extends Thread {

        private char[] chars;
        private AtomicBoolean mAtom;

        public PrintCharThread(char[] chars, AtomicBoolean atom) {
            this.chars = chars;
            this.mAtom = atom;
        }

        @Override
        public void run() {
            for (int i = 0; i < chars.length; i++) {
                while (mAtom.get()) {
                    Thread.yield();
                }
                System.out.print(chars[i] + " ");
                mAtom.set(true);
            }
        }
    }
}

```
* 自旋锁: 如果持有锁的线程在很短时间内能释放锁，那么等待锁的线程就不要进行内核态与用户态之间的切换进入阻塞状态，他们只需要等一等（自旋），持有锁的线程释放锁资源后就可以立即获得锁 ，这样就避免用户线程和内核线程之间的切换了
* Thread.yield()   
  yield 让步的意思。yield()就是暂停当前正在执行的线程对象,并让其他或者自己的线程执行，这就看谁先抢到CPU的执行权了，yield()是使当前线程重新回到可执行状态，所以执行yield()的线程有可能进入到可执行状态后又马上执行
* AtomicBoolean  原子变量的类，在多线程中是安全的。  
整个了流程就是这样的

这个可以用火车上上厕所的例子。

```java

//乘客A去厕所，如果发现门是开着的，然后直接进去了，并且把门关了，
//如果要是没开门的话，就会一直在敲门，询问是否结束了啊
while (mAtom.get()) {
    Thread.yield();
}
//乘客A开始处理生理问题
System.out.print(chars[i] + " ");
//办完手续后，打开门。
mAtom.set(true);
```

## 第三种 LockSupport
```java
public class PrintNumAndChar2 {

    public static void main(String[] args) {
        Integer[] nums = {1, 2, 3, 4, 5};
        Character[] chars = {'a', 'b', 'c', 'd', 'e'};

        PrintThread printNum = new PrintThread(nums);
        PrintThread printChar = new PrintThread(chars);

        printNum.setUnparkThread(printChar);
        printChar.setUnparkThread(printNum);
        printNum.start();
        printChar.start();
    }

    public static class PrintThread<T> extends Thread {
        private T[] nums;
        private Thread thread;

        public PrintThread(T[] a1) {
            this.nums = a1;
        }

        public void setUnparkThread(Thread printChars) {
            this.thread = printChars;
        }

        public void run() {
            for (int i = 0; i < nums.length; i++) {
                System.out.print(nums[i]);
                LockSupport.unpark(thread);//唤醒处于阻塞状态的线程
                LockSupport.park();//阻塞当前线程，只有调用unpark()或者被中断，才能从park()返回
            }
            LockSupport.unpark(thread);
        }
    }
}
```
这个和第一种很像，就不过多解释了。这个不需要使用同步代码块，更方便一些。

---
搬运地址：    

[一道多线程面试题引起的自我救赎）](https://segmentfault.com/a/1190000006671595)    

[浅谈wait()和notify()](https://www.cnblogs.com/stupid-chan/p/9465625.html)
