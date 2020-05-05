---
title: 抽象工厂模式
date: 2017-06-07 22:55:21
tags: 设计模式
categories: 设计模式
---

# 抽象工厂模式

### 定义

为创建一组相关或相互依赖的对象提供一个接口，而且无需指定它们的具体类

### 问题

> 还记得小明船厂之前的业务重组吗，前几天我们把船厂分为了小、中、大3种，这是按规格来划分不同的船去哪个厂建造。但是这样职能就太单一了，那么我们又该如何去划分呢？
<!-- more -->
### 怎么解

除了按规格来划分，我们也可以按产品族来划分。小、中、大型船厂都可以生成轮船、快艇，但是因为面积限制，每个船厂生成出的船尺寸可能不太一样。

```java
public class Ship {
    public String move() {
      System.out.println("行驶");
	}
}

public class SmallSteamShip extends Ship {
    public String move() {
      System.out.println("小型轮船在行驶");
	}
}

public class SmallSpeedboat extends Ship {
    public String move() {
      System.out.println("小型快艇在行驶");
	}
}

public class MiddleSteamShip extends Ship {
    public String move() {
      System.out.println("中型轮船在行驶");
	}
}

public class MiddleSpeedboat extends Ship {
    public String move() {
      System.out.println("中型快艇在行驶");
	}
}

public interface ShipFactory {
    SteamShip createSteamShip();

  	Speedboat createSpeedboat();
}

public class SmallFactory implements ShipFactory {
    public SteamShip createSteamShip() {
        System.out.println("生产小型轮船");
      	return new SteamShip();
    }

  	public Speedboat createSpeedboat() {
        System.out.println("生产小型快艇");
      	return new Speedboat();
    }
}

public class MiddleFactory implements ShipFactory {
    public SteamShip createSteamShip() {
        System.out.println("生产中型轮船");
      	return new SteamShip();
    }

  	public Speedboat createSpeedboat() {
        System.out.println("生产中型快艇");
      	return new Speedboat();
    }
}

public static void main(String [] args) {
    ShipFactory shipFactory = new SmallFactory();
  	shipFactory.createSteamShip();
  	shipFactory.createSpeedboat();

  	shipFactory = new MiddleFactory();
  	shipFactory.createSteamShip();
  	shipFactory.createSpeedboat();
}
```

这样如果大型船厂也需要建造这2类船的话，我们不需要去更改公共的方法，只需要新建一个具体工厂去实现即可。