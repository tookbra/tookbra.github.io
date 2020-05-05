---
title: spring-boot启动分析(一)
date: 2017-06-14 16:30:56
tags:
    - spring boot
---
![spring-boot](/images/spring-boot/spring-boot.png)

我们知道spring boot的出现简化了spring应用的搭建和开发过程，不需要去配置大量的xml以及web.xml就可以运行应用
那spring boot 是怎么通过不配置xml和web.xml 来启动应用呢？

<!-- more -->

我们先来看看spring boot提供的demo中是通过main函数去启动应用的
``` java
package hello;

import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.stereotype.*;
import org.springframework.web.bind.annotation.*;

@Controller
@EnableAutoConfiguration
public class SampleController {

    @RequestMapping("/")
    @ResponseBody
    String home() {
        return "Hello World!";
    }

    public static void main(String[] args) throws Exception {
        SpringApplication.run(SampleController.class, args);
    }
}
```
我们来看看SpringApplication中的run方法
``` java
/**
 * @param source 主要加载对象
 * @param args 应用参数
 * @return 返回ConfigurableApplicationContext对象
 */
public static ConfigurableApplicationContext run(Object source, String... args) {
    return run(new Object[]{source}, args);
}

/**
  * @param sources 主要加载对象
  * @param args 应用参数
  * @return 返回ConfigurableApplicationContext对象
  */
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
    //在这里声明了SpringApplication对象
    return (new SpringApplication(sources)).run(args);
}
```

``` java
public SpringApplication(Object... sources) {
    //创建spring应用实例
    initialize(sources);
}

private void initialize(Object[] sources) {
    if (sources != null && sources.length > 0) {
        this.sources.addAll(Arrays.asList(sources));
    }
    //判断是否是web环境
    this.webEnvironment = deduceWebEnvironment();
    /**
     * 初始化 ApplicationContextInitializer
     *
     * 加载spring-boot、spring-boot-autoconfigure中META-INF文件夹下的spring.factories文件
     * org.springframework.boot.context.config.DelegatingApplicationContextInitializer
     * org.springframework.boot.context.ContextIdApplicationContextInitializer
     * org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer
     * org.springframework.boot.context.embedded.ServerPortInfoApplicationContextInitializer
     * org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer
     * org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer
     * 创建这些对象并放入集合中进行排序
     *
     * 同理加载spring-boot、spring-boot-autoconfigure中META-INF文件夹下的spring.factories文件
     * org.springframework.boot.context.config.ConfigFileApplicationListener
     * org.springframework.boot.context.config.AnsiOutputApplicationListener
     * org.springframework.boot.logging.LoggingApplicationListener
     * org.springframework.boot.logging.ClasspathLoggingApplicationListener
     * org.springframework.boot.autoconfigure.BackgroundPreinitializer
     * org.springframework.boot.context.config.DelegatingApplicationListener
     * org.springframework.boot.builder.ParentContextCloserApplicationListener
     * org.springframework.boot.ClearCachesApplicationListener
     * org.springframework.boot.context.FileEncodingApplicationListener
     * org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
     * 创建这些对象并放入集合中进行排序
     */
    setInitializers((Collection) getSpringFactoriesInstances(
            ApplicationContextInitializer.class));
    //设置监听
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //设置启动类 也就是SpringApplication.run所在的类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

``` java
public ConfigurableApplicationContext run(String... args) {
    //通过StopWatch来判断是否在运行
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    FailureAnalyzers analyzers = null;
    //
    this.configureHeadlessProperty();
    //创建SpringApplicationRunListeners对象 还是spring.factories中的类路径
    SpringApplicationRunListeners listeners = getRunListeners(args);
    //启动监听，如果有多个监听则启动多个
    listeners.starting();

    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        //一些环境变量
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        //注意看这里，打印banner，修改些配置就可以定义自己的启动图
        Banner printedBanner = printBanner(environment);
        //创建ConfigurableApplicationContext 判断当前是否web环境 如果是用AnnotationConfigEmbeddedWebApplicationContext否则用AnnotationConfigApplicationContext 强转成ConfigurableApplicationContext
        context = createApplicationContext();
        //创建FailureAnalyzers对象，还是spring.factories中的类路径
        analyzers = new FailureAnalyzers(context);
        //
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        //
        refreshContext(context);
        //回调ApplicationRunner和CommandLineRunner的run方法
        afterRefresh(context, applicationArguments);
        //通知应用上下文可以使用了
        listeners.finished(context, (Throwable)null);
        //停止计数器
        stopWatch.stop();
        //创建StartupInfoLogger对象
        if(this.logStartupInfo) {
            (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
        }

        return context;
    } catch (Throwable var9) {
        this.handleRunFailure(context, listeners, (FailureAnalyzers)analyzers, var9);
        throw new IllegalStateException(var9);
    }
}
```
