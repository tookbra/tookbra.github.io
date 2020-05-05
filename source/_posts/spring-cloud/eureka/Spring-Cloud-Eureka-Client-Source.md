---
title: Spring Cloud Eureka(五) 客户端源码
date: 2017-08-25 14:44:58
tags:
    - eureka
categories:
    - eureka
---

![eureka-logo](/images/spring-cloud/2017-08-25-eureka-logo-2627.png)

* [Spring Cloud Eureka(一) 前身今世](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-一/)
* [Spring Cloud Eureka(二) 配置文件](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-Properties/)
* [Spring Cloud Eureka(三) 注册中心](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-Server/)
* [Spring Cloud Eureka(四) 注册中心源码](/2017/08/25/spring-cloud/eureka/Spring-Cloud-Eureka-Server-Source/)
* Spring Cloud Eureka(五) 客户端源码
* [Spring Cloud Eureka(六) 服务注册](/2017/08/29/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Register/)
* [Spring Cloud Eureka(七) 服务续约](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Heartbeat/)
* [Spring Cloud Eureka(八) 服务下线](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Cancel/)

> 本次spring-cloud-starter-eureka源码的版本为1.3.0

<!-- more -->

## 源码目录结构
``` yml
.
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── org
    │   │       └── springframework
    │   │           └── cloud
    │   │               └── netflix
    │   │                   ├── eureka
    │   │                   │   ├── CloudEurekaClient.java
    │   │                   │   ├── CloudEurekaInstanceConfig.java
    │   │                   │   ├── CloudEurekaTransportConfig.java
    │   │                   │   ├── EnableEurekaClient.java
    │   │                   │   ├── EurekaClientAutoConfiguration.java
    │   │                   │   ├── EurekaClientConfigBean.java
    │   │                   │   ├── EurekaConstants.java
    │   │                   │   ├── EurekaDiscoveryClient.java
    │   │                   │   ├── EurekaDiscoveryClientConfiguration.java
    │   │                   │   ├── EurekaHealthCheckHandler.java
    │   │                   │   ├── EurekaHealthIndicator.java
    │   │                   │   ├── EurekaInstanceConfigBean.java
    │   │                   │   ├── InstanceInfoFactory.java
    │   │                   │   ├── MutableDiscoveryClientOptionalArgs.java
    │   │                   │   ├── config
    │   │                   │   │   ├── DiscoveryClientOptionalArgsConfiguration.java
    │   │                   │   │   ├── EurekaClientConfigServerAutoConfiguration.java
    │   │                   │   │   ├── EurekaDiscoveryClientConfigServiceAutoConfiguration.java
    │   │                   │   │   └── EurekaDiscoveryClientConfigServiceBootstrapConfiguration.java
    │   │                   │   ├── http
    │   │                   │   │   ├── EurekaApplications.java
    │   │                   │   │   ├── RestTemplateDiscoveryClientOptionalArgs.java
    │   │                   │   │   ├── RestTemplateEurekaHttpClient.java
    │   │                   │   │   ├── RestTemplateTransportClientFactories.java
    │   │                   │   │   └── RestTemplateTransportClientFactory.java
    │   │                   │   └── serviceregistry
    │   │                   │       ├── EurekaAutoServiceRegistration.java
    │   │                   │       ├── EurekaRegistration.java
    │   │                   │       └── EurekaServiceRegistry.java
    │   │                   └── ribbon
    │   │                       └── eureka
    │   │                           ├── DomainExtractingServerList.java
    │   │                           ├── EurekaRibbonClientConfiguration.java
    │   │                           ├── EurekaServerIntrospector.java
    │   │                           ├── RibbonEurekaAutoConfiguration.java
    │   │                           └── ZoneUtils.java
    │   └── resources
    │       └── META-INF
    │           └── spring.factories
    └── test
        ├── java
        │   └── org
        │       └── springframework
        │           └── cloud
        │               └── netflix
        │                   ├── eureka
        │                   │   ├── ConditionalOnRefreshScopeTests.java
        │                   │   ├── EurekaClientAutoConfigurationTests.java
        │                   │   ├── EurekaClientConfigBeanTests.java
        │                   │   ├── EurekaHealthCheckHandlerTests.java
        │                   │   ├── EurekaInstanceConfigBeanTests.java
        │                   │   ├── InstanceInfoFactoryTests.java
        │                   │   ├── config
        │                   │   │   ├── ConfigRefreshTests.java
        │                   │   │   ├── DiscoveryClientConfigServiceAutoConfigurationTests.java
        │                   │   │   ├── EurekaClientConfigServerAutoConfigurationTests.java
        │                   │   │   ├── JerseyOptionalArgsConfigurationTest.java
        │                   │   │   └── RestTemplateOptionalArgsConfigurationTest.java
        │                   │   ├── healthcheck
        │                   │   │   └── EurekaHealthCheckTests.java
        │                   │   ├── http
        │                   │   │   ├── EurekaServerMockApplication.java
        │                   │   │   ├── RestTemplateEurekaHttpClientTest.java
        │                   │   │   ├── RestTemplateTransportClientFactoriesTest.java
        │                   │   │   └── RestTemplateTransportClientFactoryTest.java
        │                   │   ├── sample
        │                   │   │   ├── ApplicationTests.java
        │                   │   │   ├── EurekaSampleApplication.java
        │                   │   │   └── RefreshEurekaSampleApplication.java
        │                   │   └── serviceregistry
        │                   │       └── EurekaServiceRegistryTests.java
        │                   └── ribbon
        │                       └── eureka
        │                           ├── DomainExtractingServerListTests.java
        │                           ├── EurekaDisabledRibbonClientIntegrationTests.java
        │                           ├── EurekaRibbonClientConfigurationTests.java
        │                           ├── EurekaRibbonClientPreprocessorIntegrationTests.java
        │                           ├── EurekaRibbonClientPropertyOverrideIntegrationTests.java
        │                           ├── RibbonClientPreprocessorIntegrationTests.java
        │                           ├── RibbonEurekaAutoConfigurationTests.java
        │                           └── ZoneUtilsTests.java
        └── resources
            └── application.yml
```

