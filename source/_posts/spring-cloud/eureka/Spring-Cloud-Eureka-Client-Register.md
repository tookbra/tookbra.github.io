---
title: Spring Cloud Eureka (六) 服务注册
date: 2017-08-29 18:26:14
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
* [Spring Cloud Eureka(五) 客户端源码](/2017/08/25/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Source/)
* Spring Cloud Eureka(六) 服务注册
* [Spring Cloud Eureka(七) 服务续约](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Heartbeat/)
* [Spring Cloud Eureka(八) 服务下线](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Cancel/)

<!-- more -->

## 概述
服务注册（Registry），顾名思义就是服务注册到某个地方以便其它服务可以知道我需要调用的服务路径，在dubbo环境中我们通常把服务注册到zk上。
在eureka中服务注册到eureka服务器上，那么eureka服务是怎么注册的呢


## 服务注册（Registry）

``` java
@Singleton
public class DiscoveryClient implements EurekaClient {

    ...

    @Inject
    DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                        Provider<BackupRegistry> backupRegistryProvider) {

        ...

        initScheduledTasks();

        ...
    }

    private void initScheduledTasks() {
        ...

        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: " + renewalIntervalInSecs);

            ..

            // InstanceInfo replicator
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

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

            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }

            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
    ...
}
```

我们可以看到在DiscoveryClient构造函数中调用了initScheduledTasks方法，用户来初始化InstanceInfoReplicator对象，
在生成的InstanceInfoReplicator对象，创建了定时任务线程池来对当前对象进行调用
定时任务执行时间和eureka.client.initial-instance-info-replication-interval-seconds配置有关，默认40s


``` java
class InstanceInfoReplicator implements Runnable {

    InstanceInfoReplicator(DiscoveryClient discoveryClient, InstanceInfo instanceInfo, int replicationIntervalSeconds, int burstSize) {
        this.discoveryClient = discoveryClient;
        this.instanceInfo = instanceInfo;
        this.scheduler = Executors.newScheduledThreadPool(1,
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-InstanceInfoReplicator-%d")
                        .setDaemon(true)
                        .build());

        this.scheduledPeriodicRef = new AtomicReference<Future>();

        this.started = new AtomicBoolean(false);
        this.rateLimiter = new RateLimiter(TimeUnit.MINUTES);
        this.replicationIntervalSeconds = replicationIntervalSeconds;
        this.burstSize = burstSize;

        this.allowedRatePerMinute = 60 * this.burstSize / this.replicationIntervalSeconds;
        logger.info("InstanceInfoReplicator onDemand update allowed rate per min is {}", allowedRatePerMinute);
    }

    public void start(int initialDelayMs) {
        if (started.compareAndSet(false, true)) {
            instanceInfo.setIsDirty();  // for initial register
            Future next = scheduler.schedule(this, initialDelayMs, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }

    public boolean onDemandUpdate() {
        // burstSize固定为2,
        // this.allowedRatePerMinute = 60 * this.burstSize / this.replicationIntervalSeconds;
        // eureka 自己实现的令牌桶功能，burstSize最大请求数，allowedRatePerMinute平均没分钟请求数
        if (rateLimiter.acquire(burstSize, allowedRatePerMinute)) {
            scheduler.submit(new Runnable() {
                @Override
                public void run() {
                    logger.debug("Executing on-demand update of local InstanceInfo");

                    Future latestPeriodic = scheduledPeriodicRef.get();
                    if (latestPeriodic != null && !latestPeriodic.isDone()) {
                        logger.debug("Canceling the latest scheduled update, it will be rescheduled at the end of on demand update");
                        latestPeriodic.cancel(false);
                    }

                    InstanceInfoReplicator.this.run();
                }
            });
            return true;
        } else {
            logger.warn("Ignoring onDemand update due to rate limiter");
            return false;
        }
    }

    public void run() {
        try {
            //检查主机名是否更改，刷新instance信息
            discoveryClient.refreshInstanceInfo();

            //上次更新状态的时间戳
            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            if (dirtyTimestamp != null) {
                //注册
                discoveryClient.register();
                //如果上次更新的时间时间小于现在更新时间，则不设置更新标识位
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
            //设置定时任务Future到AtomicReference中
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }
}
```

