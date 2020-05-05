---
title: Spring Cloud Config Server 解析
date: 2017-09-06 11:29:04
tags:
---

### 概述
在常规单体项目开发中，我们会有很多配置，比如本地开发环境配置、测试环境配置、生产环境配置、开关配置等
通常我们都会放在resource文件夹下，在编译打包期间，把对应的profile打包进去。

在微服务环境下场景下，由于服务的拆分会衍生出N个服务，这样每个服务都需要去写配置文件，这样一来需要管理的配置文件数据急剧上升，上线后如果需要改一些配置，会很头疼。
然而这些都不是问题，Spring Cloud为我们提供了一个统一配置管理中心[Spring Cloud Config](http://http://cloud.spring.io/spring-cloud-config/) 来解决这些问题


### Spring Cloud Config 角色
![config-server](/images/spring-cloud/2017-09-06-config-server-1.png)
Spring Cloud Config可以分成3个部分
* 远程文件，git、subversion、native、vault
* Config Server 配置中心
* Config Client 配置客户端


### Spring CLoud Config Server 启动分析
![config-server](/images/spring-cloud/2017-09-06-config-server.png)


EnvironmentRepositoryConfiguration.java
``` java
@Configuration
@ConditionalOnMissingBean(EnvironmentRepository.class)
protected static class DefaultRepositoryConfiguration {

    @Autowired
    private ConfigurableEnvironment environment;

    @Autowired
    private ConfigServerProperties server;

    @Bean
    public MultipleJGitEnvironmentRepository defaultEnvironmentRepository() {
        MultipleJGitEnvironmentRepository repository = new MultipleJGitEnvironmentRepository(this.environment);
        if (this.server.getDefaultLabel()!=null) {
            repository.setDefaultLabel(this.server.getDefaultLabel());
        }
        return repository;
    }
}
@Configuration
@Profile("native")
protected static class NativeRepositoryConfiguration {

    @Autowired
    private ConfigurableEnvironment environment;

    @Bean
    public NativeEnvironmentRepository nativeEnvironmentRepository() {
        return new NativeEnvironmentRepository(this.environment);
    }
}

@Configuration
@Profile("git")
protected static class GitRepositoryConfiguration extends DefaultRepositoryConfiguration {}

@Configuration
@Profile("subversion")
protected static class SvnRepositoryConfiguration {
    @Autowired
    private ConfigurableEnvironment environment;

    @Autowired
    private ConfigServerProperties server;

    @Bean
    public SvnKitEnvironmentRepository svnKitEnvironmentRepository() {
        SvnKitEnvironmentRepository repository = new SvnKitEnvironmentRepository(this.environment);
        if (this.server.getDefaultLabel()!=null) {
            repository.setDefaultLabel(this.server.getDefaultLabel());
        }
        return repository;
    }
}

@Configuration
@Profile("vault")
protected static class VaultConfiguration {
    @Bean
    public VaultEnvironmentRepository vaultEnvironmentRepository(HttpServletRequest request, EnvironmentWatch watch) {
        return new VaultEnvironmentRepository(request, watch, new RestTemplate());
    }
}
```

Spring Cloud Config 默认使用git来做配置存储， 还提供了subversion、native、vault3中存储方式
需要使用其它方式只需要在配置文件中指定profile
``` yml
spring:
  cloud:
    config:
      profile:
```

Spring Cloud Config 默认使用MultipleJGitEnvironmentRepository类来对git上的配置进行加载


``` java
protected File createBaseDir() {
    try {
        final File basedir = Files.createTempDirectory("config-repo-").toFile();
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                try {
                    FileUtils.delete(basedir, FileUtils.RECURSIVE);
                }
                catch (IOException e) {
                    AbstractScmAccessor.this.logger.warn(
                            "Failed to delete temporary directory on exit: " + e);
                }
            }
        });
        return basedir;
    }
    catch (IOException e) {
        throw new IllegalStateException("Cannot create temp dir", e);
    }
}
```
Spring Cloud Config在启动的时候会创建一个临时目录并且添加进了jvm的shutdown钩子中，在jvm容器停止的时候销毁这个目录

``` java
@Override
public void afterPropertiesSet() throws Exception {
    //git访问工厂
    GitCredentialsProviderFactory credentialFactory = new GitCredentialsProviderFactory();
    super.setGitCredentialsProvider(credentialFactory.createFor(getUri(),
            getUsername(), getPassword(), getPassphrase()));
    super.afterPropertiesSet();
    for (String name : this.repos.keySet()) {
        PatternMatchingJGitEnvironmentRepository repo = this.repos.get(name);
        repo.setEnvironment(getEnvironment());
        if (!StringUtils.hasText(repo.getName())) {
            repo.setName(name);
        }
        if (repo.getPattern() == null || repo.getPattern().length == 0) {
            repo.setPattern(new String[] { name });
        }
        if (getTimeout() != 0 && repo.getTimeout() == 0) {
            repo.setTimeout(getTimeout());
        }
        String user = repo.getUsername();
        String pass = repo.getPassword();
        String passphrase = repo.getPassphrase();
        if (user == null) {
            user = getUsername();
            pass = getPassword();
        }
        if (passphrase == null) {
            passphrase = getPassphrase();
        }
        repo.setGitCredentialsProvider(credentialFactory.createFor(repo.getUri(),
                user, pass, passphrase));
        repo.afterPropertiesSet();
    }
}
```

GitCredentialsProviderFactory git服务创建工厂，提供了
* UsernamePasswordCredentialsProvider 用户名密码服务
* PassphraseCredentialsProvider sshKey服务
* AwsCodeCommitCredentialProvider awsCode服务


Config Server在接受到client请求后
1. 判断git临时目录存不存，如果存在则定位到该目录否则创建目录并且定位到该目录
2. 判断是否需要更新，如果是则fetch remote并且checkout否则just checkout
3. 合并到分支
4. 创建一个新的SpringApplication应用去返回Environment信息

``` java
@RequestMapping("/{name}/{profiles:.*[^-].*}")
public Environment defaultLabel(@PathVariable String name,
        @PathVariable String profiles) {
    return labelled(name, profiles, null);
}

@RequestMapping("/{name}/{profiles}/{label:.*}")
public Environment labelled(@PathVariable String name, @PathVariable String profiles,
        @PathVariable String label) {
    if (label != null && label.contains("(_)")) {
        // "(_)" is uncommon in a git branch name, but "/" cannot be matched
        // by Spring MVC
        label = label.replace("(_)", "/");
    }
    Environment environment = this.repository.findOne(name, profiles, label);
    return environment;
}
```


### Spring CLoud Config Client 启动分析
在spring.factories中定义了引导配置类ConfigServiceBootstrapConfiguration，它的作用是在SpringBoot上下文启动时读取resources目录下的bootstrap.properties文件，取出前缀名为spring.cloud.config的配置项，这些配置项通常包含config server的节点信息、连接该节点所需要的权限配置以及客户端所要抓取的配置信息，并以此初始化ConfigClientProperties:
``` java
@Bean
public ConfigClientProperties configClientProperties() {
    ConfigClientProperties client = new ConfigClientProperties(this.environment);
    return client;
}

@Bean
@ConditionalOnProperty(value = "spring.cloud.config.enabled", matchIfMissing = true)
public ConfigServicePropertySourceLocator configServicePropertySource(ConfigClientProperties properties) {
    ConfigServicePropertySourceLocator locator = new ConfigServicePropertySourceLocator(
            properties);
    return locator;
}
```

Spring Boot 在上下文刷新前会初始化应用上下文，其中在PropertySourceBootstrapConfiguration类中
ConfigServicePropertySourceLocator类中的locate方法会被调用，从而保证在上下文刷新前可以拿到配置信息
``` java
@Override
public void initialize(ConfigurableApplicationContext applicationContext) {
    CompositePropertySource composite = new CompositePropertySource(
            BOOTSTRAP_PROPERTY_SOURCE_NAME);
    AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
    boolean empty = true;
    ConfigurableEnvironment environment = applicationContext.getEnvironment();
    for (PropertySourceLocator locator : this.propertySourceLocators) {
        PropertySource<?> source = null;
        source = locator.locate(environment);
        if (source == null) {
            continue;
        }
        logger.info("Located property source: " + source);
        composite.addPropertySource(source);
        empty = false;
    }
    ...
}
```


Spring Cloud Config使用RestTemplate来做配置的获取，默认读取超时时间为3m5s，可能在下一个版本可以去配置
请求默认Server地址为http://ip:port/{name}/{profile}
请求返回Environment对象并把它放到CompositePropertySource中

``` java
@Override
@Retryable(interceptor = "configServerRetryInterceptor")
public org.springframework.core.env.PropertySource<?> locate(
        org.springframework.core.env.Environment environment) {
    ConfigClientProperties properties = this.defaultProperties.override(environment);
    CompositePropertySource composite = new CompositePropertySource("configService");
    RestTemplate restTemplate = this.restTemplate == null ? getSecureRestTemplate(properties)
            : this.restTemplate;
    Exception error = null;
    String errorBody = null;
    logger.info("Fetching config from server at: " + properties.getRawUri());
    try {
        String[] labels = new String[] { "" };
        if (StringUtils.hasText(properties.getLabel())) {
            labels = StringUtils.commaDelimitedListToStringArray(properties.getLabel());
        }

        String state = ConfigClientStateHolder.getState();

        // Try all the labels until one works
        for (String label : labels) {
            Environment result = getRemoteEnvironment(restTemplate,
                    properties, label.trim(), state);
            if (result != null) {
                logger.info(String.format("Located environment: name=%s, profiles=%s, label=%s, version=%s, state=%s",
                        result.getName(),
                        result.getProfiles() == null ? "" : Arrays.asList(result.getProfiles()),
                        result.getLabel(), result.getVersion(), result.getState()));

                if (result.getPropertySources() != null) { // result.getPropertySources() can be null if using xml
                    for (PropertySource source : result.getPropertySources()) {
                        @SuppressWarnings("unchecked")
                        Map<String, Object> map = (Map<String, Object>) source
                                .getSource();
                        composite.addPropertySource(new MapPropertySource(source
                                .getName(), map));
                    }
                }

                if (StringUtils.hasText(result.getState()) || StringUtils.hasText(result.getVersion())) {
                    HashMap<String, Object> map = new HashMap<>();
                    putValue(map, "config.client.state", result.getState());
                    putValue(map, "config.client.version", result.getVersion());
                    composite.addFirstPropertySource(new MapPropertySource("configClient", map));
                }
                return composite;
            }
        }
    }
    catch (HttpServerErrorException e) {
        error = e;
        if (MediaType.APPLICATION_JSON.includes(e.getResponseHeaders()
                .getContentType())) {
            errorBody = e.getResponseBodyAsString();
        }
    }
    catch (Exception e) {
        error = e;
    }
    if (properties.isFailFast()) {
        throw new IllegalStateException(
                "Could not locate PropertySource and the fail fast property is set, failing",
                error);
    }
    logger.warn("Could not locate PropertySource: "
            + (errorBody == null ? error==null ? "label not found" : error.getMessage() : errorBody));
    return null;

}
```
