---
layout: post
title: 解决线程池本地变量问题，TransmittableThreadLocal详解
date: 2023-02-24
tags: [java]
description: 欲买桂花同载酒，终不似，少年游
---

### ThreadLocal
> ThreadLocal又叫本地线程变量,仅供当前线程使用。但是只能用于当前线程，无法传递变量到子线程中。
````java
package com.fandf.demo.threadlocal;

/**
 * 父子线程之间传递变量
 *
 * @author fandongfeng
 * @date 2022/7/8 15:40
 */
public class ThreadLocalDemo {

    public static ThreadLocal<Integer> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) throws Exception {
        threadLocal.set(12345);
        //主线程中生成一个子线程
        Thread thread = new MyThread();
        thread.start();
        System.out.println("主线程threadLocal = " + threadLocal.get());
        threadLocal.remove();
    }

    static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("子线程threadLocal = " + threadLocal.get());
        }
    }

}
````
运行结果
```
子线程threadLocal = null
主线程threadLocal = 12345
```
### InheritableThreadLocal
> 实际工作中可能会出现 父线程创建几个子线程并发执行任务，那么父线程的本地变量如何传递到子线程呢? 答案是使用InheritableThreadLocal。
```java
package com.fandf.demo.threadlocal;

/**
 * 父子线程之间传递变量
 *
 * @author fandongfeng
 * @date 2022/7/8 15:40
 */
public class InheritableThreadLocalDemo {

    public static ThreadLocal<Integer> threadLocal = new InheritableThreadLocal<>();

    public static void main(String[] args) throws Exception {
        threadLocal.set(12345);
        //主线程中生成一个子线程
        Thread thread = new MyThread();
        thread.start();
        System.out.println("主线程threadLocal = " + threadLocal.get());
        threadLocal.remove();
    }

    static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("子线程threadLocal = " + threadLocal.get());
        }
    }

}
```
运行结果
```
子线程threadLocal = 12345
主线程threadLocal = 12345
```
#### 源码分析
```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    /**
     * 父线程创建子线程时，向子线程复制InheritableThreadLocal变量时使用
     *
     * @param parentValue the parent thread's value
     * @return the child thread's initial value
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
     * 获取Thread对象里的inheritableThreadLocals
     *
     * @param t the current thread
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * 将父线程的threadLocal变量拷贝到子线程
     
     * @param t the current thread
     * @param firstValue value for the initial entry of the table.
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```
查看 java.lang.Thread类新建线程的方法
```java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
     /** 省略  **/
    //从父线程拷贝线程本地变量到子线程
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);

}
```
查看ThreadLocal.createInheritedMap方法
```java
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}

//创建子线程时逐个读取父线程的本地变量，赋值给子线程本地变量
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}

```
#### 线程池问题
* `InheritableThreadLocal`只有在创建的时候才拷贝，只拷贝一次，然后就放到线程中的`inheritableThreadLocals`属性缓存起来。由于使用了线程池，该线程可能会存活很久甚至一直存活，那么`inheritableThreadLocals`属性将不会看到父线程的本地变量的变化.
```java
/**
 * 线程池环境下变量读取
 *
 * @author fandongfeng
 * @date 2022/7/8 16:25
 */
public class InheritableThreadLocalDemo2 {

    public static ThreadLocal<Integer> threadLocal = new InheritableThreadLocal<>();
    public static ExecutorService executorService = Executors.newFixedThreadPool(1);

    public static void main(String[] args) throws InterruptedException {
        System.out.println("主线程开启");
        threadLocal.set(1);

        executorService.submit(() -> {
            System.out.println("子线程读取本地变量：" + threadLocal.get());
        });

        TimeUnit.SECONDS.sleep(1);

        threadLocal.set(2);

        executorService.submit(() -> {
            System.out.println("子线程读取本地变量：" + threadLocal.get());
        });
    }

}
```
```
主线程开启
子线程读取本地变量：1
子线程读取本地变量：1
```
### TransmittableThreadLocal
TransmittableThreadLocal就是为了解决线程池之间ThreadLocal本地变量传递的问题。\

下面是几个典型场景例子。
-   分布式跟踪系统
-   日志收集记录系统上下文
-   Session级Cache
-   应用容器或上层框架跨应用代码给下层SDK传递信息
```java
/**
 * @author fandongfeng
 * @date 2022/7/8 16:31
 */
public class TransmittableThreadLocalDemo {

    public static ThreadLocal<Map<String, Object>> threadLocal = new TransmittableThreadLocal<>();
    public static ExecutorService executorService =
            TtlExecutors.getTtlExecutorService(Executors.newFixedThreadPool(1));

    public static void main(String[] args) throws InterruptedException {
        System.out.println("主线程开启");
        HashMap<String, Object> map = new HashMap<>(4);
        map.put("name", "fandf");
        threadLocal.set(map);
        System.out.println("主线程读取本地变量：" + threadLocal.get().get("name"));

        executorService.submit(() -> {
            System.out.println("子线程读取本地变量：" + threadLocal.get().get("name"));
        });

        TimeUnit.SECONDS.sleep(1);

        threadLocal.get().put("name", "ffff");
        System.out.println("主线程读取本地变量：" + threadLocal.get().get("name"));

        executorService.submit(() -> {
            //[读到了主线程修改后的新值]
            System.out.println("子线程读取本地变量：" + threadLocal.get().get("name"));
            threadLocal.get().put("name", "dddd");
            System.out.println("子线程读取本地变量：" + threadLocal.get().get("name"));
        });

        TimeUnit.SECONDS.sleep(1);
        System.out.println("主线程读取本地变量：" + threadLocal.get().get("name"));
    }

}
```
输出如下
```
主线程开启
主线程读取本地变量：fandf
子线程读取本地变量：fandf
主线程读取本地变量：ffff
子线程读取本地变量：ffff
子线程读取本地变量：dddd
主线程读取本地变量：dddd
```

代码地址：https://github.com/fandf/SpringCloudLearning/tree/master/fdf-demo/demo/src/main/java/com/fandf/demo/threadlocal