---
title: Spring Cloud Eureka(四) 注册中心源码
date: 2017-08-25 11:26:13
tags:
    - eureka
categories:
    - eureka
---

![eureka-logo](/images/spring-cloud/2017-08-25-eureka-logo-2627.png)

[Spring Cloud Eureka]
* [Spring Cloud Eureka(一) 前身今世](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-一/)
* [Spring Cloud Eureka(二) 配置文件](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-Properties/)
* [Spring Cloud Eureka(三) 注册中心](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-Server/)
* Spring Cloud Eureka(四) 注册中心源码
* [Spring Cloud Eureka(五) 客户端源码](/2017/08/25/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Source/)
* [Spring Cloud Eureka(六) 服务注册](/2017/08/29/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Register/)
* [Spring Cloud Eureka(七) 服务续约](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Heartbeat/)
* [Spring Cloud Eureka(八) 服务下线](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Cancel/)

> 本次spring-cloud-starter-eureka-server源码的版本为1.3.0

<!-- more -->

## spring-cloud-starter-eureka-server 目录结构
``` yml
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── org
    │   │       └── springframework
    │   │           └── cloud
    │   │               └── netflix
    │   │                   └── eureka
    │   │                       └── server
    │   │                           ├── CloudJacksonJson.java
    │   │                           ├── EnableEurekaServer.java
    │   │                           ├── EurekaController.java
    │   │                           ├── EurekaDashboardProperties.java
    │   │                           ├── EurekaServerAutoConfiguration.java
    │   │                           ├── EurekaServerBootstrap.java
    │   │                           ├── EurekaServerConfigBean.java
    │   │                           ├── EurekaServerInitializerConfiguration.java
    │   │                           ├── EurekaServerMarkerConfiguration.java
    │   │                           ├── InstanceRegistry.java
    │   │                           ├── InstanceRegistryProperties.java
    │   │                           └── event
    │   │                               ├── EurekaInstanceCanceledEvent.java
    │   │                               ├── EurekaInstanceRegisteredEvent.java
    │   │                               ├── EurekaInstanceRenewedEvent.java
    │   │                               ├── EurekaRegistryAvailableEvent.java
    │   │                               └── EurekaServerStartedEvent.java
    │   ├── resources
    │   │   ├── META-INF
    │   │   │   └── spring.factories
    │   │   ├── eureka
    │   │   │   └── server.properties
    │   │   ├── static
    │   │   │   └── eureka
    │   │   │       ├── fonts
    │   │   │       │   ├── montserrat-webfont.eot
    │   │   │       │   ├── montserrat-webfont.svg
    │   │   │       │   ├── montserrat-webfont.ttf
    │   │   │       │   ├── montserrat-webfont.woff
    │   │   │       │   ├── varela_round-webfont.eot
    │   │   │       │   ├── varela_round-webfont.svg
    │   │   │       │   ├── varela_round-webfont.ttf
    │   │   │       │   └── varela_round-webfont.woff
    │   │   │       └── images
    │   │   │           ├── 404-icon.png
    │   │   │           ├── homepage-bg.jpg
    │   │   │           ├── platform-bg.png
    │   │   │           ├── platform-spring-xd.png
    │   │   │           ├── spring-logo-eureka-mobile.png
    │   │   │           └── spring-logo-eureka.png
    │   │   └── templates
    │   │       └── eureka
    │   │           ├── header.ftl
    │   │           ├── lastn.ftl
    │   │           ├── navbar.ftl
    │   │           └── status.ftl
    │   └── wro
    │       ├── header.less
    │       ├── main.less
    │       ├── responsive.less
    │       ├── typography.less
    │       ├── wro.properties
    │       └── wro.xml
    └── test
        ├── java
        │   └── org
        │       └── springframework
        │           └── cloud
        │               └── netflix
        │                   └── eureka
        │                       └── server
        │                           ├── ApplicationContextTests.java
        │                           ├── ApplicationDashboardDisabledTests.java
        │                           ├── ApplicationDashboardPathTests.java
        │                           ├── ApplicationServletPathTests.java
        │                           ├── ApplicationTests.java
        │                           ├── EurekaControllerReplicasTests.java
        │                           ├── EurekaControllerTests.java
        │                           ├── EurekaCustomPeerNodesTests.java
        │                           └── InstanceRegistryTests.java
        └── resources
            └── application.properties
```

## spring-boot-starter-eureka-server启动分析
在META-INF目录下有个spring.factories文件
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```

spring-boot在启动过程中通过加载、解析spring.factories文件中的信息，对一些bean进行自动装配，在这里我们就不关注了[spring-boot 自动配置]()


我们可以看到EurekaServerAutoConfiguration类由spring ioc来装配, 加载了EurekaServerInitializerConfiguration类
并且把eureka dashboard、register等配置属性注入到EurekaDashboardProperties和InstanceRegistryProperties类中
``` java
@Configuration
@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
		InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
    ......
}

