---
title: spring schema extends(一)
date: 2017-11-30 14:32:28
tags:
    - spring
categories:
    - spring
---
![spring-schema](/images/spring/spring-schema.png)

## 概述

从spring 2.0以后，spring提供了schema扩展，当spring的xml配置满足不了我们的时候，我们可以使用spring schema expand来自定义xml解析器

像我们熟知的spring-context,spring-bean,spring-tx,dubbo等框架都使用了spring schema expand

我们通过以下步骤来创建一个spring schema expand

- 创建xml描述
- 编写xsd文件，描述自定义对象
- 实现自定义NamespaceHandlerSupport
- 实现自定义BeanDefinitionParser
- 注册解析器到spring容器中
- 验证结果


<!-- more -->
## 创建xml描述

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:monine="http://www.tookbra.com/schema/monine"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                            http://www.tookbra.com/schema/monine http://www.tookbra.com/schema/monine.xsd">

    <monine:application name="provider" version="0.0.1"/>
</beans>
```

这里我们创建了一个Application对象，你可以理解成创建如下的一个spring bean

```xml
<bean id="application" class="com.tookbra.monine.config.Application">
  <property name="name" value="provider"/>
  <property name="version" value="0.0.1"/>
</bean>
```



## 创建xsd描述文件

在这个文件中我定义了xml命名空间为 "http://www.tookbra.com/schema/monine", 名称为application，类型是applicationType

```xml
<?xml version="1.0" encoding="utf-8" ?>
<xsd:schema xmlns="http://www.tookbra.com/schema/monine"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            xmlns:bean="http://www.springframework.org/schema/beans"
            targetNamespace="http://www.tookbra.com/schema/monine">

    <xsd:import namespace="http://www.springframework.org/schema/beans"/>

    <xsd:element name="application" type="applicationType">

    </xsd:element>

    <xsd:complexType name="applicationType">
        <xsd:attribute name="name" type="xsd:string" use="required">
            <xsd:annotation>
                <xsd:documentation><![CDATA[ application name. ]]></xsd:documentation>
            </xsd:annotation>
        </xsd:attribute>
        <xsd:attribute name="version" type="xsd:string" use="required">
            <xsd:annotation>
                <xsd:documentation><![CDATA[ application version. ]]></xsd:documentation>
            </xsd:annotation>
        </xsd:attribute>
    </xsd:complexType>
</xsd:schema>
```

具体xml schema格式可以查看下面2个链接

[xml schema part 1](https://www.w3.org/TR/2004/REC-xmlschema-1-20041028/)
[xml schema part 2](https://www.w3.org/TR/2004/REC-xmlschema-2-20041028/)


## 实现自定义NamespaceHandler

在这里我们注册了ApplicationBeanDefinitionParser作为application 	bean解析器

```java
package com.tookbra.monine.config.spring.schema;

import org.springframework.beans.factory.xml.NamespaceHandlerSupport;

/**
 * @author tookbra
 * @date 2017/11/30
 * @description
 */
public class MonineNamespaceHandler extends NamespaceHandlerSupport {


    public void init() {
        registerBeanDefinitionParser("application", new ApplicationBeanDefinitionParser());
    }
}

```



NamespaceHandler接口有3个方法

```java
/*
 * Copyright 2002-2012 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.beans.factory.xml;

import org.w3c.dom.Element;
import org.w3c.dom.Node;

import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.BeanDefinitionHolder;

/**
 * Base interface used by the {@link DefaultBeanDefinitionDocumentReader}
 * for handling custom namespaces in a Spring XML configuration file.
 *
 * <p>Implementations are expected to return implementations of the
 * {@link BeanDefinitionParser} interface for custom top-level tags and
 * implementations of the {@link BeanDefinitionDecorator} interface for
 * custom nested tags.
 *
 * <p>The parser will call {@link #parse} when it encounters a custom tag
 * directly under the {@code <beans>} tags and {@link #decorate} when
 * it encounters a custom tag directly under a {@code <bean>} tag.
 *
 * <p>Developers writing their own custom element extensions typically will
 * not implement this interface directly, but rather make use of the provided
 * {@link NamespaceHandlerSupport} class.
 *
 * @author Rob Harrop
 * @author Erik Wiersma
 * @since 2.0
 * @see DefaultBeanDefinitionDocumentReader
 * @see NamespaceHandlerResolver
 */
public interface NamespaceHandler {

	/**
	 * Invoked by the {@link DefaultBeanDefinitionDocumentReader} after
	 * construction but before any custom elements are parsed.
	 * @see NamespaceHandlerSupport#registerBeanDefinitionParser(String, BeanDefinitionParser)
	 */
	void init();

