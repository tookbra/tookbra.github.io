---
title: Spring Cloud Eureka(一) 前身今世
date: 2017-08-24 14:20:37
tags:
    - eureka
categories:
    - eureka
---

![eureka-logo](/images/spring-cloud/2017-08-25-eureka-logo-2627.png)

[Spring Cloud Eureka]
* Spring Cloud Eureka(一) 前身今世
* [Spring Cloud Eureka(二) 配置文件](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-Properties/)
* [Spring Cloud Eureka(三) 注册中心](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-Server/)
* [Spring Cloud Eureka(四) 注册中心源码](/2017/08/25/spring-cloud/eureka/Spring-Cloud-Eureka-Server-Source/)
* [Spring Cloud Eureka(五) 客户端源码](/2017/08/25/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Source/)
* [Spring Cloud Eureka(六) 服务注册](/2017/08/29/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Register/)
* [Spring Cloud Eureka(七) 服务续约](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Heartbeat/)
* [Spring Cloud Eureka(八) 服务下线](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Cancel/)

## 什么是Eureka
> Eureka(尢里卡)是Netflix公司开源的基于REST（Representational State Transfer）的服务
> 主要用于AWS云中，用于定位服务，以实现中间层服务器的负载平衡和故障转移。我们称这个服务，尤里卡服务器。
> 尤里卡还提供了一个基于Java的客户端组件Eureka Client，它与服务的交互更容易。
> 客户端还具有内置负载平衡器，可进行基本的循环负载平衡。
> 在Netflix，一个更复杂的负载均衡器将Eureka包装，以提供基于诸如流量，资源使用，错误条件等多个因素的加权负载平衡，以提供卓越的弹性。

Spring Cloud把Netflix公司开源的Eureka整合到了Spring Cloud Netflix项目中，抽象出接口，方便使用。

<!-- more -->


## Eureka相对于zookeeper做为注册中心优势在哪
在这分(C)布(A)式(P)理论横行的年代里，我们都知道一个分布式系统不可能同时满足C(一致性)、A(可用性)和P(分区容错性)。
由于分区容错性在是分布式系统中必须要保证的，所以我们要么保证一致性，要么保证分区容错性。

**zookeeper做为注册中心:**
> 当向注册中心查询服务列表时，我们可以容忍注册中心返回的是几分钟以前的注册信息，但不能接受服务直接down掉不可用。也就是说，服务注册功能对可用性的要求要高于一致性。但是zk会出现这样一种情况，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个zk集群都是不可用的，这就导致在选举期间注册服务瘫痪。在云部署的环境下，因网络问题使得zk集群失去master节点是较大概率会发生的事，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用是不能容忍的。

**eureka做为注册中心:**
> Eureka看明白了这一点，因此在设计时就优先保证可用性。Eureka各个节点都是平等的，几个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和查询服务。而Eureka的客户端在向某个Eureka注册或时如果发现连接失败，则会自动切换至其它节点，只要有一台Eureka还在，就能保证注册服务可用(保证可用性)，只不过查到的信息可能不是最新的(不保证强一致性)。除此之外，Eureka还有一种自我保护机制，如果在15分钟内超过85%的节点都没有正常的心跳，那么Eureka就认为客户端与注册中心出现了网络故障，此时会出现以下几种情况：

> 1. Eureka不再从注册列表中移除因为长时间没收到心跳而应该过期的服务
> 2. Eureka仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上(即保证当前节点依然可用)
> 3. 当网络稳定时，当前实例新的注册信息会被同步到其它节点中

> 因此， Eureka可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像zookeeper那样使整个注册服务瘫痪。

总的来说**对于服务发现而言，可用性比数据一致性更加重要——AP胜过CP**

## 附录
https://github.com/Netflix/eureka/wiki
http://cloud.spring.io/spring-cloud-netflix/single/spring-cloud-netflix.html
