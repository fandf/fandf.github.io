---
layout: post
title: ThreadLocal如何避免内存泄漏
date: 2023-05-19
tags: [java]
description: 找工作真难啊
---


## ThreadLocal简介
`ThreadLocal`是Java 中的一个线程本地存储机制，它允许每个线程拥有一个独立的本地存储空间，用于存储该线程的变量。`ThreadLocal`提供了一种简单的方式来解决多线程环境下共享变量的问题，避免了在多线程环境下出现的线程安全问题。


## ThreadLocal简单用法

```java
public class ThreadLocalDemo {  
    private static final ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> 0);  
    
    //当前值+1
    public static void increment() {  
        int value = threadLocal.get();  
        threadLocal.set(value + 1);  
    }  

    public static void main(String[] args) throws InterruptedException {  
        for (int i = 0; i < 10; i++) {  
            new Thread(() -> {  
                String threadName = Thread.currentThread().getName();  
                increment();  
                System.out.println(threadName + " 当前threadLocal的值为：" + threadLocal.get());  
            }).start();  
        }  
    }  
  
}
```
输出结果为

```java
Thread-0 当前threadLocal的值为：1
Thread-5 当前threadLocal的值为：1
Thread-3 当前threadLocal的值为：1
Thread-4 当前threadLocal的值为：1
Thread-6 当前threadLocal的值为：1
Thread-2 当前threadLocal的值为：1
Thread-9 当前threadLocal的值为：1
Thread-1 当前threadLocal的值为：1
Thread-7 当前threadLocal的值为：1
Thread-8 当前threadLocal的值为：1
```

我们发现每个线程虽然都共享同一个threadLocal实例，但它们并没有发生相互干扰的情况，而是各自产生独立的值，这是因为我们通过ThreadLocal为每一个线程提供了单独的副本。

## 使用场景
### spring事务模板类
```java
//使用示例
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();

//查看源码
public static TransactionStatus currentTransactionStatus() throws NoTransactionException {  
    TransactionInfo info = currentTransactionInfo();  
    if (info == null || info.transactionStatus == null) {  
        throw new NoTransactionException("No transaction aspect-managed TransactionStatus in scope");  
    }  
    return info.transactionStatus;  
}

//currentTransactionInfo方法
@Nullable  
protected static TransactionInfo currentTransactionInfo() throws NoTransactionException {  
    return transactionInfoHolder.get();  
}

//这里使用了ThreadLocal
private static final ThreadLocal<TransactionInfo> transactionInfoHolder =  
new NamedThreadLocal<>("Current aspect-driven transaction");

```

### HttpServletRequest

项目中要获取当前HttpServletRequest可以使用
```java
HttpServletRequest request =  
((ServletRequestAttributes)RequestContextHolder.getRequestAttributes()).getRequest();
```
我们来看看RequestContextHolder.getRequestAttributes()方法
```java
@Nullable  
public static RequestAttributes getRequestAttributes() {  
    RequestAttributes attributes = requestAttributesHolder.get();  
    if (attributes == null) {  
        attributes = inheritableRequestAttributesHolder.get();  
    }  
    return attributes;  
}

private static final ThreadLocal<RequestAttributes> requestAttributesHolder =  
new NamedThreadLocal<>("Request attributes");  
  
private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder =  
new NamedInheritableThreadLocal<>("Request context");

```

### aop调用链传递
LCN/seata都是使用ThreadLocal传递调用链的，这里就不展开讲了。

## 内存泄漏与内存溢出

### 内存泄漏

内存泄漏指的是程序中存在某些对象或资源没有被妥善地释放，导致这些对象或资源一直占用着内存，而无法被回收。随着时间的推移，这些未释放的对象或资源会越来越多，最终耗尽系统的内存资源，导致系统崩溃。

常见的内存泄漏包括：

-   对象被创建后，没有及时被销毁，成为垃圾对象。
-   没有正确关闭IO资源。
-   缓存没有被清空。
-   静态集合类对象未删除引用。
-   单例模式下对象未及时释放等。

### 内存溢出

内存溢出指的是程序在申请内存时，无法获得足够的内存空间，导致程序无法正常运行。通常情况下，当程序需要使用的内存超过了系统能够提供的内存时，就会发生内存溢出。

