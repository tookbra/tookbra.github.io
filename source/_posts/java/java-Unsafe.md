---
title: Java并发包揭秘之Unsafe
date: 2017-07-11 00:19:35
tags:
    - java
    - 并发
categories:
    - java
---

##### 最近在看java.util.concurrent并发包的时候经常看到Unsafe这个类，那么这个类有什么作用呢
我们知道java中是不能直接操作操作系统底层，而是通过native来访问，那么Unsafe这个类就帮我们去做了些底层的操作
从字面上来看unsafe是不安全的意思，我们知道java是一种安全的编程语言，防止了程序猿犯很多愚蠢的错误(比如说我)


**那么java设计者为什么要去定义这样的一个类呢**
因为Unsafe类在提升Java运行效率，增强Java语言底层操作能力方面起了很大的作用。
Unsafe类使Java拥有了像C语言的指针一样操作内存空间的能力，同时也带来了指针的问题。过度的使用Unsafe类会使得出错的几率变大，因此Java官方并不建议使用的，官方文档也几乎没有。Oracle正在计划从Java 9中去掉Unsafe类，如果真是如此影响就太大了

Unsafe的构造方法如下：
``` java
private Unsafe() {}

private static final Unsafe theUnsafe = new Unsafe();

/**
* Provides the caller with the capability of performing unsafe
* operations.
*
* <p> The returned <code>Unsafe</code> object should be carefully guarded
* by the caller, since it can be used to read and write data at arbitrary
* memory addresses.  It must never be passed to untrusted code.
*
* <p> Most methods in this class are very low-level, and correspond to a
* small number of hardware instructions (on typical machines).  Compilers
* are encouraged to optimize these methods accordingly.
*
* <p> Here is a suggested idiom for using unsafe operations:
*
* <blockquote><pre>
* class MyTrustedClass {
*   private static final Unsafe unsafe = Unsafe.getUnsafe();
*   ...
*   private long myCountAddress = ...;
*   public int getCount() { return unsafe.getByte(myCountAddress); }
* }
* </pre></blockquote>
*
* (It may assist compilers to make the local variable be
* <code>final</code>.)
*
* @exception  SecurityException  if a security manager exists and its
*             <code>checkPropertiesAccess</code> method doesn't allow
*             access to the system properties.
*/
public static Unsafe getUnsafe() {
  Class cc = sun.reflect.Reflection.getCallerClass(2);
  if (cc.getClassLoader() != null)
      throw new SecurityException("Unsafe");
  return theUnsafe;
}
```

我们可以看到Unsafe类使用了单例模式，提供了一个getUnsafe方法用于返回是实例化后的对象

我们就用这个方法来获取Unsafe对象看
``` java
public static void main(String[] args) {
    Unsafe unsafe = Unsafe.getUnsafe();
    System.out.println(unsafe.getClass());
}

控制台输出：
Exception in thread "main" java.lang.SecurityException: Unsafe
	at sun.misc.Unsafe.getUnsafe(Unsafe.java:90)
	at com.java.util.concurrent.AtomicBooleanDemo.main(AtomicBooleanDemo.java:11)
```

为什么会抛出这个异常呢，<font color=red>**注意看Class cc = sun.reflect.Reflection.getCallerClass(2);这行代码
这里定义了只有主类加载器加载的类才能调用这个方法， 否者抛出异常**</font>

**参考**
Unsafe源码：[Unsafe](http://www.docjar.com/html/api/sun/misc/Unsafe.java.html)
具体api可以参考这篇文章：[Java Magic. Part 4: sun.misc.Unsafe](http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/)
视频: [A Post-Apocalyptic sun.misc.Unsafe World - Christoph Engelbert - JOTB16](https://www.youtube.com/watch?v=FwTkFqJfW8U)