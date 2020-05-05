---
title: Java并发包揭秘之AtomicBoolean
date: 2017-07-14 00:16:25
tags:
    - java
    - 并发
categories:
    - java
---


``` java
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicBoolean.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

private volatile int value;
```

我们可以看到在AtomicBoolean在类加载的时候，通过unsafe的objectFieldOffset方法来获取valueOffset相对于AtomicBoolean实例对象的其实位置偏移量

**那么AtomicBoolean是如何在多线程中保证value的可见性和原子性呢**

``` java
/**
 * Atomically sets the value to the given updated value
 * if the current value {@code ==} the expected value.
 *
 * @param expect the expected value
 * @param update the new value
 * @return {@code true} if successful. False return indicates that
 * the actual value was not equal to the expected value.
 */
public final boolean compareAndSet(boolean expect, boolean update) {
    int e = expect ? 1 : 0;
    int u = update ? 1 : 0;
    return unsafe.compareAndSwapInt(this, valueOffset, e, u);
}
```

> 我们可以看到compareAndSwapInt其实就是CAS（比较与交换，Compare and swap）无锁算法
当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试

compareAndSet方法有2个参数，一个是期望值(expected)，一个是更新值(update)；
如果value的值和期望值一致，则更新值设定给value，返回true，表明成功；否则就不设定，并返回false。

**示例**
``` java
public static void main(String[] args) {
    AtomicBoolean atomicBoolean = new AtomicBoolean(false);

    new Thread(new Runnable() {
        @Override
        public void run() {
            atomicBoolean.compareAndSet(true, true);
            System.out.println(atomicBoolean.get());
        }
    }).start();

    new Thread(new Runnable() {
        @Override
        public void run() {
            atomicBoolean.compareAndSet(false, true);
            System.out.println(atomicBoolean.get());
        }
    }).start();
}
```

**参考**
https://en.wikipedia.org/wiki/Compare-and-swap