``` java
@Singleton
public class DiscoveryClient implements EurekaClient {
    ...

    //注册
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

    ...
}
```
``` java
public abstract class AbstractJerseyEurekaHttpClient implements EurekaHttpClient {
    ...

    //注册请求
    @Override
    public EurekaHttpResponse<Void> register(InstanceInfo info) {
        String urlPath = "apps/" + info.getAppName();
        ClientResponse response = null;
        try {
            Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
            addExtraHeaders(resourceBuilder);
            response = resourceBuilder
                    .header("Accept-Encoding", "gzip")
                    .type(MediaType.APPLICATION_JSON_TYPE)
                    .accept(MediaType.APPLICATION_JSON)
                    .post(ClientResponse.class, info);
            return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("Jersey HTTP POST {}/{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                        response == null ? "N/A" : response.getStatus());
            }
            if (response != null) {
                response.close();
            }
        }
    }

    ...
}

```
定时任务调用服务注册会把当前的InstanceInfo信息传递过去

![eureka-client-register](/images/spring-cloud/2017-08-29-eureka-client-register.png)

## 服务注册（server端）
在eureka-core项目中有个ApplicationResource类来接收服务注册请求
``` java
/**
 * Registers information about a particular instance for an
 * {@link com.netflix.discovery.shared.Application}.
 *
 * @param info
 *            {@link InstanceInfo} information of the instance.
 * @param isReplication
 *            a header parameter containing information whether this is
 *            replicated from other nodes.
 */
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info,
                            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
    // validate that the instanceinfo contains all the necessary required fields
    if (isBlank(info.getId())) {
        return Response.status(400).entity("Missing instanceId").build();
    } else if (isBlank(info.getHostName())) {
        return Response.status(400).entity("Missing hostname").build();
    } else if (isBlank(info.getAppName())) {
        return Response.status(400).entity("Missing appName").build();
    } else if (!appName.equals(info.getAppName())) {
        return Response.status(400).entity("Mismatched appName, expecting " + appName + " but was " + info.getAppName()).build();
    } else if (info.getDataCenterInfo() == null) {
        return Response.status(400).entity("Missing dataCenterInfo").build();
    } else if (info.getDataCenterInfo().getName() == null) {
        return Response.status(400).entity("Missing dataCenterInfo Name").build();
    }

    // handle cases where clients may be registering with bad DataCenterInfo with missing data
    DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
    if (dataCenterInfo instanceof UniqueIdentifier) {
        String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
        if (isBlank(dataCenterInfoId)) {
            boolean experimental = "true".equalsIgnoreCase(serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
            if (experimental) {
                String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                return Response.status(400).entity(entity).build();
            } else if (dataCenterInfo instanceof AmazonInfo) {
                AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
                String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
                if (effectiveId == null) {
                    amazonInfo.getMetadata().put(AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
                }
            } else {
                logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
            }
        }
    }

    registry.register(info, "true".equals(isReplication));
    return Response.status(204).build();  // 204 to be backwards compatible
}
```

在这个处理请求中，我们可以看到一些必须要设置的eureka属性：
* eureka.instance.instanceId
* eureka.instance.hostname
* eureka.instance.appname

spring-cloud-netflix-eureka-server项目中使用InstanceRegistry继承了PeerAwareInstanceRegistryImpl，重写register方法
在register方法中做了EurekaInstanceRegisteredEvent事件推送, 我们可以自定义listen来处理这件事情
``` java
@Override
public void register(final InstanceInfo info, final boolean isReplication) {
    handleRegistration(info, resolveInstanceLeaseDuration(info), isReplication);
    super.register(info, isReplication);
}
```

