---
title: java spi
date: 2019-03-12 21:11:53
tags:
    - java
categories:
    - java
---

### 概念
spi 全称（Service Provider Interface)，是JDK内置的一种服务发现机制, 不同的厂商可以通过扩展的形式来动态替换需要实现的方法

### 约定
- 在META-INF/services/目录中创建以Service接口路径命名的文本文件, 文件中包含实现接口实现类的路径，多个实现类以换行符分割
- 使用ServiceLoader.load 动态加载Service接口的实现类
- 如SPI的实现类为jar，则需要将其放在当前程序的classpath下

### 应用
在实际开发过程中我们可能会用到apacheHttp和okHttp的一些框架，我们可以让用户自主选择用一个他们比较熟悉的http框架
我们首先来创建一个spi项目
```xml
.
├── pom.xml
├── spi.iml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── com
│   │   │       └── tookbra
│   │   │           └── spi
│   │   │               ├── ApacheHttpProxy.java
│   │   │               ├── HttpProxy.java
│   │   │               ├── Main.java
│   │   │               └── OkHttpProxy.java
│   │   └── resources
│   │       └── META-INF
│   │           └── services
│   │               └── com.tookbra.spi.HttpProxy
│   └── test
│       └── java
└── target
    ├── classes
    │   ├── META-INF
    │   │   └── services
    │   │       └── com.tookbra.spi.HttpProxy
    │   └── com
    │       └── tookbra
    │           └── spi
    │               ├── ApacheHttpProxy.class
    │               ├── HttpProxy.class
    │               ├── Main.class
    │               └── OkHttpProxy.class
    └── generated-sources
        └── annotations

```

1.创建接口 com.tookbra.spi.HttpProxy
```java
package com.tookbra.spi;

/**
 * @author tookbra
 * @date 2019/1/9
 * @description
 */
public interface HttpProxy {

    void proxy();
}

```

2.创建对应的实现类
com.tookbra.spi.ApacheHttpProxy
```java
package com.tookbra.spi;

/**
 * @author tookbra
 * @date 2019/1/9
 * @description
 */
public class ApacheHttpProxy implements HttpProxy {
    public void proxy() {
        System.out.println("apache proxy");
    }
}

```

com.tookbra.spi.OkHttpProxy
```java
package com.tookbra.spi;

/**
 * @author tookbra
 * @date 2019/1/9
 * @description
 */
public class OkHttpProxy implements HttpProxy {
    public void proxy() {
        System.out.println("okhttp proxy");
    }
}
```

3.执行使用实现类的文件 META-INF/services/com.tookbra.spi.HttpProxy
```txt
com.tookbra.spi.ApacheHttpProxy
com.tookbra.spi.OkHttpProxy
```


4.运行
```java
package com.tookbra.spi;

import java.util.ServiceLoader;

/**
 * @author tookbra
 * @date 2019/1/9
 * @description
 */
public class Main {
    public static void main(String[] args) {
        ServiceLoader<HttpProxy> serviceLoader = ServiceLoader.load(HttpProxy.class);
        serviceLoader.forEach(s -> {
            s.proxy();
        });
    }
}

```

5.输出
```txt
apache proxy
okhttp proxy
```

### 解析
先看第一行代码，主要创建了服务加载对象以及初始化类加载器、加载的服务类或接口、以及内部的迭代器
```java
ServiceLoader<HttpProxy> serviceLoader = ServiceLoader.load(HttpProxy.class);

public final class ServiceLoader<S> implements Iterable<S> {
    private static final String PREFIX = "META-INF/services/";

    // 加载的服务类或接口
    private final Class<S> service;

    // 类加载器
    private final ClassLoader loader;

    // 权限控制器上下文
    private final AccessControlContext acc;

    // 以初始化的顺序缓存<接口全名称, 实现类实例>
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

    // 当前真正的迭代器
    private LazyIterator lookupIterator;

    ...
    ...

    public static <S> ServiceLoader<S> load(Class<S> service) {
        // 获取当前线程上下文的类加载器
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }


    /**
     * 创建一个新的服务加载对象
     * @param service 服务接口或抽象类
     * @param loader 类加载器
     */
    public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader){
        return new ServiceLoader<>(service, loader);
    }


    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        // 判断加载的服务类或接口是否为null
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        // 判断类加载器是否为空 如果为空 则获取启动应用程序的类加载器
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        // 初始化权限控制器上下文
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        // 刷新
        reload();
    }

     public void reload() {
        // 清空还款
        providers.clear();
        // 创建迭代器
        lookupIterator = new LazyIterator(service, loader);
    }

    ...
}
```

