---
layout: post
title: 并发编程之BlockingQueue实现原理
date: 2023-05-22
tags: [并发]
description: 找工作真难啊
---


## 什么是队列

**队列（Queue）** 是一种线性数据结构，也称为**队列树（Queue Tree）** 或 **先进先出（First In First Out, FIFO）队列**。这是一种环形结构，其中队列中的元素按照插入顺序或者访问顺序排列。

队列中的元素可以是任何类型的数据，包括数字、字符串、布尔值、对象等。队列的最大容量由系统决定，java中一般为 Integer.MAX\_VALUE即2的32次方减1。

队列的基本操作包括：入队（Enqueue）、出队（Dequeue）、判断队列是否为空（IsEmpty）和获取队头元素（Front）。


## 有界队列和无界队列

*   有界队列：队列中的元素数量有限制，当队列已满时，新元素将被拒绝插入。

*   无界队列：队列中的元素数量没有限制。

在Java中，有界队列和无界队列的区别主要体现在以下几个方面：

1.  可用性：有界队列在队列已满时会拒绝插入新元素，而无界队列不会受到这个限制。
2.  空间使用：有界队列需要预留一定的空间来存储队列元素，而无界队列不需要。
3.  插入和删除效率：有界队列在插入和删除元素时需要考虑队列的容量，而无界队列不需要。
4.  实现难度：有界队列比无界队列更容易实现，但需要预留更多的空间和额外的内存管理。

总之，有界队列和无界队列在使用上有很大的区别，需要根据具体的应用场景来选择合适的数据结构。

## 数组和链表的区别

Java中实现队列最常见的数据结构就是数组和链表。\
接下来我们看看数组和联表有什么区别呢。

### 数组结构

数组是一段连续固定的内存空间。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6561feba1ac349e78372ac3f81b344a8~tplv-k3u1fbpfcp-watermark.image?)

优点： 可以直接根据下标查询，时间复杂度为O(1), 支持随机访问。\
缺点： 增加、删除元素效率慢（插入或删除后续元素都需要移动，以保证连续）。

### 链表结构

链表结构又分为单链表、双向链表和环形链表。我们来看看双向链表。

在双向链表中，每个节点都有两个指针，一个指向前一个节点，另一个指向后一个节点。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eebab71de29748449424950ca38a537d~tplv-k3u1fbpfcp-watermark.image?)

优点： 增加删除效率高，只需要修改双向指针指向的节点就可以。\
缺点： 查询效率慢，除了头尾节点，访问中间节点需要遍历，时间复杂度为O(n)。

### 数组和链表比较

如果查询场景多，数组优于链表。如果增删场景多，链表结构由于数组。\
数组扩容效率低，链表扩容简单。

## 什么是阻塞队列

阻塞队列是一种特殊的队列数据结构，它允许在队列为空时等待，同时在队列已满时阻塞插入操作。这种队列通常用于实现生产者-消费者模型，其中一个生产者将元素添加到队列中，而一个消费者从队列中获取元素。由于阻塞队列的特性，生产者和消费者可以共享同一个队列，从而避免了线程安全问题。

阻塞队列有三种基本类型：LinkedBlockingQueue、ArrayBlockingQueue 和SynchronousQueue。其中，LinkedBlockingQueue是阻塞队列的标准实现，ArrayBlockingQueue在某些方面类似于LinkedBlockingQueue，而SynchronousQueue则是线程安全的阻塞队列。

阻塞队列的实现通常涉及到使用线程池来管理队列操作，以确保多线程环境下的线程安全。在创建阻塞队列时，需要注意队列的容量和大小，以避免队列溢出和插入操作被阻塞的情况。

## ArrayBlockingQueue

底层基于数组实现.

### api简单使用

```java
package com.fandf.test.queue;  
  
import java.util.concurrent.ArrayBlockingQueue;  
import java.util.concurrent.TimeUnit;  
  
/**  
* @author fandongfeng  
* @date 2023/5/21 16:57  
*/  
public class QueueDemo {  
  
    public static void main(String[] args) throws InterruptedException {  
        ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<String>(2);  
        //存入队列  
        queue.offer("java");  
        queue.offer("python");  
        queue.offer("aaa");  
        //poll 从队列中取出第一个元素，如果取出成功则删除该元素  
        //输出  
        //java  
        //python  
        //null  
        System.out.println(queue.poll());  
        System.out.println(queue.poll());  
        System.out.println(queue.poll());  

        //存入队列  
        queue.offer("java");  
        queue.offer("python");  
        //peek取出第一个元素，但不会删除  
        //输出  
        //java  
        //java  
        //java  
        System.out.println(queue.peek());  
        System.out.println(queue.peek());  
        System.out.println(queue.peek());  

        //最多等待三秒，若队列还是为空，直接返回为null  
        //输出  
        //java  
        //python  
        //阻塞三秒  
        //null  
        System.out.println(queue.poll(3, TimeUnit.SECONDS));  
        System.out.println(queue.poll(3, TimeUnit.SECONDS));  
        System.out.println(queue.poll(3, TimeUnit.SECONDS));  
    }  
  
}
```

