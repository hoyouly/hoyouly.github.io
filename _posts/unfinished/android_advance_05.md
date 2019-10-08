---
layout: post
title:
category:
tags:
---
* content
{:toc}

## CPU
### 获得设备的CPU信息
```
//获取CPU 核心数目
cat /sys/devcies/system/cpu/possible

//获取某个CPU的频率
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
```

WAITING 和BLOCKED

BLOCKED 只线程正在等待获取锁，对应的是下面代码中的情况一
WAITING 是指线程正在等待其他线程的唤醒动作，对应的是代码中的情况二

```java
synchronized(object){ //情况一 在这里卡主  BLOCKED
  object.wait();  //情况二，在这里卡主  WAITING
}
```
当线程处于WAITING状态时候，它不仅会释放CPU资源，还会将持有的Object锁也同时释放，


### ARN 日志说明
其中 utm 代表 utime，HZ 代表 CPU 的时钟频,将 utime 转换为毫秒的公式是“time * 1000/HZ
```java
// 线程名称 ; 优先级 ; 线程 id; 线程状态
"main" prio=5 tid=1 Suspended
  // 线程组 ;  线程 suspend 计数 ; 线程 debug suspend 计数 ;
  | group="main" sCount=1 dsCount=0 obj=0x74746000 self=0xf4827400
  // 线程 native id; 进程优先级 ; 调度者优先级 ;
  | sysTid=28661 nice=-4 cgrp=default sched=0/0 handle=0xf72cbbec
  // native 线程状态 ; 调度者状态 ; 用户时间 utime; 系统时间 stime; 调度的 CPU
  | state=D schedstat=( 3137222937 94427228 5819 ) utm=218 stm=95 core=2 HZ=100
  // stack 相关信息
  | stack=0xff717000-0xff719000 stackSize=8MB

```



---
搬运地址：
