---
title: 工厂方法模式
date: 2017-06-05 20:12:03
tags: 设计模式
categories: 设计模式
---

# 工厂方法模式

### 定义

工厂方法模式又称为工厂模式，抽象出对象创建方法，将实际创建对象的方法下沉到对应的工厂。

### 问题

> 小明船厂的业务越来越多，当初租的一块场地已经拥挤不堪，所有的业务都在岛上的划分的一块区域进行，现在需要进行业务重组，再租几块场地来进行开发，那么怎么划分比较合理呢？
<!-- more -->
### 怎么解

我们可以分为总部，小型船厂、中型船厂、大型船厂。总部接单下达生产指标分别到不同的船厂，不同的船厂造不同尺寸的船



```java
public interface ShipFactory {
    public Ship createShip();
}

public class Ship {
    public String move() {
        return "";
    }
}

/**
 * 小型船
 */
public class SmallShip extends Ship {
    public String move() {
        return "2海里/小时";
    }
}

/**
 * 中型船
 */
public class MiddleShip extends Ship {
    public String move() {
        return "10海里/小时";
    }
}

/**
 * 大型船
 */
public class BigShip extends Ship {
    public String move() {
        return "20海里/小时";
    }
}

/**
 * 小型船厂
 */
public class SmallFactory implements ShipFactory() {
    public Ship createShip() {
        return new SmallShip();
    }
}

/**
 * 中型船厂
 */
public class MiddleFactory implements ShipFactory() {
    public Ship createShip() {
        return new MiddleShip();
    }
}

/**
 * 大型船厂
 */
public class BigFactory implements ShipFactory() {
    public Ship createShip() {
        return new BigShip();
    }
}

public static void main(String [] args) {
    ShipFactory smallFactory = new SmallFactory();
  	Ship ship = smallFactory.createShip();
  	ship.move();

  	ShipFactory middleFactory = new MiddleFactory();
  	ship = middleFactory.createShip();
  	ship.move();

  	ShipFactory bigFactory = new BigFactory();
  	ship = bigFactory.createShip();
  	ship.move();
}
```



在上述代码中，我们通过不同的工厂来创建出同一类的产品（小船厂也造不出大型船），用户不需要知道我是从哪个工厂怎么造出来的，以后如果业务线再扩大时，我们可以新建另外一个工厂来解决
