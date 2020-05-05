---
title: Spring Cloud 统一配置
date: 2017-09-05 21:40:23
tags:
---

## 概述
[Spring Cloud Config](http://http://cloud.spring.io/spring-cloud-config/)是一个为分布式系统提供统一配置支持的项目。
在[Spring Cloud Config](http://http://cloud.spring.io/spring-cloud-config/)还未出来前，我们可以用[zookeeper](http://zookeeper.apache.org)来做统一配置管理中心

## Spring Cloud Config Server

怎么创建一个配置中心呢？

### 目录结构
```
.
├── config-server.iml
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── tookbra
    │   │           └── ConfigApplication.java
    │   └── resources
    │       └── application.yml
    └── test
        └── java
```

### pom.xml
``` xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
</dependencies>
```

### application.java
``` java
@EnableConfigServer
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### application.yml（git配置方式）
``` yml
server:
  port: 8000

spring:
  cloud:
    config:
      server:
        git:
          uri: https://git.oschina.net/tookbra/dht-config.git
  application:
    name: config-server
```

### application.yml（svn配置方式）
Spring Cloud Config也支持svn方式
使用svn来做配置，需要引入svnkit.jar
``` xml
<dependency>
    <groupId>org.tmatesoft.svnkit</groupId>
    <artifactId>svnkit</artifactId>
</dependency>
```

``` yml
server:
  port: 8000

spring:
  cloud:
    config:
      server:
        svn:
          uri: http://svn
      profile: subversion
  application:
    name: config-server
```
### 其它配置
Spring Cloud Config还提供了native和vault的配置中心

## 启动Config Server
运行spring-boot的maven插件
curl http://localhost:8000/test-dev.yml
``` yml
test: true
```
gateway.yml是我在[码云](http://git.oschina.net)上传的文件



## Spring Cloud Config Client

### 目录结构
```
.
├── config-client.iml
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── dht
    │   │           └── Application.java
    │   └── resources
    │       └── bootstrap.yml
    └── test
        └── java
```

### pom.xml
``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### application.java
``` java
@RestController
@SpringBootApplication
public class Application {

    @Value("${test}")
    private String zone;

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @RequestMapping("/")
    public String home() {
        return zone;
    }
}
```

### bootstrap.yml
``` yml
server:
  port: 8001

spring:
  cloud:
    config:
      uri: http://localhost:8000
      name: test

  application:
    name: test
logging:
  level: debug
```

## 启动Config Server
运行spring-boot的maven插件
curl http://localhost:8001
``` yml
true
```