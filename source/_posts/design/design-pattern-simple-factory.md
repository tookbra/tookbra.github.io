---
title: 简单工厂模式
date: 2017-06-03 20:30:03
tags: 设计模式
categories: 设计模式
---



# 简单工厂模式

### 定义

简单工厂模式又称为静态工厂方法, 它通过工厂对象来创建出你所需要的对象
<!-- more -->
### 问题

> 小明在年初投资开了个小船厂，头几个月陆陆续续有订单进来，造的船都是摆渡船，经过几个月后船厂名声远播，越来越多的人来小明这边造船，并有其它船的需求。

在这里我们可以把船厂看做为工厂，摆渡船为工厂造出来的船系列之一，那么我们要造别的船该如何去去做呢？

### 怎么解

工厂对外，对我(需求方)我下了单我只关心最终结果，你们内部怎么弄是你们的事情。

工厂对内，分几个流水线，来造不同的产品，最终生成出同一个抽象的产品。

```java
/**
* 船
**/
public interface Ship {
  	//移动
    void move();
}

public class SmallShip implements Ship {
  	void move() {
        System.out.println("小三板在移动");
    }
}

public class FishingVessel implements Ship {
    void move() {
      System.out.println("渔船在移动");
	}
}

public class ShipFactory() {
  public static Ship createShip(String shipName) {
      if("小三板".equals(shipName)) {
          return new SmallShip();
      } else if("渔船".equals(shipName)) {
          return new FishingVessel();
      } else {
          return null;
      }
  }
}

public static void main(String [] args) {
    Ship smallShip = ShipFactory.createShip("小三板");
  	Ship fishingVessel = ShipFactory.createShip("渔船");
  	smallShip.move();
  	fishingVessel.move();
}
```



在这里工厂的流水线都在同一件事情就是造船，不同的是造不同的船。只要需要什么船，下单到ShipFactory工厂，工厂会根据shipName给你对应的船。用户不需要关心怎么生产，只关心拿到的对象，这样对象的创建和对象的业务之间耦合度降低。

如果再加一种船的品种，我们只需要在工厂中再加个分支判断即可，但是随着业务线的扩展，越来越多船的品种需要创建，我们可能会创建上百种分支，这样对于维护的成本或者复杂度都会上升， 不利于系统的健壮。