## 客户端启动分析

``` java
@SpringBootApplication
@EnableDiscoveryClient
public class ClientApplication {
	public static void main(String[] args) {
		SpringApplication.run(ClientApplication.class, args);
	}
}
```

EnableDiscoveryClient类
``` java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import({EnableDiscoveryClientImportSelector.class})
public @interface EnableDiscoveryClient {
    boolean autoRegister() default true;
}
```

``` java
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableDiscoveryClientImportSelector
		extends SpringFactoryImportSelector<EnableDiscoveryClient> {

	@Override
	public String[] selectImports(AnnotationMetadata metadata) {
		String[] imports = super.selectImports(metadata);

		AnnotationAttributes attributes = AnnotationAttributes.fromMap(
				metadata.getAnnotationAttributes(getAnnotationClass().getName(), true));

		boolean autoRegister = attributes.getBoolean("autoRegister");

		if (autoRegister) {
			List<String> importsList = new ArrayList<>(Arrays.asList(imports));
			importsList.add("org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
			imports = importsList.toArray(new String[0]);
		}

		return imports;
	}

	...

}
```
eureka客户端只要在启动类中加上@EnableDiscoveryClient注解，便表示这是一个eureka客户端
在EnableDiscoveryClient类中，可以看到引入了EnableDiscoveryClientImportSelector类
在super.selectImports方法中加载了META-INF目录下有spring.factories文件中的org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration

``` yml
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration

org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceBootstrapConfiguration

org.springframework.cloud.client.discovery.EnableDiscoveryClient=\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration
```


