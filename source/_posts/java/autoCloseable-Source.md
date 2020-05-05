---
title: jdk源码-AutoCloseable
date: 2017-06-13 09:58:00
tags:
    - java
categories:
    - java
---

![two-sum](/images/java/autocloseable.png)

jdk 1.7 新增类AutoCloseable

我们来看看源(<font color=red>蹩</font>)码(<font color=red>脚</font>)中(<font color=red>的</font>)的(<font color=red>翻</font>)注(<font color=red>译</font>)释
<!-- more -->

----

``` java
/**
 * 一个对象可能持有资源（文件或者套接字句柄）直到他们被关闭。
 * close方法在声明了try-with-resource块,对象会自动退出
 * try-with-resource块的声明确保及时的释放或避免错误的发生
 */
public interface AutoCloseable {
    /**
     * 关闭此资源
     * 此方法自动被try-with-resources调用
     *
     * 这个方法不是幂等的
     */
    void close() throws Exception;
}
```

我们来看下具体的demo
AutoCloseableBean
``` java
package com.java.io;

/**
 * Created by tookbra on 2017/6/13.
 */
public class AutoCloseableBean implements AutoCloseable {
    @Override
    public void close() throws Exception {
        System.out.println(this.getClass().getName() + ":close");

    }
}

```
测试类

``` java
package com.java.io;

/**
 * Created by tookbra on 2017/6/13.
 */
public class AutoCloseableDemo {

    public static void main(String[] args) {
        try(AutoCloseableBean autoCloseableBean = new AutoCloseableBean()) {
            System.out.println("autoCloseableBean init");
            throw new Exception("error");
        } catch (Exception e) {
            e.printStackTrace();
            System.out.println("catch exception");
        } finally {
            System.out.println("finally");
        }
    }

}
```
运行的结果
``` java
autoCloseableBean init
com.java.io.AutoCloseableBean:close
catch exception
finally
```

我们可以看到AutoCloseableBean实现了AutoCloseable的close方法,并打印close
从运行结果可以看到不管try中语句是正常的执行还是有异常，最终AutoCloseableBean会在catch、finally之前执行关闭