PeerAwareInstanceRegistryImpl.java
``` java
@Override
public void register(final InstanceInfo info, final boolean isReplication) {
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    super.register(info, leaseDuration, isReplication);
    //状态同步
    replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
```

AbstractInstanceRegistry.java
``` java
    /**
     * Registers a new instance with a given duration.
     *
     * @see com.netflix.eureka.lease.LeaseManager#register(java.lang.Object, int, boolean)
     */
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // Retain the last dirty timestamp without overwriting it, if there is already a lease
            if (existingLease != null && (existingLease.getHolder() != null)) {
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = existingLease.getHolder();
                }
            } else {
                // The lease does not exist and hence it is a new registration
                synchronized (lock) {
                    if (this.expectedNumberOfRenewsPerMin > 0) {
                        // Since the client wants to cancel it, reduce the threshold
                        // (1
                        // for 30 seconds, 2 for a minute)
                        this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin + 2;
                        this.numberOfRenewsPerMinThreshold =
                                (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            gMap.put(registrant.getId(), lease);
            synchronized (recentRegisteredQueue) {
                recentRegisteredQueue.add(new Pair<Long, String>(
                        System.currentTimeMillis(),
                        registrant.getAppName() + "(" + registrant.getId() + ")"));
            }
            // This is where the initial state transfer of overridden status happens
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            //把注册信息存放在本地缓存中，使用了guava库中的LoadingCache
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
        } finally {
            read.unlock();
        }
    }
```


在服务注册完成后，需要向其它eureka节点状态同步
```java
/**
 * Replicates all eureka actions to peer eureka nodes except for replication
 * traffic to this node.
 *
 */
private void replicateToPeers(Action action, String appName, String id,
                              InstanceInfo info /* optional */,
                              InstanceStatus newStatus /* optional */, boolean isReplication) {
    Stopwatch tracer = action.getTimer().start();
    try {
        if (isReplication) {
            numberOfReplicationsLastMin.increment();
        }
        // If it is a replication already, do not replicate again as this will create a poison replication
        if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
            return;
        }

        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // If the url represents this host, do not replicate to yourself.
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    } finally {
        tracer.stop();
    }
}
```

``` java
private ReplicationInstanceResponse dispatch(ReplicationInstance instanceInfo) {
    ApplicationResource applicationResource = createApplicationResource(instanceInfo);
    InstanceResource resource = createInstanceResource(instanceInfo, applicationResource);

    String lastDirtyTimestamp = toString(instanceInfo.getLastDirtyTimestamp());
    String overriddenStatus = toString(instanceInfo.getOverriddenStatus());
    String instanceStatus = toString(instanceInfo.getStatus());

    Builder singleResponseBuilder = new Builder();
    switch (instanceInfo.getAction()) {
        case Register:
            singleResponseBuilder = handleRegister(instanceInfo, applicationResource);
            break;
        case Heartbeat:
            singleResponseBuilder = handleHeartbeat(resource, lastDirtyTimestamp, overriddenStatus, instanceStatus);
            break;
        case Cancel:
            singleResponseBuilder = handleCancel(resource);
            break;
        case StatusUpdate:
            singleResponseBuilder = handleStatusUpdate(instanceInfo, resource);
            break;
        case DeleteStatusOverride:
            singleResponseBuilder = handleDeleteStatusOverride(instanceInfo, resource);
            break;
    }
    return singleResponseBuilder.build();
}
```

PeerEurekaNode.java
``` java
/**
 * Sends the registration information of {@link InstanceInfo} receiving by
 * this node to the peer node represented by this class.
 *
 * @param info
 *            the instance information {@link InstanceInfo} of any instance
 *            that is send to this instance.
 * @throws Exception
 */
public void register(final InstanceInfo info) throws Exception {
    long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
    batchingDispatcher.process(
            taskId("register", info),
            new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                public EurekaHttpResponse<Void> execute() {
                    return replicationClient.register(info);
                }
            },
            expiryTime
    );
}
```
![eureka-server-register](/images/spring-cloud/2017-08-30-eureka-server-register.png)