ServiceLoader实现了Iterable接口的iterator方法，会先从缓存map去找，没有数据再从内部lookupIterator找
```java
public Iterator<S> iterator() {
    return new Iterator<S>() {

        Iterator<Map.Entry<String,S>> knownProviders = providers.entrySet().iterator();

        public boolean hasNext() {
            // 判断缓存中有没有数据
            if (knownProviders.hasNext())
                return true;
            // 如果没有数据 判断内部迭代器中有没数据
            return lookupIterator.hasNext();
        }

        public S next() {
            // 判断缓存中有没有数据
            if (knownProviders.hasNext())
                return knownProviders.next().getValue();
            // 如果没有数据，从内部迭代器中找
            return lookupIterator.next();
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    };
}

private class LazyIterator implements Iterator<S> {

    // 加载的服务类或接口
    Class<S> service;
    // 类加载器
    ClassLoader loader;
    //
    Enumeration<URL> configs = null;
    //
    Iterator<String> pending = null;
    // 服务类路径
    String nextName = null;

    private LazyIterator(Class<S> service, ClassLoader loader) {
        this.service = service;
        this.loader = loader;
    }

    /**
     * 判断是否有接口服务
     * 首先判断下一个接口名是否存在 如果存在则返回true
     * 加载META-INF/services/接口路径 下的文件并判断是否有下一个元素并储存到pending链表中，如果有则赋值给nextName
     */
    private boolean hasNextService() {
        if (nextName != null) {
            return true;
        }
        if (configs == null) {
            try {
                String fullName = PREFIX + service.getName();
                if (loader == null)
                    configs = ClassLoader.getSystemResources(fullName);
                else
                    configs = loader.getResources(fullName);
            } catch (IOException x) {
                fail(service, "Error locating configuration files", x);
            }
        }
        while ((pending == null) || !pending.hasNext()) {
            if (!configs.hasMoreElements()) {
                return false;
            }
            pending = parse(service, configs.nextElement());
        }
        nextName = pending.next();
        return true;
    }

    /**
     * 首先判断是否有接口服务，如果没有则直接抛出异常
     * 其次创建该类的实例并转化为对应的接口类型
     * 最后存储在provider缓存中并返回转型后的实现类实例
     */
    private S nextService() {
        if (!hasNextService())
            throw new NoSuchElementException();
        String cn = nextName;
        nextName = null;
        Class<?> c = null;
        try {
            c = Class.forName(cn, false, loader);
        } catch (ClassNotFoundException x) {
            fail(service,
                 "Provider " + cn + " not found");
        }
        if (!service.isAssignableFrom(c)) {
            fail(service,
                 "Provider " + cn  + " not a subtype");
        }
        try {
            S p = service.cast(c.newInstance());
            providers.put(cn, p);
            return p;
        } catch (Throwable x) {
            fail(service,
                 "Provider " + cn + " could not be instantiated",
                 x);
        }
        throw new Error();          // This cannot happen
    }

    public boolean hasNext() {
        if (acc == null) {
            return hasNextService();
        } else {
            PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                public Boolean run() { return hasNextService(); }
            };
            return AccessController.doPrivileged(action, acc);
        }
    }

    public S next() {
        if (acc == null) {
            return nextService();
        } else {
            PrivilegedAction<S> action = new PrivilegedAction<S>() {
                public S run() { return nextService(); }
            };
            return AccessController.doPrivileged(action, acc);
        }
    }

    public void remove() {
        throw new UnsupportedOperationException();
    }

}
```

### 应用于
- 数据库连接
- xml处理
- 文件处理
- 日志处理
...


### 缺陷
"""
需要遍历所有的实现，并实例化，然后我们在循环中才能找到我们需要的实现。
配置文件中只是简单的列出了所有的扩展实现，而没有给他们命名。导致在程序中很难去准确的引用它们。
扩展如果依赖其他的扩展，做不到自动注入和装配
不提供类似于Spring的IOC和AOP功能
扩展很难和其他的框架集成，比如扩展里面依赖了一个Spring bean，原生的Java SPI不支持