| 方式       | 抛出异常    | 有返回值且不抛出异常 | 阻塞等待 | 超时等待      |
| -------- | ------- | ---------- | ---- | --------- |
| 添加       | add     | offer      | put  | offer(,,) |
| 移除       | remove  | poll       | take | poll(,,)  |
| 查看队列首位元素 | element | peek       | 无    | 无         |

### 源码解析

new ArrayBlockingQueue<>(int capacity);

```java
/**  
* Creates an {@code ArrayBlockingQueue} with the given (fixed)  
* capacity and default access policy.  
*  
* @param capacity the capacity of this queue  
* @throws IllegalArgumentException if {@code capacity < 1}  
*/  
public ArrayBlockingQueue(int capacity) {  
    this(capacity, false);  
}

/**  
* Creates an {@code ArrayBlockingQueue} with the given (fixed)  
* capacity and the specified access policy.  
*  
* @param capacity the capacity of this queue  
* @param fair if {@code true} then queue accesses for threads blocked  
* on insertion or removal, are processed in FIFO order;  
* if {@code false} the access order is unspecified.  
* @throws IllegalArgumentException if {@code capacity < 1}  
*/  
public ArrayBlockingQueue(int capacity, boolean fair) {  
    if (capacity <= 0)  
        throw new IllegalArgumentException();  
    //直接初始化数组长度
    this.items = new Object[capacity];  
    lock = new ReentrantLock(fair);  
    notEmpty = lock.newCondition();  
    notFull = lock.newCondition();  
}
```

这里看到，初始化ArrayBlockingQueue的时候，直接会初始化队列指定大小的数组items。

查看其offer存入方法源码，使用ReentrantLock锁来保证线程安全

```java
public boolean offer(E e) {  
    checkNotNull(e);  
    final ReentrantLock lock = this.lock;  
    lock.lock();  
    try { 
        //count: Number of elements in the queue
        //items: The queued items
        //当前队列元素之和等于队列长度，直接返回入队失败
        if (count == items.length)  
            return false;  
        else {  
            //入队
            enqueue(e);  
            return true;  
        }  
    } finally {  
        lock.unlock();  
    }  
}

/**  
* Inserts element at current put position, advances, and signals.  
* Call only when holding lock.  
*/  
private void enqueue(E x) {  
    // assert lock.getHoldCount() == 1;  
    // assert items[putIndex] == null;  
    final Object[] items = this.items;  
    //putIndex: items index for next put, offer, or add 数组入队位置
    items[putIndex] = x;  
    //下个入队位置++， 如果超过数组长度，置为0
    if (++putIndex == items.length)  
        putIndex = 0;  
    //队列元素之和+1
    count++;  
    //notEmpty: Condition for waiting takes
    notEmpty.signal();  
}
```

查看offer(E e, long timeout, TimeUnit unit)方法

```java
/**  
* Inserts the specified element at the tail of this queue, waiting  
* up to the specified wait time for space to become available if  
* the queue is full.  
*  
* @throws InterruptedException {@inheritDoc}  
* @throws NullPointerException {@inheritDoc}  
*/  
public boolean offer(E e, long timeout, TimeUnit unit)  
throws InterruptedException {  
  
    checkNotNull(e);  
    long nanos = unit.toNanos(timeout);  
    final ReentrantLock lock = this.lock;  
    lock.lockInterruptibly();  
    try {  
        //队列已满
        while (count == items.length) {  
            //如果等待时间小于0，直接返回false
            if (nanos <= 0)  
                return false;  
            //生产者阻塞等待
            nanos = notFull.awaitNanos(nanos);  
        }  
        enqueue(e);  
        return true;  
    } finally {  
        lock.unlock();  
    }  
}
```

查看其poll出队方法源码

```java
public E poll() {  
    final ReentrantLock lock = this.lock;  
    //加锁保证线程安全，并且入队和出队使用的同一把锁，为什么不使用读写锁呢？
    lock.lock();  
    try {  
        //dequeue出队方法
        return (count == 0) ? null : dequeue();  
    } finally {  
        lock.unlock();  
    }  
}
```

看看dequeue()源码

```java
/**  
* Extracts element at current take position, advances, and signals.  
* Call only when holding lock.  
*/  
private E dequeue() {  
    // assert lock.getHoldCount() == 1;  
    // assert items[takeIndex] != null;  
    //
    final Object[] items = this.items;  
    @SuppressWarnings("unchecked")  
    // takeIndex 默认为0，默认取第0个位置
    E x = (E) items[takeIndex];  
    //取出成功后，将第0个位置变为null
    items[takeIndex] = null;  
    //取出位置+1，如果已经超过数组长度，则置为0
    if (++takeIndex == items.length)  
        takeIndex = 0;  
    //队列元素之和--
    count--;  
    if (itrs != null)  
        itrs.elementDequeued();  
    //notFull：Condition for waiting puts
    notFull.signal();  
    return x;  
}
```

### 简单总结下

