---
title: Spring Cloud Eureka(三) 注册中心
date: 2017-08-24 22:18:00
tags:
    - eureka
categories:
    - eureka
---

![eureka-logo](/images/spring-cloud/2017-08-25-eureka-logo-2627.png)

[Spring Cloud Eureka]
* [Spring Cloud Eureka(一) 前身今世](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-一/)
* [Spring Cloud Eureka(二) 配置文件](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-Properties/)
* Spring Cloud Eureka(三) 注册中心
* [Spring Cloud Eureka(四) 注册中心源码](/2017/08/25/spring-cloud/eureka/Spring-Cloud-Eureka-Server-Source/)
* [Spring Cloud Eureka(五) 客户端源码](/2017/08/25/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Source/)
* [Spring Cloud Eureka(六) 服务注册](/2017/08/29/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Register/)
* [Spring Cloud Eureka(七) 服务续约](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Heartbeat/)
* [Spring Cloud Eureka(八) 服务下线](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Cancel/)

<!-- more -->

## eureka-server项目

### 目录结构
```
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── com
│   │   │       └── tookbra
│   │   │           └── eureka
│   │   │               └── Application.java
│   │   └── resources
│   │       └── application.yml
```
### pom.xml
``` xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>
</dependencies>
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

### application.java
``` java
@EnableEurekaServer
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### application.yml
``` yml
spring:
  application:
    name: eureka

server:
  port: 10005

eureka:
  client:
    # 是否向 Eureka 注册服务
    registerWithEureka: false
    # 是否检索服务
    fetchRegistry: false
```

## 启动eureka注册中心
运行spring-boot的maven插件
在游览器中打开http://localhost:10005/
![注册中心](/images/spring-cloud/2017-08-25-Eureka-register.png)
我们可以看到注册中心的页面上涵盖了:
1. 系统状态(环境、系统时间、续租时间等)
2. 服务注册信息(服务名称、服务状态、服务地址)
3. 基础信息(cpu、内存)
4. 注册中心实例信息（ip地址、状态）


## eureka 高可用
![eureka_architecture](/images/spring-cloud/2017-08-31-eureka_architecture.png)

多个eureka服务相互注册，来保证服务高可用
``` yml
---
spring:
  profiles: node1
server:
  port: 10001

eureka:
  instance:
    hostname: node1
    instanceId: node1
#    non-secure-port: 80
  client:
    serviceUrl:
      defaultZone: http://node2:10002/eureka,http://node3:10003/eureka
debug: true

---
spring:
  profiles: node2
server:
  port: 10002

eureka:
  instance:
    hostname: node2
    instanceId: node2
  client:
    serviceUrl:
      defaultZone: http://node1:10001/eureka,http://node3:10003/eureka

---
spring:
  profiles: node3
server:
  port: 10003

eureka:
  instance:
    hostname: node3
    instanceId: node3
  client:
    serviceUrl:
      defaultZone: http://node1:10001/eureka/,http://node2:10002/eureka/
```


## eureka信息安全
在生产环境中，我们通常不会吧eureka暴露在外网
但是在开发环境中，我们可能会把eureka暴露在外网上，那么eureka的信息安全如何保证？
我们可以引入spring security，来做请求认证
``` xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-security</artifactId>
</dependency>
```

``` yml
security:
  basic:
    enabled: true
  user:
    name: user
    password: 123456
```



如果加了请求认证，需要把eureka.client.serviceUrl.defaultZone也加上认证头
``` xml
eureka.client.serviceUrl.defaultZone=http://user:password@node2:10002/eureka
```
