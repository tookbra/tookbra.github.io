---
title: spring schema extends2(二)
date: 2017-12-01 14:28:11
tags:
    - spring
categories:
    - spring
---

![spring-schema](/images/spring/spring-schema.png)

## 概述

接[上文](/2017/11/30/spring-schema-extends1/) 我们描述了怎么去实现spring schema扩展，本文我们来介绍通过另外一种方式也来实现[上文]()中实现的功能。

在以前开发的时候我们通常通过配置xml来告诉spring容器在应用程序中如何去实例化，配置和组装应用程序中的对象，但是xml配置不是唯一配置元数据的方式。

- [基于注释的配置](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/beans.html#beans-annotation-config) 从Spring 2.5开始引入了对基于注释的配置元数据的支持
- [基于Java的配置](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/beans.html#beans-java) 从Spring3.0开始，提供了基于Java的配置方式，我们可以使用java类来定义配置属性

在Spring boot中默认是使用java配置的方式。

我们按照以下步骤来扩展java配置

- 自定义注解
- 实现ImportBeanDefinitionRegistrar
- 实现BeanFactoryPostProcessor
- 验证结果

<!-- more -->

## 自定义注解

基于Java配置扩展关键一点就是使用`@Import`注解，Spring 3.0提供了这个注解用来支持在Configuration类中引入其它的配置类，包括`Configuration`类, `ImportSelector`和`ImportBeanDefinitionRegistrar`的实现类。

我们可以通过这个注解来引入自定义的扩展Bean。

```java
package com.tookbra.monine.config;


import com.tookbra.monine.config.spring.configuration.ProtocolBeanDefinitionRegistrar;
import org.springframework.context.annotation.Import;

import java.lang.annotation.*;

/**
 * @author tookbra
 * @date 2017/12/1
 * @description
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(ProtocolBeanDefinitionRegistrar.class)
public @interface EnableProtocol {

    String name() default "";


    int port();
}

```

在EnableProtocol注解类上，我们使用了@Import(ProtocolBeanDefinitionRegistrar.class)，在Spring容器处理`@EnableProtocold`的时候会实例化并调用`ApolloConfigRegistrar`的方法。



## 实现ImportBeanDefinitionRegistrar

`ImportBeanDefinitionRegistrar`接口定义了`registerBeanDefinitions`方法，从而允许我们向Spring注册必要的Bean。

自定义ImportBeanDefinitionRegistrar实现（`ApolloConfigRegistrar`）主要做了两件事情：

1. 描述protocol bean的结构并注册
2. 向Spring注册Bean：ProtocolProcessor

```java
package com.tookbra.monine.config.spring.configuration;

import com.tookbra.monine.config.EnableProtocol;
import com.tookbra.monine.config.Protocol;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.GenericBeanDefinition;
import org.springframework.context.annotation.ImportBeanDefinitionRegistrar;
import org.springframework.core.annotation.AnnotationAttributes;
import org.springframework.core.type.AnnotationMetadata;

/**
 * @author tookbra
 * @date 2017/12/1
 * @description
 */
public class ProtocolBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        AnnotationAttributes annotationAttributes = AnnotationAttributes.fromMap(annotationMetadata
                .getAnnotationAttributes(EnableProtocol.class.getName()));
        String name = (String) annotationAttributes.get("name");
        int port = (Integer) annotationAttributes.get("port");

        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(Protocol.class);
        beanDefinition.setScope(ConfigurableBeanFactory.SCOPE_SINGLETON);
        beanDefinition.getPropertyValues().addPropertyValue("name", name);
        beanDefinition.getPropertyValues().addPropertyValue("port", port);
        beanDefinitionRegistry.registerBeanDefinition("protocol", beanDefinition);


        BeanDefinitionBuilder bdb = BeanDefinitionBuilder.genericBeanDefinition(ProtocolProcessor.class);
        bdb.addPropertyValue("protocol", beanDefinition);
        beanDefinitionRegistry.registerBeanDefinition(ProtocolProcessor.class.getName(), bdb.getBeanDefinition());
    }
}
```



## 实现BeanFactoryPostProcessor

`BeanFactoryPostProcessor`提供的`postProcessBeanFactory`方法可以在Spring容器实例化任何bean之前对bean属性进行更改操作，我们可以配置多个`BeanFactoryPostProcessor`并设置order属性来控制执行顺序

```java
package com.tookbra.monine.config.spring.configuration;

import com.tookbra.monine.config.Protocol;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.context.EnvironmentAware;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.Environment;

import java.util.Objects;

/**
 * @author tookbra
 * @date 2017/12/1
 * @description
 */
public class ProtocolProcessor implements BeanFactoryPostProcessor, EnvironmentAware {

    private ConfigurableEnvironment environment;



    private Protocol protocol;

    /**
     * 容器初始化前调用
     * @param beanFactory
     * @throws BeansException
     */
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        if(Objects.isNull(protocol)) {
            return;
        }
        System.out.println("============>>"+ protocol.getName());
        System.out.println("============>>"+ protocol.getPort());
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = (ConfigurableEnvironment)environment;
    }


    public Protocol getProtocol() {
        return protocol;
    }

    public void setProtocol(Protocol protocol) {
        this.protocol = protocol;
    }
}
```



## BeanPostProcessor

`BeanPostProcessor`接口可以让我们在Srping容器完成实例化，配置和初始化bean之后实现一些自定义逻辑。

`BeanPostProcessor`提供了两个方法：`postProcessBeforeInitialization`和`postProcessAfterInitialization`，主要针对bean初始化提供扩展。

- `postProcessBeforeInitialization`会在每一个bean实例化之后、初始化（如afterPropertiesSet方法）之前被调用。
- `postProcessAfterInitialization`则在每一个bean初始化之后被调用。

我们常用的`@Autowired`注解就是通过`postProcessBeforeInitialization`实现的（`AutowiredAnnotationBeanPostProcessor`）。



## 验证结果

```java
package com.tookbra.monine;

import com.tookbra.monine.config.EnableProtocol;
import org.springframework.boot.SpringApplication;
import org.springframework.context.ConfigurableApplicationContext;

/**
 * @author tookbra
 * @date 2017/12/1
 * @description
 */
@EnableProtocol(name = "monine", port = 6400)
public class Main2 {

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(Main2.class, args);
        context.close();
    }
}

```



## 参考资料
[本文代码地址](https://github.com/tookbra/spring-learn/blob/master/spring-schema/src/main/java/com/tookbra/monine/Main2.java)
[基于注释的配置](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/beans.html#beans-annotation-config)
[基于Java的配置](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/beans.html#beans-java)
[扩展Spring的几种方式](http://nobodyiam.com/2017/02/26/several-ways-to-extend-spring/)