1.  ArrayBlockingQueue是基于数组实现的。
2.  入队方法 采用ReentrantLock锁来保证线程安全问题。
3.  有界队列，初始化时会创建指定大小的数组。
4.  入队和出队使用的同一把锁，会相互阻塞。

## 基于ArrayBlockingQueue实现生产者与消费者案例

```java
package com.fandf.test.queue;  
  
import java.util.concurrent.ArrayBlockingQueue;  
import java.util.concurrent.LinkedBlockingQueue;  
  
/**  
* @author fandongfeng  
* @date 2023/5/21 19:05  
*/  
public class ArrayBlockingQueueDemo {  
  
    public static void main(String[] args) {  
        LinkedBlockingQueue<String> linkedBlockingQueue = new LinkedBlockingQueue<>();  
        linkedBlockingQueue.offer("a");  
        linkedBlockingQueue.poll();  
        //创建队列，供生产者投递，消费者消费  
        ArrayBlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<>(10);  
        //生产者线程  
        new Thread(() -> {  
            for (int i = 0; i < 20; i++) {  
            boolean offer = blockingQueue.offer(i);  
            System.out.println(Thread.currentThread().getName() + ",入队：" + i + ",结果：" + offer);  
            }  
        }, "生产者").start();  

        //消费者线程  
        new Thread(() -> {  
            while (true) {  
                Integer poll = blockingQueue.poll();  
                if(poll != null) {  
                    System.out.println(Thread.currentThread().getName() + ",出队：" + poll);  
                }  
            }  
        }, "消费者").start();  

    }  
  
}
```

输出

```java
生产者,入队：0,结果：true
生产者,入队：1,结果：true
生产者,入队：2,结果：true
生产者,入队：3,结果：true
生产者,入队：4,结果：true
消费者,出队：0
生产者,入队：5,结果：true
生产者,入队：6,结果：true
消费者,出队：1
消费者,出队：2
生产者,入队：7,结果：true
生产者,入队：8,结果：true
消费者,出队：3
生产者,入队：9,结果：true
消费者,出队：4
生产者,入队：10,结果：true
消费者,出队：5
生产者,入队：11,结果：true
消费者,出队：6
生产者,入队：12,结果：true
消费者,出队：7
消费者,出队：8
消费者,出队：9
生产者,入队：13,结果：true
消费者,出队：10
生产者,入队：14,结果：true
消费者,出队：11
生产者,入队：15,结果：true
消费者,出队：12
生产者,入队：16,结果：true
消费者,出队：13
生产者,入队：17,结果：true
消费者,出队：14
生产者,入队：18,结果：true
消费者,出队：15
生产者,入队：19,结果：true
消费者,出队：16
消费者,出队：17
消费者,出队：18
消费者,出队：19
```

## LinkedBlockingQueue

查看offer源码

```java
/**  
* Inserts the specified element at the tail of this queue if it is  
* possible to do so immediately without exceeding the queue's capacity,  
* returning {@code true} upon success and {@code false} if this queue  
* is full.  
* When using a capacity-restricted queue, this method is generally  
* preferable to method {@link BlockingQueue#add add}, which can fail to  
* insert an element only by throwing an exception.  
*  
* @throws NullPointerException if the specified element is null  
*/  
public boolean offer(E e) {  
    if (e == null) throw new NullPointerException();  
    final AtomicInteger count = this.count;  
    if (count.get() == capacity)  
        return false;  
    int c = -1;  
    Node<E> node = new Node<E>(e); 
    //写锁
    final ReentrantLock putLock = this.putLock;  
    putLock.lock();  
    try {  
        if (count.get() < capacity) {  
            enqueue(node);  
            c = count.getAndIncrement();  
            if (c + 1 < capacity)  
            notFull.signal();  
        }  
    } finally {  
        putLock.unlock();  
    }  
    if (c == 0)  
        signalNotEmpty();  
    return c >= 0;  
}
```

查看poll源码

```java
public E poll() {  
    final AtomicInteger count = this.count;  
    if (count.get() == 0)  
        return null;  
    E x = null;  
    int c = -1;  
    //写锁
    final ReentrantLock takeLock = this.takeLock;  
    takeLock.lock();  
    try {  
        if (count.get() > 0) {  
            x = dequeue();  
            c = count.getAndDecrement();  
            if (c > 1)  
                notEmpty.signal();  
        }  
    } finally {  
        takeLock.unlock();  
    }  
    if (c == capacity)  
        signalNotFull();  
    return x;  
}
```

LinkedBlockingQueue入队，出队使用的是两把锁，采用的是读写分离锁的思路。

## LinkedBlockingQueue与ArrayBlockingQueue区别

1.  ArrayBlockingQueue底层基于数组，而LinkedBlockingQueue底层基于链表。
2.  ArrayBlockingQueue默认有界队列，必须指定队列容器大小。LinkedBlockingQueue默认无界队列，长度为Integer最大值。
3.  ArrayBlockingQueue读写采用的同一把锁，LinkedBlockingQueue用的是读写分离锁。
4.  ArrayBlockingQueue使用int count计数，LinkedBlockingQueue使用AtomicInteger。
