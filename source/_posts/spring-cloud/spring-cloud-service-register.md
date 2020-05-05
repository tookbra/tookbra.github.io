---
title: Spring Cloud服务注册
date: 2020-04-21 23:50:19
tags:
    - Spring Cloud
categories:
    - Spring Cloud    
#photos:
#    - /images/spring-cloud/register.jpg    
---
## Spring Cloud 服务注册

### 什么是服务注册

> 服务注册是一种目录服务，有利于服务的定义，服务的选择和在执行服务政策

通俗点讲，比如我们(服务)去餐厅(注册中心)吃饭，每个餐桌都有一个编号(服务注册属性)对应我们(服务)，这样服务员就可以通过编号找到我们，再进行点餐、结账操



### Spring Cloud 怎么实现服务注册

在构建Spring Cloud微服务项目的时候，我们通常会在服务启动类加上<font color=#c7254e>@SpringCloudApplication</font>注解并在配置文件中写上<font color=#c7254e>spring.cloud.discovery</font>指定服务注册地址来实现服务注册，这服务注册看似很简单嘛（🙂）。那Spring Cloud做了哪些让我们服务注册很简单呢？带着这个疑问我们来一步步揪出罪魁祸首。

首先看SpringCloudApplication注解，在这类中有一个<font color=#c7254e>@EnableDiscoveryClient(开启服务发现客户端)</font>注解

```java
/**
 * @author Spencer Gibb
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public @interface SpringCloudApplication {

}
```

那为啥引用@EnableDiscoveryClient注解就可以实现服务发现呢，我们再来看看EnableDiscoveryClient类中做了些什么

```java
/**
 * Annotation to enable a DiscoveryClient implementation.
 * @author Spencer Gibb
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)
public @interface EnableDiscoveryClient {

	/**
	 * If true, the ServiceRegistry will automatically register the local server.
	 * 如果是真的，那么这个服务注册将自动注册到本地服务
	 * @return - {@code true} if you want to automatically register.
	 */
	boolean autoRegister() default true;

}
```

从EnableDiscoveryClient类中我们可以看到定义了autoRegister方法，默认返回true，并且它还引用了EnableDiscoveryClientImportSelector类，那这个类做了什么值得EnableDiscoveryClient去引用呢

```java
/**
 * @author Spencer Gibb
 */
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
			importsList.add(
					"org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
			imports = importsList.toArray(new String[0]);
		}
		else {
			Environment env = getEnvironment();
			if (ConfigurableEnvironment.class.isInstance(env)) {
				ConfigurableEnvironment configEnv = (ConfigurableEnvironment) env;
				LinkedHashMap<String, Object> map = new LinkedHashMap<>();
				map.put("spring.cloud.service-registry.auto-registration.enabled", false);
				MapPropertySource propertySource = new MapPropertySource(
						"springCloudDiscoveryClient", map);
				configEnv.getPropertySources().addLast(propertySource);
			}

		}

		return imports;
	}

	@Override
	protected boolean isEnabled() {
		return getEnvironment().getProperty("spring.cloud.discovery.enabled",
				Boolean.class, Boolean.TRUE);
	}

	@Override
	protected boolean hasDefaultFactory() {
		return true;
	}

}
```

我们可以看到EnableDiscoveryClientImportSelector继承了SpringFactoryImportSelector并实现了selectImports方法，而<font color=#c7254e>SpringFactoryImportSelector</font>其实就是Spring用来实现[SPI](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface)机制的抽象类。

在<font color=#c7254e>selectImports</font>方法中首先调用了<font color=#c7254e>super.selectImports(metadata)</font>方法来找到在<font color=#c7254e>spring.factories</font>中配置key为<font color=#c7254e>org.springframework.cloud.client.discovery.EnableDiscoveryClient</font>的自动装配类

当<font color=#c7254e>autoRegister=true</font>时，会将<font color=#c7254e>org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration</font>类添加到自动装配中去

当<font color=#c7254e>autoRegister=false</font>时，会将环境变量<font color=#c7254e>spring.cloud.service-registry.auto-registration.enabled</font>改成false。这样如果我们不需要服务注册时，我们可以使用<font color=#c7254e>@EnableDiscoveryClient(autoRegister = false)</font>

看到这里有点懵(😵)，难道Spring Cloud的服务注册就这样完成了？不是的，我们稍息一会继续往下看

