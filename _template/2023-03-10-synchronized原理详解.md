---
layout: post 
title: synchronized原理详解
date: 2023-03-09
tags: [并发]
description: 穷首皓经，追求梦想！
---

## 前言

一说到锁，大家可能会想到分布式锁。当然，只用jvm级别锁是无法解决分布式环境下的同步问题。
但是这并不意味着分布式环境下jvm锁就没用了。  

JVM的锁是用于同一个进程(即JVM)内部的多个线程之间的同步的机制。分布式环境的话，
其同步考虑的是多个进程(即多个JVM)的线程之间的同步问题。这个时候JVM的锁机制自然无用。

分布式环境往往比较复杂，总有一些功能只在一个jvm里面跑，这时候还是能用到jvm级别锁的。并且理解了jvm级别锁，也能帮助我们更好的理解分布式锁。

>欢迎关注个人公众号【好好学技术】交流学习

## synchronized基本使用

Java中的synchronized是一个关键字，它用于实现线程之间的同步。  
synchronized关键字可以用来修饰实例方法、静态方法和代码块。

synchronized方法和synchronized同步代码块的区别：
- synchronized同步代码块只是锁定了该代码块，代码块外面的代码还是可以被访问的。
- synchronized方法是粗粒度的并发控制，某一个时刻只能有一个线程执行该synchronized方法。
- synchronized同步代码块是细粒度的并发控制，只会将块中的代码同步，代码块之外的代码可以被其他线程同时访问。

 ![img.png](_template/img.png)



## 原理
。，

它的底层原理是基于Java对象头的Monitor（监视器）实现的。
Java对象头中有一个用于存储锁信息的字段，该字段的值为0表示对象没有被锁定，为1表示对象已经被锁定。

当一个线程进入synchronized块时，它会尝试获取锁，如果锁没有被其他线程占用，
则该线程将获得锁并将对象头中的锁信息字段的值置为1。如果锁已经被其他线程占用，则该线程将进入等待状态。


当一个线程释放锁时，它会将对象头中的锁信息字段的值置为0，同时唤醒一个等待的线程。
如果有多个线程在等待该锁，则唤醒其中一个线程，唤醒哪个线程是由JVM内部实现的线程调度机制来决定的。
在Java虚拟机中，每个对象都有一个与之对应的监视器锁（Monitor锁）。当一个线程试图获取某个对象的Monitor锁时，
该线程会进入Monitor锁的EntrySet中等待，直到该对象的Monitor锁被释放。当一个线程成功获取某个对象的Monitor锁时，该线程就可以执行synchronized块中的代码了，同时该对象的Monitor锁的Owner指向该线程，表示该线程拥有该对象的Monitor锁。
在Java虚拟机中，每个线程都有一个线程私有的锁记录器（Lock Record）。当一个线程在执行synchronized块时，
它会将该对象的Monitor锁的Owner指向自己，并将该对象的Monitor锁的计数器加1，表示该线程已经获取了该对象的Monitor锁。
当该线程执行完synchronized块中的代码后，它会将该对象的Monitor锁的计数器减1，如果计数器为0，
说明该线程已经释放了该对象的Monitor锁，同时将该对象的Monitor锁的Owner字段置为null，
表示该对象的Monitor锁没有被任何线程持有。





## synchronized锁升级
无锁->偏向锁->轻量级锁->重量级锁
锁升级
绝大部分的线程获得锁以后，在非常短的时间内会去释放锁。自旋会占用cpu资源，所以在指定的自旋次数之后，如果还没有获得轻量级锁，锁膨胀成重量级锁
设置自旋次数  jvm参数修改
自适应自旋  jdk自己做适应

假如有两个线程A、B
1.只有A访问--> 偏向锁
2.A和B交替访问->轻量级锁->自旋， 默认旋转十次，还得不到锁，升级为重量级锁 （自旋是在用户态，不到内核态）
3.多个线程访问->阻塞

什么时候用sync?什么时候用自旋锁？
执行时间长，线程多，用系统锁（sync），否则太多线程自旋
执行时间短，线程少，用自旋

Java对象头：
在Hotspot虚拟机中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充；Java对象头是实现synchronized的锁对象的基础，一般而言，synchronized使用的锁对象是存储在Java对象头里。它是轻量级锁和偏向锁的关键
Mawrk Word：
Mark Word用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等。Java对象头一般占有两个机器码（在32位虚拟机中，1个机器码等于4字节，也就是32bit），下面就是对象头的一些信息：

![img_2.png](_template/img_2.png)
![img_1.png](_template/img_1.png)



说说synchronized关键字的底层原理是什么？
synchronized 同步语句块的实现，使用的是 monitorenter 和 monitorexit 指令，其中 monitorenter 指令指向同步代码块的开始位置，monitorexit 指令则指明同步代码块的结束位置。
synchronized 修饰方法，并没有 monitorenter 指令和 monitorexit 指令，取得代之的是 ACC_SYNCHRONIZED 标识，该标识指明了该方法是一个同步方法，JVM 通过该 ACC_SYNCHRONIZED 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

如果一个线程第一次synchronized那里，获取到了myObject对象的monitor的锁，计数器加1，然后第二次synchronized那里，会再次获取myObject对象的monitor的锁，这个就是重入加锁了，然后计数器会再次加1，变成2。
这个时候，其他的线程在第一次synchronized那里，会发现说myObject对象的monitor锁的计数器是大于0的，意味着被别人加锁了，然后此时线程就会进入block阻塞状态，什么都干不了，就是等着获取锁。
接着如果出了synchronized修饰的代码片段的范围，就会有一个monitorexit的指令，在底层。此时获取锁的线程就会对那个对象的monitor的计数器减1，如果有多次重入加锁就会对应多次减1，直到最后，计数器是0。



能聊聊你对CAS的理解以及其底层实现原理可以吗？
![img_3.png](_template/img_3.png)

缺点
循环时间长开销大
只能保证一个共享变量的原子操作
ABA问题