``` java
@Order(2147483547)
public class EnableDiscoveryClientImportSelector extends SpringFactoryImportSelector<EnableDiscoveryClient> {
    public EnableDiscoveryClientImportSelector() {
    }

    //返回需要加载的EurekaDiscoveryClientConfiguration bean
    public String[] selectImports(AnnotationMetadata metadata) {
        //org.springframework.cloud.client.discovery.EnableDiscoveryClient=\org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration
        String[] imports = super.selectImports(metadata);
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(this.getAnnotationClass().getName(), true));
        boolean autoRegister = attributes.getBoolean("autoRegister");
        if(autoRegister) {
            List<String> importsList = new ArrayList(Arrays.asList(imports));
            importsList.add("org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
            imports = (String[])importsList.toArray(new String[0]);
        }

        return imports;
    }

    protected boolean isEnabled() {
        return ((Boolean)(new RelaxedPropertyResolver(this.getEnvironment())).getProperty("spring.cloud.discovery.enabled", Boolean.class, Boolean.TRUE)).booleanValue();
    }

    protected boolean hasDefaultFactory() {
        return true;
    }
}
```

spring-boot在启动过程中通过加载、解析spring.factories文件中的信息，对一些bean进行自动装配，在这里我们就不关注了[spring-boot 自动配置]()
在EurekaServerAutoConfiguration类中

``` java
@Configuration
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@ConditionalOnBean(EurekaDiscoveryClientConfiguration.Marker.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@AutoConfigureBefore({ NoopDiscoveryClientAutoConfiguration.class,
		CommonsClientAutoConfiguration.class, ServiceRegistryAutoConfiguration.class })
@AutoConfigureAfter(name = "org.springframework.cloud.autoconfigure.RefreshAutoConfiguration")
public class EurekaClientAutoConfiguration {
   ...

   @Bean
   	public DiscoveryClient discoveryClient(EurekaInstanceConfig config,
   			EurekaClient client) {
   		return new EurekaDiscoveryClient(config, client);
   	}

   	@Configuration
    @ConditionalOnRefreshScope
    protected static class RefreshableEurekaClientConfiguration {

        @Autowired
        private ApplicationContext context;

        @Autowired(required = false)
        private DiscoveryClientOptionalArgs optionalArgs;

        @Bean(destroyMethod = "shutdown")
        @ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
        @org.springframework.cloud.context.config.annotation.RefreshScope
        @Lazy
        public EurekaClient eurekaClient(ApplicationInfoManager manager,
                EurekaClientConfig config, EurekaInstanceConfig instance) {
            manager.getInfo(); // force initialization
            return new CloudEurekaClient(manager, config, this.optionalArgs,
                    this.context);
        }

        @Bean
        @ConditionalOnMissingBean(value = ApplicationInfoManager.class, search = SearchStrategy.CURRENT)
        @org.springframework.cloud.context.config.annotation.RefreshScope
        @Lazy
        public ApplicationInfoManager eurekaApplicationInfoManager(
                EurekaInstanceConfig config) {
            InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
            return new ApplicationInfoManager(config, instanceInfo);
        }

    }

    @Bean
    public DiscoveryClient discoveryClient(EurekaInstanceConfig config,
            EurekaClient client) {
        return new EurekaDiscoveryClient(config, client);
    }

   ...
}
```

![CloudEurekaClient](/images/spring-cloud/2017-08-29-CloudEurekaClient.png)
spring cloud使用CloudEurekaClient对eureka的DiscoveryClient初始化

EurekaAutoServiceRegistration类实现了SmartLifecycle接口的start方法，会在所有spring bean都初始化完成后调用该方法
``` java
public class EurekaAutoServiceRegistration implements AutoServiceRegistration, SmartLifecycle, Ordered {
    ...

    @Override
	public void start() {
		// only set the port if the nonSecurePort is 0 and this.port != 0
		if (this.port.get() != 0 && this.registration.getNonSecurePort() == 0) {
			this.registration.setNonSecurePort(this.port.get());
		}

		// only initialize if nonSecurePort is greater than 0 and it isn't already running
		// because of containerPortInitializer below
		if (!this.running.get() && this.registration.getNonSecurePort() > 0) {
            // 设置 eureka 初始化状态 eureka.instance.initial-status
			this.serviceRegistry.register(this.registration);
            // 传递eureka注册事件
            // DiscoveryClientHealthIndicator类监听
			this.context.publishEvent(
					new InstanceRegisteredEvent<>(this, this.registration.getInstanceConfig()));
			// 设置在spring容器关闭的时候执行stop方法
			this.running.set(true);
		}
	}

	....
}
```