我们看看AutoServiceRegistrationConfiguration这个类做了哪些事情

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(AutoServiceRegistrationProperties.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled",
		matchIfMissing = true)
public class AutoServiceRegistrationConfiguration {

}
```
   
![](/images/spring-cloud/wenhao.jpg)

我都已经把裤子脱下了，给我看这些？

AutoServiceRegistrationConfiguration类中什么都没做啊，只是使AutoServiceRegistrationProperties生效并且开启自动注册，难道AutoServiceRegistrationProperties大有学问？通过反向查找找到了AbstractAutoServiceRegistration(抽象服务注册类) 一看这个名字就自动这里面肯定大有学问(🐕)

```java
public abstract class AbstractAutoServiceRegistration<R extends Registration>
		implements AutoServiceRegistration, ApplicationContextAware,
		ApplicationListener<WebServerInitializedEvent> {
    
    ....

    private AutoServiceRegistrationProperties properties;

	protected AbstractAutoServiceRegistration(ServiceRegistry<R> serviceRegistry,
			AutoServiceRegistrationProperties properties) {
		this.serviceRegistry = serviceRegistry;
		this.properties = properties;
	}     
            
    @Override
    @SuppressWarnings("deprecation")
    public void onApplicationEvent(WebServerInitializedEvent event) {
        bind(event);
    }

    @Deprecated
    public void bind(WebServerInitializedEvent event) {
        ApplicationContext context = event.getApplicationContext();
        if (context instanceof ConfigurableWebServerApplicationContext) {
            if ("management".equals(((ConfigurableWebServerApplicationContext) context)
                                    .getServerNamespace())) {
                return;
            }
        }
        this.port.compareAndSet(0, event.getWebServer().getPort());
        this.start();
    }
          
    ....   
        
    public void start() {
		if (!isEnabled()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Discovery Lifecycle disabled. Not starting");
			}
			return;
		}

		// only initialize if nonSecurePort is greater than 0 and it isn't already running
		// because of containerPortInitializer below
		if (!this.running.get()) {
			this.context.publishEvent(
					new InstancePreRegisteredEvent(this, getRegistration()));
			register();
			if (shouldRegisterManagement()) {
				registerManagement();
			}
			this.context.publishEvent(
					new InstanceRegisteredEvent<>(this, getConfiguration()));
			this.running.compareAndSet(false, true);
		}

	}
            
   /**
	 * Register the local service with the {@link ServiceRegistry}.
	 */
	protected void register() {
		this.serviceRegistry.register(getRegistration());
	}
}
```



我们可以看到<font color=#c7254e>AbstractAutoServiceRegistration</font>抽象类中监听<font color=#c7254e>WebServerInitializedEvent</font>(web容器初始化完成事件)，而服务注册必定是在服务启动完成后进行注册的，不然服务还没启动完去注册，别的服务连接进来不就访问不到了，所以此处必有妖。

接下来我们可以看到在starter方法中首先判断是否开启服务注册<font color=#c7254e>isEnabled()</font>，这个由子类去实现。再通过<font color=#c7254e>running</font>属性判断服务注册是否正在运行中，如果不在运行则发送<font color=#c7254e>InstancePreRegisteredEvent预注册事件</font>，再进行注册，最后发送<font color=#c7254e>InstanceRegisteredEvent</font>注册完成事件并修改running属性(模板方法的影子)。

我们可以看到register方法中调用了serviceRegistry.register, 而<font color=#c7254e>ServiceRegistry</font>是一个服务注册接口，我们可以看到它是这样描述的

> Contract to register and deregister instances with a Service Registry.
>
> 通过ServiceRegistry我们可以去注册或注销实例



so，看到这我们是不是明白点什么了，只要我们只要去继承<font color=#c7254e>AbstractAutoServiceRegistration</font>类并注入我们定义的<font color=#c7254e>ServiceRegistry</font>实现类，这不妥妥的我们想怎么实现服务注册就怎么实现了。那又会有同学问了，为什么Spring Cloud会这样去做呢，其实我觉得Spring Cloud只是定义了一套服务注册的规范，具体怎么实现每个人都有每个人的想法(nacos、eureka)，就好比一千个人眼中一千个哈姆雷特。



### 注

本文源码是基于<font color=#c7254e>Spring Cloud 2.2.2.RELEASE</font>版本。小弟才疏学浅，如有错误地方请大神们多多指教。
