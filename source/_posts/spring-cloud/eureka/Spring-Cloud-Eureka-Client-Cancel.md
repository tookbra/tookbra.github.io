---
title: Spring Cloud Eureka (八) 服务下线
date: 2017-08-30 16:33:53
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
* [Spring Cloud Eureka(七) 服务续约](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Heartbeat/)
* Spring Cloud Eureka(八) 服务下线

<!-- more -->

## 概述
服务下线，通常在服务关闭的时候触发，在注册中心剔除下线的服务

## 服务下线（cancel）
``` java
/**
 * Shuts down Eureka Client. Also sends a deregistration request to the
 * eureka server.
 */
@PreDestroy
@Override
public synchronized void shutdown() {
    if (isShutdown.compareAndSet(false, true)) {
        logger.info("Shutting down DiscoveryClient ...");
        // 移除监听
        if (statusChangeListener != null && applicationInfoManager != null) {
            applicationInfoManager.unregisterStatusChangeListener(statusChangeListener.getId());
        }
        // 取消定时任务(心跳等)
        cancelScheduledTasks();

        // If APPINFO was registered
        // 服务下线
        if (applicationInfoManager != null && clientConfig.shouldRegisterWithEureka()) {
            applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
            unregister();
        }

        if (eurekaTransport != null) {
            eurekaTransport.shutdown();
        }
        //停止监控
        heartbeatStalenessMonitor.shutdown();
        registryStalenessMonitor.shutdown();

        logger.info("Completed shut down of DiscoveryClient");
    }
}
/**
 * unregister w/ the eureka service.
 */
void unregister() {
    // It can be null if shouldRegisterWithEureka == false
    if(eurekaTransport != null && eurekaTransport.registrationClient != null) {
        try {
            logger.info("Unregistering ...");
            EurekaHttpResponse<Void> httpResponse = eurekaTransport.registrationClient.cancel(instanceInfo.getAppName(), instanceInfo.getId());
            logger.info(PREFIX + appPathIdentifier + " - deregister  status: " + httpResponse.getStatusCode());
        } catch (Exception e) {
            logger.error(PREFIX + appPathIdentifier + " - de-registration failed" + e.getMessage(), e);
        }
    }
}
```

![eureka-client-cancel](/images/spring-cloud/2017-08-29-eureka-client-cancel.png)

## 服务下线（Server端）

InstanceResource.java
``` java
/**
 * Handles cancellation of leases for this particular instance.
 *
 * @param isReplication
 *            a header parameter containing information whether this is
 *            replicated from other nodes.
 * @return response indicating whether the operation was a success or
 *         failure.
 */
@DELETE
public Response cancelLease(
        @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    boolean isSuccess = registry.cancel(app.getName(), id,
            "true".equals(isReplication));

    if (isSuccess) {
        logger.debug("Found (Cancel): " + app.getName() + " - " + id);
        return Response.ok().build();
    } else {
        logger.info("Not Found (Cancel): " + app.getName() + " - " + id);
        return Response.status(Status.NOT_FOUND).build();
    }
}
```

PeerAwareInstanceRegistryImpl.java
``` java
@Override
public boolean cancel(final String appName, final String id,
                      final boolean isReplication) {
    if (super.cancel(appName, id, isReplication)) {
        replicateToPeers(Action.Cancel, appName, id, null, null, isReplication);
        synchronized (lock) {
            if (this.expectedNumberOfRenewsPerMin > 0) {
                // Since the client wants to cancel it, reduce the threshold (1 for 30 seconds, 2 for a minute)
                this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin - 2;
                this.numberOfRenewsPerMinThreshold =
                        (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
            }
        }
        return true;
    }
    return false;
}
```

AbstractInstanceRegistry.java
``` java
/**
 * Cancels the registration of an instance.
 *
 * <p>
 * This is normally invoked by a client when it shuts down informing the
 * server to remove the instance from traffic.
 * </p>
 *
 * @param appName the application name of the application.
 * @param id the unique identifier of the instance.
 * @param isReplication true if this is a replication event from other nodes, false
 *                      otherwise.
 * @return true if the instance was removed from the {@link AbstractInstanceRegistry} successfully, false otherwise.
 */
@Override
public boolean cancel(String appName, String id, boolean isReplication) {
    return internalCancel(appName, id, isReplication);
}

/**
 * {@link #cancel(String, String, boolean)} method is overridden by {@link PeerAwareInstanceRegistry}, so each
 * cancel request is replicated to the peers. This is however not desired for expires which would be counted
 * in the remote peers as valid cancellations, so self preservation mode would not kick-in.
 */
protected boolean internalCancel(String appName, String id, boolean isReplication) {
    try {
        read.lock();
        CANCEL.increment(isReplication);
        Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
        Lease<InstanceInfo> leaseToCancel = null;
        if (gMap != null) {
            leaseToCancel = gMap.remove(id);
        }
        synchronized (recentCanceledQueue) {
            recentCanceledQueue.add(new Pair<Long, String>(System.currentTimeMillis(), appName + "(" + id + ")"));
        }
        InstanceStatus instanceStatus = overriddenInstanceStatusMap.remove(id);
        if (instanceStatus != null) {
            logger.debug("Removed instance id {} from the overridden map which has value {}", id, instanceStatus.name());
        }
        if (leaseToCancel == null) {
            CANCEL_NOT_FOUND.increment(isReplication);
            logger.warn("DS: Registry: cancel failed because Lease is not registered for: {}/{}", appName, id);
            return false;
        } else {
            leaseToCancel.cancel();
            InstanceInfo instanceInfo = leaseToCancel.getHolder();
            String vip = null;
            String svip = null;
            if (instanceInfo != null) {
                instanceInfo.setActionType(ActionType.DELETED);
                recentlyChangedQueue.add(new RecentlyChangedItem(leaseToCancel));
                instanceInfo.setLastUpdatedTimestamp();
                vip = instanceInfo.getVIPAddress();
                svip = instanceInfo.getSecureVipAddress();
            }
            invalidateCache(appName, vip, svip);
            logger.info("Cancelled instance {}/{} (replication={})", appName, id, isReplication);
            return true;
        }
    } finally {
        read.unlock();
    }
}
```

![eureka-server-cancel](/images/spring-cloud/2017-08-30-eureka-server-cancel.png)