``` java
public class EurekaServiceRegistry implements ServiceRegistry<EurekaRegistration> {
    public void register(EurekaRegistration reg) {
		maybeInitializeClient(reg);

		if (log.isInfoEnabled()) {
			log.info("Registering application " + reg.getInstanceConfig().getAppname()
					+ " with eureka with status "
					+ reg.getInstanceConfig().getInitialStatus());
		}

		reg.getApplicationInfoManager()
				.setInstanceStatus(reg.getInstanceConfig().getInitialStatus());

		if (reg.getHealthCheckHandler() != null) {
			reg.getEurekaClient().registerHealthCheck(reg.getHealthCheckHandler());
		}
	}
}
```

``` java
@Singleton
public class DiscoveryClient implements EurekaClient {
    @Inject
    DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider) {
        if (args != null) {
            this.healthCheckHandlerProvider = args.healthCheckHandlerProvider;
            this.healthCheckCallbackProvider = args.healthCheckCallbackProvider;
            this.eventListeners.addAll(args.getEventListeners());
        } else {
            this.healthCheckCallbackProvider = null;
            this.healthCheckHandlerProvider = null;
        }

        this.applicationInfoManager = applicationInfoManager;
        //获取 instance
        InstanceInfo myInfo = applicationInfoManager.getInfo();
        //客户端配置
        clientConfig = config;
        staticClientConfig = clientConfig;
        //传输配置，心跳时间等
        transportConfig = config.getTransportConfig();
        instanceInfo = myInfo;
        if (myInfo != null) {
            appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId();
        } else {
            logger.warn("Setting instanceInfo to a passed in null value");
        }

        this.backupRegistryProvider = backupRegistryProvider;

        this.urlRandomizer = new EndpointUtils.InstanceInfoBasedUrlRandomizer(instanceInfo);
        localRegionApps.set(new Applications());

        fetchRegistryGeneration = new AtomicLong(0);

        remoteRegionsToFetch = new AtomicReference<String>(clientConfig.fetchRegistryForRemoteRegions());
        remoteRegionsRef = new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));

        //是否需要信息检索 eureka.client.fetchRegistry=true
        //
        if (config.shouldFetchRegistry()) {
            this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        //是否注册到eureka eureka.client.registerWithEureka=true
        if (config.shouldRegisterWithEureka()) {
            this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        logger.info("Initializing Eureka in region {}", clientConfig.getRegion());

        //如果不开启注册到eureka和从eureka检索服务
        //把定时任务、心跳设置为null
        if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
            logger.info("Client configured to neither register nor query for data.");
            scheduler = null;
            heartbeatExecutor = null;
            cacheRefreshExecutor = null;
            eurekaTransport = null;
            instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

            // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
            // to work with DI'd DiscoveryClient
            DiscoveryManager.getInstance().setDiscoveryClient(this);
            DiscoveryManager.getInstance().setEurekaClientConfig(config);

            initTimestampMs = System.currentTimeMillis();

            logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                    initTimestampMs, this.getApplications().size());
            return;  // no need to setup up an network tasks and we are done
        }

        // 配置作业
        try {
            scheduler = Executors.newScheduledThreadPool(3,
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-%d")
                            .setDaemon(true)
                            .build());

            heartbeatExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            cacheRefreshExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            eurekaTransport = new EurekaTransport();
            scheduleServerEndpointTask(eurekaTransport, args);

            AzToRegionMapper azToRegionMapper;
            // eureka.client.use-dns-for-fetching-service-urls
            if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
                azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
            } else {
                azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
            }
            if (null != remoteRegionsToFetch.get()) {
                azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
            }
            instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
        } catch (Throwable e) {
            throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
        }
        //刷新服务注册地址
        if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
            fetchRegistryFromBackup();
        }

        //初始化作业，心跳，状态、缓存
        initScheduledTasks();
        try {
            Monitors.registerObject(this);
        } catch (Throwable e) {
            logger.warn("Cannot register timers", e);
        }

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, this.getApplications().size());
    }

    /**
     * Register with the eureka service by making the appropriate REST call.
     */
    boolean register() throws Throwable {
        logger.info(PREFIX + appPathIdentifier + ": registering service...");
        EurekaHttpResponse<Void> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
        } catch (Exception e) {
            logger.warn("{} - registration failed {}", PREFIX + appPathIdentifier, e.getMessage(), e);
            throw e;
        }
        if (logger.isInfoEnabled()) {
            logger.info("{} - registration status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
        }
        return httpResponse.getStatusCode() == 204;
    }
}
```
在DiscoveryClient构造函数中：
* 创建了scheduler定时任务的线程池，heartbeatExecutor心跳检查线程池(服务续约)，cacheRefreshExecutor(服务获取)
* 然后initScheduledTasks()开启上面三个线程池，往上面3个线程池分别添加相应任务。然后创建了一个instanceInfoReplicator(Runnable任务)，然后调用InstanceInfoReplicator.start方法，把这个任务放进上面scheduler定时任务线程池(服务注册并更新)。