	/**
	 * Parse the specified {@link Element} and register any resulting
	 * {@link BeanDefinition BeanDefinitions} with the
	 * {@link org.springframework.beans.factory.support.BeanDefinitionRegistry}
	 * that is embedded in the supplied {@link ParserContext}.
	 * <p>Implementations should return the primary {@code BeanDefinition}
	 * that results from the parse phase if they wish to be used nested
	 * inside (for example) a {@code <property>} tag.
	 * <p>Implementations may return {@code null} if they will
	 * <strong>not</strong> be used in a nested scenario.
	 * @param element the element that is to be parsed into one or more {@code BeanDefinitions}
	 * @param parserContext the object encapsulating the current state of the parsing process
	 * @return the primary {@code BeanDefinition} (can be {@code null} as explained above)
	 */
	BeanDefinition parse(Element element, ParserContext parserContext);

	/**
	 * Parse the specified {@link Node} and decorate the supplied
	 * {@link BeanDefinitionHolder}, returning the decorated definition.
	 * <p>The {@link Node} may be either an {@link org.w3c.dom.Attr} or an
	 * {@link Element}, depending on whether a custom attribute or element
	 * is being parsed.
	 * <p>Implementations may choose to return a completely new definition,
	 * which will replace the original definition in the resulting
	 * {@link org.springframework.beans.factory.BeanFactory}.
	 * <p>The supplied {@link ParserContext} can be used to register any
	 * additional beans needed to support the main definition.
	 * @param source the source element or attribute that is to be parsed
	 * @param definition the current bean definition
	 * @param parserContext the object encapsulating the current state of the parsing process
	 * @return the decorated definition (to be registered in the BeanFactory),
	 * or simply the original bean definition if no decoration is required.
	 * A {@code null} value is strictly speaking invalid, but will be leniently
	 * treated like the case where the original bean definition gets returned.
	 */
	BeanDefinitionHolder decorate(Node source, BeanDefinitionHolder definition, ParserContext parserContext);

}
```

 - init
  `NamespaceHandler`被使用之前调用，完成`NamespaceHandler的初始化`
 - parse
  当遇到顶层元素时被调用
 - decorate
  当遇到一个属性或者嵌套元素的时候调用


## 实现自定义BeanDefinitionParser

在这个解析器中我们可以通过Element来访问xml元素来解析我们自定义的xml内容

在创建xsd描述文件的时候我们没有去定义id属性作为bean的标识符，那么这里我们去重写shouldGenerateId方法来生成id

```java
package com.tookbra.monine.config.spring.schema;

import com.tookbra.monine.config.Application;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.xml.AbstractSimpleBeanDefinitionParser;
import org.springframework.beans.factory.xml.ParserContext;
import org.springframework.util.StringUtils;
import org.w3c.dom.Element;

/**
 * @author tookbra
 * @date 2017/11/30
 * @description
 */
public class ApplicationBeanDefinitionParser extends AbstractSimpleBeanDefinitionParser {

    @Override
    protected Class<?> getBeanClass(Element element) {
        return Application.class;
    }

    @Override
    protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
        String name = element.getAttribute("name");
        String version = element.getAttribute("version");

        if(!StringUtils.isEmpty(name)) {
            builder.addPropertyValue("name", name);
        }
        if(!StringUtils.isEmpty(version)) {
            builder.addPropertyValue("version", version);
        }
    }

    @Override
    protected boolean shouldGenerateId() {
        return true;
    }
}

```



## 注册解析器到spring容器中

解析器，xsd描述文件都写好了，我们怎么让spring解析xml的过程中识别我们定义的元素并交给我们定义的NamespaceHandler处理呢？

我们需要在META-INF文件下创建2个文件，分别是spring.handlers，spring.schemas

- META-INF/spring.handlers
  ```properties
  http\://www.tookbra.com/schema/monine=com.tookbra.monine.config.spring.schema.MonineNamespaceHandler
  ```
  在spring.handlers文件中格式为：XML Schema Namespace=命名空间处理类
- META-INF/spring.schemas
  ```properties
  http\://www.tookbra.com/schema/monine.xsd=META-INF/monine.xsd
  ```
  在spring.handlers文件中格式为 xsi:schemaLocation=xsd描述文件路径
  `:`字符是Java属性格式中的分隔符，因此`:`URI中的 字符需要使用反斜杠进行转义



## 验证结果

```java
package com.tookbra.monine;

import com.tookbra.monine.config.Application;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * @author tookbra
 * @date 2017/11/30
 * @description
 */
public class Main {

    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("monine.xml");
        context.start();
        Application applicationConfig = (Application)context.getBean("provider");
        System.out.printf(applicationConfig.getName());

    }
}

```

不出意外的话，我们可以看到输出结果为：

```
provider
```



## 参考资料
[本文代码地址](https://github.com/tookbra/spring-learn/blob/master/spring-schema/src/main/java/com/tookbra/monine/Main.java)
[spring schema expand 1](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/xsd-configuration.html)
[spring schema expand 2](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/xml-custom.html)


  ​

  ​

  ​

  ​



  ​

  ​

  ​









