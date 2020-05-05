---
title: Spring Cloud Eureka (七) 服务续约
date: 2017-08-30 14:30:47
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
* [Spring Cloud Eureka(六) 服务注册](/2017/08/29/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Register/)
* Spring Cloud Eureka(七) 服务续约
* [Spring Cloud Eureka(八) 服务下线](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Cancel/)

<!-- more -->

## 概述
在dubbo环境中，服务通过netty心跳来保持长连接，来确认服务所在的服务器是否可以访问
在eureka环境中，eureka客户端通过定时任务来发送心跳来确认服务所在的服务器是否可以访问

## 服务续约（Renew）
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

                ...

        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
    ...
}
```

我们可以看到在DiscoveryClient构造函数中创建了scheduler定时任务线程池，定时执行HeartbeatThread类中的run方法
定时任务执行时间和eureka.client.lease-renewal-interval-in-second配置有关，默认40s

``` java
/**
 * The heartbeat task that renews the lease in the given intervals.
 */
private class HeartbeatThread implements Runnable {

    public void run() {
        if (renew()) {
            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
        }
    }
}

/**
 * Renew with the eureka service by making the appropriate REST call
 */
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug("{} - Heartbeat status: {}", PREFIX + appPathIdentifier, httpResponse.getStatusCode());
        if (httpResponse.getStatusCode() == 404) {
            REREGISTER_COUNTER.increment();
            logger.info("{} - Re-registering apps/{}", PREFIX + appPathIdentifier, instanceInfo.getAppName());
            return register();
        }
        return httpResponse.getStatusCode() == 200;
    } catch (Throwable e) {
        logger.error("{} - was unable to send heartbeat!", PREFIX + appPathIdentifier, e);
        return false;
    }
}
```
![eureka-client-heart](/images/spring-cloud/2017-08-29-eureka-client-heart-1.png)

## 服务续约（Server端）

InstanceResource.java
``` java
/**
 * A put request for renewing lease from a client instance.
 *
 * @param isReplication
 *            a header parameter containing information whether this is
 *            replicated from other nodes.
 * @param overriddenStatus
 *            overridden status if any.
 * @param status
 *            the {@link InstanceStatus} of the instance.
 * @param lastDirtyTimestamp
 *            last timestamp when this instance information was updated.
 * @return response indicating whether the operation was a success or
 *         failure.
 */
@PUT
public Response renewLease(
        @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
        @QueryParam("overriddenstatus") String overriddenStatus,
        @QueryParam("status") String status,
        @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
    boolean isFromReplicaNode = "true".equals(isReplication);
    boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);

    // Not found in the registry, immediately ask for a register
    if (!isSuccess) {
        logger.warn("Not Found (Renew): {} - {}", app.getName(), id);
        return Response.status(Status.NOT_FOUND).build();
    }
    // Check if we need to sync based on dirty time stamp, the client
    // instance might have changed some value
    Response response = null;
    if (lastDirtyTimestamp != null && serverConfig.shouldSyncWhenTimestampDiffers()) {
        response = this.validateDirtyTimestamp(Long.valueOf(lastDirtyTimestamp), isFromReplicaNode);
        // Store the overridden status since the validation found out the node that replicates wins
        if (response.getStatus() == Response.Status.NOT_FOUND.getStatusCode()
                && (overriddenStatus != null)
                && !(InstanceStatus.UNKNOWN.name().equals(overriddenStatus))
                && isFromReplicaNode) {
            registry.storeOverriddenStatusIfRequired(app.getAppName(), id, InstanceStatus.valueOf(overriddenStatus));
        }
    } else {
        response = Response.ok().build();
    }
    logger.debug("Found (Renew): {} - {}; reply status={}" + app.getName(), id, response.getStatus());
    return response;
}
```

spring-cloud-netflix-eureka-server项目中使用InstanceRegistry继承了PeerAwareInstanceRegistryImpl，重写renew方法
在renew方法中做了EurekaInstanceRenewedEvent事件推送, 我们可以自定义listen来处理这件事情
``` java
@Override
public boolean renew(final String appName, final String serverId,
        boolean isReplication) {
    log("renew " + appName + " serverId " + serverId + ", isReplication {}"
            + isReplication);
    List<Application> applications = getSortedApplications();
    for (Application input : applications) {
        if (input.getName().equals(appName)) {
            InstanceInfo instance = null;
            for (InstanceInfo info : input.getInstances()) {
                if (info.getId().equals(serverId)) {
                    instance = info;
                    break;
                }
            }
            publishEvent(new EurekaInstanceRenewedEvent(this, appName, serverId,
                    instance, isReplication));
            break;
        }
    }
    return super.renew(appName, serverId, isReplication);
}
```

在服务续约完成后，需要向其它eureka节点进行状态同步
PeerAwareInstanceRegistryImpl.java
``` java
public boolean renew(final String appName, final String id, final boolean isReplication) {
    if (super.renew(appName, id, isReplication)) {
        replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
        return true;
    }
    return false;
}
```
服务续约操作
AbstractInstanceRegistry.java
``` java
/**
 * Marks the given instance of the given app name as renewed, and also marks whether it originated from
 * replication.
 *
 * @see com.netflix.eureka.lease.LeaseManager#renew(java.lang.String, java.lang.String, boolean)
 */
public boolean renew(String appName, String id, boolean isReplication) {
    RENEW.increment(isReplication);
    Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
    Lease<InstanceInfo> leaseToRenew = null;
    if (gMap != null) {
        leaseToRenew = gMap.get(id);
    }
    if (leaseToRenew == null) {
        RENEW_NOT_FOUND.increment(isReplication);
        logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
        return false;
    } else {
        InstanceInfo instanceInfo = leaseToRenew.getHolder();
        if (instanceInfo != null) {
            // touchASGCache(instanceInfo.getASGName());
            InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(
                    instanceInfo, leaseToRenew, isReplication);
            if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                logger.info("Instance status UNKNOWN possibly due to deleted override for instance {}"
                        + "; re-register required", instanceInfo.getId());
                RENEW_NOT_FOUND.increment(isReplication);
                return false;
            }
            if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                Object[] args = {
                        instanceInfo.getStatus().name(),
                        instanceInfo.getOverriddenStatus().name(),
                        instanceInfo.getId()
                };
                logger.info(
                        "The instance status {} is different from overridden instance status {} for instance {}. "
                                + "Hence setting the status to overridden status", args);
                instanceInfo.setStatus(overriddenInstanceStatus);
            }
        }
        renewsLastMin.increment();
        leaseToRenew.renew();
        return true;
    }
}
```

![eureka-server-renew](/images/spring-cloud/2017-08-30-eureka-server-renew-1.png)