常见的内存溢出包括：

-   堆内存溢出：由于创建了过多的对象或者某些对象太大，导致堆内存不足。
-   栈内存溢出：由于方法调用过多或者某些方法的递归调用层数过多，导致栈内存不足。
-   永久代内存溢出：由于创建了过多的类或者字符串，导致永久代内存不足。

### 区别

内存泄漏和内存溢出的区别在于它们发生的原因和表现形式。内存泄漏是指对象或者资源无法被妥善释放，导致系统资源浪费，而内存溢出则是指系统不能分配所需内存，导致程序崩溃或者异常。通常情况下，内存泄漏会逐渐消耗系统资源，而内存溢出则是突然发生的。

解决内存泄漏的方法是找到未被正确释放的对象或资源，并手动进行释放。而解决内存溢出的方法则需要优化程序代码，减少内存使用量，或增加系统内存大小等方式来解决。


## java强软弱虚

Java中的引用类型有四种：强引用（Strong Reference）、软引用（Soft Reference）、弱引用（Weak Reference）和虚引用（Phantom Reference）。它们之间的主要区别在于对象被垃圾回收时的行为不同。

### 强引用

强引用是默认类型的引用，当我们通过“new”关键字创建一个对象时，该对象会被分配到堆内存中，并且默认情况下，该对象的引用是强引用。只要强引用存在，垃圾回收器就不会将其回收。

例如：

```java
Object obj = new Object();
```

在上面的代码中，obj是一个强引用，因此只有当obj变量被显示地设置为null时，才能使对象成为垃圾，等待垃圾回收器收集。

### 软引用

软引用可以让对象存活更长时间，直到内存不足时才回收它。如果垃圾回收器需要更多的内存，则会回收只被软引用引用的对象。当一个对象只被软引用引用时，它会被保留在内存中，直到系统内存不够用或者垃圾回收器需要更多空间为止。通过软引用可以实现一些缓存功能。

例如：

```java
SoftReference<Object> softRef = new SoftReference<>(new Object());
```

在上面的代码中，softRef是一个软引用，当垃圾回收器需要内存时，它可以将该对象回收，并释放所占用的内存。

### 弱引用

弱引用比软引用生命期更短，当一个对象只被弱引用引用时，当垃圾回收器运行时，不管当前内存是否充足，都会将其回收。弱引用通常用于实现缓存机制或者观察者模式。

例如：

```java
WeakReference<Object> weakRef = new WeakReference<>(new Object());
```

在上面的代码中，weakRef是一个弱引用，这意味着垃圾回收器可以随时将该对象回收，而无需考虑系统内存是否充足。

### 虚引用

虚引用也称为幽灵引用，与其他三种引用方式不同，它并不会决定对象是否能存活。如果一个对象只有虚引用，那么就像没有任何引用一样，它的内存会被回收，但是在回收之前会调用finalize()方法。虚引用主要用于管理DirectBuffer的生命周期。

例如：

```java
PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), null);
```

在上面的代码中，phantomRef是一个虚引用，当垃圾回收器发现该对象的内存已被回收时，它会将其插入队列中，并在下一次调用垃圾回收器时通知引用对象被回收了。

## ThreadLocal原理

### ThreadLocal

ThreadLocal是一个泛型类，它提供了get()、set()和remove()方法来获取、设置和删除当前线程的变量副本。它的原理是在每个Thread对象中都有一个ThreadLocalMap类型的私有变量threadLocals，该变量存储着当前线程所对应的所有ThreadLocal变量的值。

```java
public T get() { 
    //获取当前线程
    Thread t = Thread.currentThread();  
    //获取当前线程的ThreadLocalMap变量
    ThreadLocalMap map = getMap(t);  
    if (map != null) {  
        ThreadLocalMap.Entry e = map.getEntry(this);  
        if (e != null) {  
            @SuppressWarnings("unchecked")  
            T result = (T)e.value;  
            return result;  
        }  
    }  
    return setInitialValue();  
}

public void set(T value) {  
    Thread t = Thread.currentThread();  
    ThreadLocalMap map = getMap(t);  
    if (map != null)  
        map.set(this, value);  
    else  
        createMap(t, value);  
}

public void remove() {  
    ThreadLocalMap m = getMap(Thread.currentThread());  
    if (m != null)  
        m.remove(this);  
}
```