```


EurekaServerInitializerConfiguration 实现了SmartLifecycle接口的start方法，会在所有spring bean都初始化完成后调用该方法
``` java
@Configuration
@CommonsLog
public class EurekaServerInitializerConfiguration
		implements ServletContextAware, SmartLifecycle, Ordered {

	...

	@Override
	public void start() {
		new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					//TODO: is this class even needed now?
					// 初始化eureka server
					eurekaServerBootstrap.contextInitialized(EurekaServerInitializerConfiguration.this.servletContext);
					log.info("Started Eureka Server");

                    // 传递eureka注册事件，找来找去没看到有listen去处理这件事件
                    // spring包中没有去处理，如果我们有需要可以自己处理
                    // https://github.com/spring-cloud/spring-cloud-netflix/issues/1726
					publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
					// 设置在spring容器关闭的时候执行stop方法
					EurekaServerInitializerConfiguration.this.running = true;
					publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
				}
				catch (Exception ex) {
					// Help!
					log.error("Could not initialize Eureka servlet context", ex);
				}
			}
		}).start();
	}


    @Bean
    public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
            EurekaServerContext serverContext) {
        return new EurekaServerBootstrap(this.applicationInfoManager,
                this.eurekaClientConfig, this.eurekaServerConfig, registry,
                serverContext);
    }
	...
}
```

Eureka Config wiki: https://github.com/Netflix/eureka/wiki/Configuring-Eureka

``` java
@CommonsLog
public class EurekaServerBootstrap {

    ....

    public void contextInitialized(ServletContext context) {
        try {
            //初始化eureka环境
            initEurekaEnvironment();
            //初始化eureka上下文
            initEurekaServerContext();

            context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
        }
        catch (Throwable e) {
            log.error("Cannot bootstrap eureka server :", e);
            throw new RuntimeException("Cannot bootstrap eureka server :", e);
        }
    }

    protected void initEurekaEnvironment() throws Exception {
        log.info("Setting the eureka configuration..");

        //如果在云环境中运行，需要传递java命令行属性-Deureka.datacenter = cloud，以便Eureka客户端/服务器知道初始化AWS云特定的信息
        String dataCenter = ConfigurationManager.getConfigInstance()
                .getString(EUREKA_DATACENTER);
        if (dataCenter == null) {
            log.info(
                    "Eureka data center value eureka.datacenter is not set, defaulting to default");
            ConfigurationManager.getConfigInstance()
                    .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
        }
        else {
            ConfigurationManager.getConfigInstance()
                    .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
        }

        //eureka 运行环境，java命令行属性-Deureka.environment=eureka-client-{test，prod}
        String environment = ConfigurationManager.getConfigInstance()
                .getString(EUREKA_ENVIRONMENT);
        if (environment == null) {
            ConfigurationManager.getConfigInstance()
                    .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
            log.info(
                    "Eureka environment value eureka.environment is not set, defaulting to test");
        }
        else {
            ConfigurationManager.getConfigInstance()
                    .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, environment);
        }
    }

    // eureka使用XStream来对json、xml格式的转换
    protected void initEurekaServerContext() throws Exception {
        // 向后兼容
        //注册状态转换器
        JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
                XStream.PRIORITY_VERY_HIGH);
        XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
                XStream.PRIORITY_VERY_HIGH);

        //判断是否亚马逊的数据中心
        if (isAws(this.applicationInfoManager.getInfo())) {
            this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
                    this.eurekaClientConfig, this.registry, this.applicationInfoManager);
            //
            this.awsBinder.start();
        }

        //初始化eureka server上下文
        EurekaServerContextHolder.initialize(this.serverContext);

        log.info("Initialized server context");

        // 从相邻的eureka节点复制注册表
        int registryCount = this.registry.syncUp();
        // 默认每30秒发送心跳，1分钟就是2次
        // 修改eureka状态为up
        this.registry.openForTraffic(this.applicationInfoManager, registryCount);

        // 注册所有监控
        EurekaMonitors.registerAllStats();
    }

    ...
}
```

``` java
    @Override
    // 同步注册表
    public int syncUp() {
        // Copy entire entry from neighboring DS node
        int count = 0;

        // 需要设置 eureka.client.registerWithEureka=true
        // 默认5次, 可以通过eureka.server.registry-sync-retries属性来设置
        for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
            if (i > 0) {
                try {
                    Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
                } catch (InterruptedException e) {
                    logger.warn("Interrupted during registry transfer..");
                    break;
                }
            }
            Applications apps = eurekaClient.getApplications();
            for (Application app : apps.getRegisteredApplications()) {
                for (InstanceInfo instance : app.getInstances()) {
                    try {
                        if (isRegisterable(instance)) {
                            register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                            count++;
                        }
                    } catch (Throwable t) {
                        logger.error("During DS init copy", t);
                    }
                }
            }
        }
        return count;
    }
```

## 事件处理
Spring Cloud Netflix Eureka Server中提供了5个事件通知类
* EurekaInstanceCanceledEvent 服务下线事件
* EurekaInstanceRegisteredEvent 服务注册事件
* EurekaInstanceRenewedEvent 服务续约事件
* EurekaRegistryAvailableEvent Eureka注册中心启动事件
* EurekaServerStartedEvent Eureka Server启动事件

我可以实现listen去处理这些动作