``` java
public class ApplicationInfoManager {
    public synchronized void setInstanceStatus(InstanceStatus status) {
        InstanceStatus next = instanceStatusMapper.map(status);
        if (next == null) {
            return;
        }

        InstanceStatus prev = instanceInfo.setStatus(next);
        if (prev != null) {
            for (StatusChangeListener listener : listeners.values()) {
                try {
                    //状态改变
                    listener.notify(new StatusChangeEvent(prev, next));
                } catch (Exception e) {
                    logger.warn("failed to notify listener: {}", listener.getId(), e);
                }
            }
        }
    }
}
```

``` java
private void initScheduledTasks() {
    if (clientConfig.shouldFetchRegistry()) {
        // registry cache refresh timer
        int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
        int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
        scheduler.schedule(
                new TimedSupervisorTask(
                        "cacheRefresh",
                        scheduler,
                        cacheRefreshExecutor,
                        registryFetchIntervalSeconds,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new CacheRefreshThread()
                ),
                registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }

    if (clientConfig.shouldRegisterWithEureka()) {
        int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: " + "renew interval is: " + renewalIntervalInSecs);

        // Heartbeat timer
        scheduler.schedule(
                new TimedSupervisorTask(
                        "heartbeat",
                        scheduler,
                        heartbeatExecutor,
                        renewalIntervalInSecs,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new HeartbeatThread()
                ),
                renewalIntervalInSecs, TimeUnit.SECONDS);

        // InstanceInfo replicator
        instanceInfoReplicator = new InstanceInfoReplicator(
                this,
                instanceInfo,
                clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                2); // burstSize

        //这里实现了状态改变更新服务
        statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
            @Override
            public String getId() {
                return "statusChangeListener";
            }

            @Override
            public void notify(StatusChangeEvent statusChangeEvent) {
                if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                        InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                    // log at warn level if DOWN was involved
                    logger.warn("Saw local status change event {}", statusChangeEvent);
                } else {
                    logger.info("Saw local status change event {}", statusChangeEvent);
                }
                instanceInfoReplicator.onDemandUpdate();
            }
        };

        //eureka.client.on-demand-update-status-change
        //默认为true，一般不设置成false
        if (clientConfig.shouldOnDemandUpdateStatusChange()) {
            //状态改变监听
            applicationInfoManager.registerStatusChangeListener(statusChangeListener);
        }
        //开启
        //初始化实例信息到Eureka服务端的间隔时间 eureka.client.initialInstanceInfoReplicationIntervalSeconds 默认40s
        instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```





