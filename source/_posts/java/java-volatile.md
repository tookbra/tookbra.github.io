---
title: Java并发包揭秘之volatile
date: 2017-07-11 21:33:26
tags:
    - java
    - 并发
categories:
    - java
---

##### 最近在看java.util.concurrent并发包的时候经常看到有volatile关键字来修饰的变量，那么这个关键字有什么作用呢
我们先来看下以下代码
``` java
private static boolean flag = true;

public static void main(String[] args)  {
    new Thread(() -> {
        int i = 0;
        while (flag) {
            i++;
        }
        System.out.println(Thread.currentThread().getName() + ":线程结束");
    }).start();

    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    new Thread(() -> {
        flag = false;
        System.out.println(Thread.currentThread().getName() + ":退出循环");
    }).start();
}
```
```
输出结果：Thread-1:退出循环
```
**为什么这里我修改了flag的值为false，但线程中还没跳出while循环呢?**

在[Java内存模型](http://tutorials.jenkov.com/java-concurrency/java-memory-model.html)中我们可以看到，在多线程的情况下，每个线程在jvm中都有自己的线程栈，当线程创建时，它会将主内存所有可访问变量的值复制到自己的工作内存中，线程结束后会将变量从工作内存同步回主内存中
![线程内存模型](/images/java/thread-jmm.png)

在上面的代码中定义了2个线程，在第一个线程中对flag变量进行判断，如为true则一直循环，反之结束循环，在第二个线程中修改了共享变量flag的值，因为每个线程中都是共享变量flag的拷贝，第二个线程中对flag值进行修改，第一个线程是感知不到的

**那如何让线程之间共享变量的修改在每个线程中都可以看见？**
噔噔噔～～～ volatile出现了

我们对上面的代码稍微修改下，对共享变量flag加上volatile关键字，再来看看运行结果
``` java
private static volatile boolean flag = true;

public static void main(String[] args)  {
    new Thread(() -> {
        int i = 0;
        while (flag) {
            i++;
        }
        System.out.println(Thread.currentThread().getName() + ":线程结束");
    }).start();

    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    new Thread(() -> {
        flag = false;
        System.out.println(Thread.currentThread().getName() + ":退出循环");
    }).start();
}
```
```
输出结果：
Thread-1:退出循环
Thread-0:线程结束
```
从上面列子中我们可以看到volatile修饰的变量在线程中是相互可见的。

**参考**
http://tutorials.jenkov.com/java-concurrency/volatile.html
https://en.wikipedia.org/w/index.php?title=Volatile_(computer_programming)&cm_mc_uid=14053210175014961926835&cm_mc_sid_50200000=1499876133
https://www.ibm.com/developerworks/cn/java/j-jtp06197.html