### ThreadLocalMap

ThreadLocalMap是ThreadLocal的内部类，它实际上就是一个HashMap，用于存储当前线程所对应的所有ThreadLocal变量的值。每个ThreadLocal对象都会被保存在ThreadLocalMap中，并且使用ThreadLocal作为key来访问它的变量值。这样做的好处是每个线程都可以独立维护自己的数据，而不会与其他线程产生冲突。

```java
ThreadLocalMap getMap(Thread t) {  
    return t.threadLocals;  
}

ThreadLocal.ThreadLocalMap threadLocals = null;
```
这里需要注意的是ThreadLocalMap每个元素都是Entry，而它是用弱引用对象作为key存储在ThreadLocalMap中
```java

static class Entry extends WeakReference<ThreadLocal<?>> {  
    /** The value associated with this ThreadLocal. */  
    Object value;  

    Entry(ThreadLocal<?> k, Object v) {  
        super(k);  
        value = v;  
    }  
}
```
### 实现原理

当我们通过ThreadLocal类创建一个新的变量时，实际上是在当前线程的threadLocals变量中创建了一个新的Entry对象，该对象的key是ThreadLocal对象本身，value则是我们设置的变量值。这个Entry对象会存储在ThreadLocalMap中。

当我们需要在当前线程中访问这个变量时，ThreadLocal会根据当前线程获取对应的ThreadLocalMap对象，并根据ThreadLocal对象作为key来查找该变量的值。由于每个线程都有自己独立的ThreadLocalMap对象，因此不同线程之间的变量互不干扰。

## ThreadLocal内存泄漏原因


每个ThreadLocal对象都会被存储在当前线程的ThreadLocalMap中，并且使用ThreadLocal对象作为key来访问它的变量值。由于使用的是弱引用对象作为key，当一个ThreadLocal对象没有被任何线程引用时，该对象就会被回收。

但是，即使ThreadLocal对象已经被回收，对应的变量副本仍然存在于该线程的ThreadLocalMap中。这是因为ThreadLocalMap内部使用了强引用对象（Entry对象）来引用变量副本，只有在当前线程被回收时，ThreadLocalMap中对应的Entry才会被回收。

也就是说，ThreadLocal对象虽然使用的是弱引用，但是与之关联的变量副本却是通过强引用对象间接引用的，因此在ThreadLocal对象被回收后，其变量副本可能不会立刻被回收。如果我们没有手动调用remove()方法将变量副本从ThreadLocalMap中清除，那么它就会一直存在于内存中，从而导致内存泄漏问题。

## ThreadLocal内存泄漏常见场景

ThreadLocal内存泄漏的原因主要是由于线程复用导致的。

### 线程池

当我们使用线程池时，如果在线程中使用了ThreadLocal变量，那么该变量并不会被自动清除。线程池中的线程是可以被重复利用的，如果我们在一个线程中使用了ThreadLocal变量，并且没有在该线程结束前手动清除它，那么这个变量将会一直存在于ThreadLocalMap中，即使该线程已经被回收，这就会导致内存泄漏。

### 长时间持有

如果我们在一个线程中创建了ThreadLocal变量，并且一直持有它却不使用，这也会导致内存泄漏问题。在这种情况下，由于该变量一直存在于ThreadLocalMap中，即使该线程已经被回收，该变量也无法被释放，最终会导致内存泄漏。

## ThreadLocal避免内存泄漏方法

为了避免ThreadLocal内存泄漏问题，我们可以采取以下措施：

1.  在使用完ThreadLocal变量后，应该尽快调用remove()方法将其从ThreadLocalMap中清除，以便让垃圾回收器回收它们。
2.  将ThreadLocal变量定义成private static类型的，并且在使用完之后手动清除，以避免线程重用时引起的内存泄漏问题。
3.  不要在线程池中使用ThreadLocal变量，如果必须使用，应该在使用完后手动清理。


## 总结

虽然ThreadLocal可以解决线程安全问题，但是在使用完之后需要手动清除，以避免线程重用时引起的内存泄漏问题。  

