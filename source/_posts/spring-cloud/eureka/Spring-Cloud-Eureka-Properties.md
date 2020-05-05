---
title: Spring Cloud Eureka(二) 配置文件
date: 2017-08-24 14:21:21
tags:
    - eureka
categories:
    - eureka
---

![eureka-logo](/images/spring-cloud/2017-08-25-eureka-logo-2627.png)

[Spring Cloud Eureka]
* [Spring Cloud Eureka(一) 前身今世](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-一/)
* Spring Cloud Eureka(二) 配置文件
* [Spring Cloud Eureka(三) 注册中心](/2017/08/24/spring-cloud/eureka/Spring-Cloud-Eureka-Server/)
* [Spring Cloud Eureka(四) 注册中心源码](/2017/08/25/spring-cloud/eureka/Spring-Cloud-Eureka-Server-Source/)
* [Spring Cloud Eureka(五) 客户端源码](/2017/08/25/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Source/)
* [Spring Cloud Eureka(六) 服务注册](/2017/08/29/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Register/)
* [Spring Cloud Eureka(七) 服务续约](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Heartbeat/)
* [Spring Cloud Eureka(八) 服务下线](/2017/08/30/spring-cloud/eureka/Spring-Cloud-Eureka-Client-Cancel/)

<!-- more -->

eureka属性可以在spring-cloud-netflix-eureka-server和spring-cloud-netflix-eureka中META-INF文件夹下的spring-configuration-metadata.json中找到

### Eureka Instance

| element                          | description                              |
| :------------------------------- | ---------------------------------------- |
| appname                          | 注册到eureka的应用名称                           |
| app-group-name                   | 注册到eureka的应用组                            |
| nonSecurePort                    | 如果设置了eureka.instance.non-secure-port通信端口，那么会优先于server.port |
| securePort                       | 加密端口                                     |
| nonSecurePortEnabled             | 是否启动非加密端口进行通信，默认开启                       |
| securePortEnabled                | 是否启动加密端口进行通信，默认关闭                        |
| leaseRenewalIntervalInSeconds    | 客户端发送心跳频率，默认30s                          |
| leaseExpirationDurationInSeconds | 服务端接收心跳后等待时间, 如果超过这个时间没有收到心跳则踢除此节点，默认90秒 |
| virtualHostName                  | 虚拟主机名                                    |
| instanceId                       | 全局唯一ID                                   |
| secureVirtualHostName            | 安全虚拟主机名                                  |
| ipAddress                        | ip地址                                     |
| statusPageUrlPath                | 状态路径                                     |
| statusPageUrl                    | 优先于statusPageUrlPath                     |
| homePageUrlPath                  | 首页，默认／                                   |
| homePageUrl                      | 优先于homePageUrlPath                       |
| healthCheckUrlPath               | 健康检查路径                                   |
| healthCheckUrl                   | 优先于healthCheckUrlPath                    |
| secureHealthCheckUrl             |                                          |
| namespace                        | 命名空间，在Spring Cloud被忽略，默认eureka           |
| hostname                         | 主机名称                                     |
| preferIpAddress                  | ipAddress优先，默认false                      |
| initial-status                   | 初始化状态(UP、DOWN、STARTING、OUT_OF_SERVICE、UNKNOW) |
| default-address-resolution-order |                                              |


### Eureka Server

| element                                  | description                              |
| :--------------------------------------- | ---------------------------------------- |
| a-s-g-cache-expiry-timeout-ms            | 缓存的到期时间，默认600000毫秒                       |
| a-s-g-query-timeout-ms                   | 查询aws信息超时时间，默认300毫秒                      |
| a-s-g-update-interval-ms                 | 从aws查询asg信息间隔时间，默认30000毫秒                |
| a-w-s-access-id                          | aws access-id                            |
| a-w-s-secret-key                         | aws secret-ky                            |
| batch-replication                        | 是否开启批量复制，默认关闭                            |
| binding-strategy                         | 配置绑定，AwsBindingStrategy                  |
| delta-retention-timer-interval-in-ms     |                                          |
| disable-delta                            |                                          |
| disable-delta-for-remote-regions         |                                          |
| disable-transparent-fallback-to-other-region |                                          |
| e-i-p-bind-rebind-retries                |                                          |
| e-i-p-binding-retry-interval-ms          |                                          |
| e-i-p-binding-retry-interval-ms-when-unbound |                                          |
| enable-replicated-request-compression    | 是否开启请求数据压缩，默认关闭                          |
| enable-self-preservation                 | 是否开启自我bao保护，默认开启                         |
| eviction-interval-timer-in-ms            | 续期时间，即扫描失效服务的间隔时间（缺省为60*1000ms）          |
| g-zip-content-from-remote-region         | 是否开启gzip压缩，默认开启                          |
| json-codec-name                          | json编解码器类名，默认Jackson                     |
| list-auto-scaling-groups-role-name       |                                          |
| log-identity-headers                     | 是否开启认证头，默认开启                             |
| max-elements-in-peer-replication-pool    | 节点复制任务最大数量，默认1000                        |
| max-elements-in-status-replication-pool  | 节点状态复制任务最大数量，默认1000                      |
| max-idle-thread-age-in-minutes-for-peer-replication | 节点复制线程可以存活的空闲时间，默认15分钟                   |
| max-idle-thread-in-minutes-age-for-status-replication | 状态复制线程可以存活的空闲时间，默认10分钟                   |
| max-threads-for-peer-replication         | 节点复制线程最大数量，默认20                          |
| max-threads-for-status-replication       | 状态复制线程最大数量，默认1                           |
| max-time-for-replication                 | 复制任务最大过期时间，默认30000毫秒                     |
| min-available-instances-for-peer-replication | 判断健康状态最小节点数量，默认-1，这样永远为true              |
| min-threads-for-peer-replication         | 最小节点复制线程数量，默认5                           |
| min-threads-for-status-replication       | 最小状态复制线程数量，默认1                           |
| number-of-replication-retries            | 复制重试次数，默认5次                              |
| peer-eureka-nodes-update-interval-ms     | 节点更新 定时任务initialDelay、 period值，默600000毫秒 |
| peer-eureka-status-refresh-time-interval-ms | 状态更新 定时任务initialDelay、 period值，默3000毫秒   |
| peer-node-connect-timeout-ms             | 节点请求超时时间，默认200毫秒                         |
| peer-node-connection-idle-timeout-seconds | 节点请求空闲时间，默认30毫秒                          |
| peer-node-read-timeout-ms                | 节点请求读超时时间，默认200毫秒                        |
| peer-node-total-connections              | 节点请求连接池最大连接数，默认1000                      |
| peer-node-total-connections-per-host     | 节点请求连接池每个路由默认连接数，默认500                   |
| prime-aws-replica-connections            |                                          |
| property-resolver                        |                                          |
| rate-limiter-burst-size                  |                                          |
| rate-limiter-enabled                     |                                          |
| rate-limiter-full-fetch-average-rate     |                                          |
| rate-limiter-privileged-clients          |                                          |
| rate-limiter-registry-fetch-average-rate |                                          |
| rate-limiter-throttle-standard-clients   |                                          |
| registry-sync-retries                    |                                          |
| registry-sync-retry-wait-ms              |                                          |
| remote-region-app-whitelist              |                                          |
| remote-region-connect-timeout-ms         |                                          |
| remote-region-connection-idle-timeout-seconds |                                          |
| remote-region-fetch-thread-pool-size     |                                          |
| remote-region-read-timeout-ms            |                                          |
| remote-region-registry-fetch-interval    |                                          |
| remote-region-total-connections          |                                          |
| remote-region-total-connections-per-host |                                          |
| remote-region-trust-store                |                                          |
| remote-region-trust-store-password       |                                          |
| remote-region-urls                       |                                          |
| remote-region-urls-with-name             |                                          |
| renewal-percent-threshold                |                                          |
| renewal-threshold-update-interval-ms     |                                          |
| response-cache-auto-expiration-in-seconds |                                          |
| response-cache-update-interval-ms        |                                          |
| retention-time-in-m-s-in-delta-queue     | 默认18000毫秒                                |
| route53-bind-rebind-retries              |                                          |
| route53-binding-retry-interval-ms        |                                          |
| route53-domain-t-t-l                     |                                          |
| sync-when-timestamp-differs              | 当当前节点和eureka节点时间不同时是否开启节点同步，默认开启         |
| use-read-only-response-cache             | 是否开启缓存更新任务，默认开启                          |
| wait-time-in-ms-when-sync-empty          | 等待eureka注册完成时间，默认300000毫秒                |
| xml-codec-name                           | xml编解码器，默认XStreamXml                     |



### Eureka Client

| element                                  | descriptio                         |
| :--------------------------------------- | ---------------------------------- |
| allow-redirects                          | 是否开启请求重定向到备份服务器，默认关闭               |
| availability-zones                       |                                    |
| backup-registry-impl                     |                                    |
| cache-refresh-executor-exponential-back-off-bound | scheduler 在超时的情况下，延迟执行时间最大乘数延，默认10 |
| cache-refresh-executor-thread-pool-size  | cacheRefreshExecutor 缓存刷新数量，默认2    |
| client-data-accept                       | EurekaAccept 客户端数据接收名称             |
| decoder-name                             |                                    |
| disable-delta                            |                                    |
| dollar-replacement                       |                                    |
| enabled                                  | 是否开启eureka客户端，默认开启                 |
| encoder-name                             |                                    |
| escape-char-replacement                  |                                    |
| eureka-connection-idle-timeout-seconds   | 连接到eureka server空闲时间，默认5秒          |
| eureka-server-connect-timeout-seconds    | 连接到eureka server超时时间，默认5秒          |
| eureka-server-d-n-s-name                 | eureka server dns名称                |
| eureka-server-port                       | eureka server端口                    |
| eureka-server-read-timeout-seconds       | 请求eureka server读超时时间，默认8秒          |
| eureka-server-total-connections          | 节点请求连接池最大连接数，默认200                 |
| eureka-server-total-connections-per-host | 节点请求连接池每个路由默认连接数，默认50              |
| eureka-server-u-r-l-context              |                                    |
| eureka-service-url-poll-interval-seconds | eureka注册服务器轮询时间，默认300秒             |
| fetch-registry                           | 是否开启检索注册信息，默认开启                    |
| fetch-remote-regions-registry            | aws区域列表                            |
| filter-only-up-instances                 | 是否过滤up状态的节点，默认开启                   |
| g-zip-content                            | 是否开启gzip压缩，默认开启                    |
| heartbeat-executor-exponential-back-off-bound | 心跳任务，在超时的情况下，延迟执行时间最大乘数延，默认10      |
| heartbeat-executor-thread-pool-size      | 心跳线程池大小，默认2                        |
| initial-instance-info-replication-interval-seconds | 复制间隔时间，默认40秒                       |
| instance-info-replication-interval-seconds | 服务注册任务间隔时间，默认30秒                   |
| log-delta-diff                           |                                    |
| on-demand-update-status-change           | 是否开启状态改变通知，默认true                  |
| prefer-same-zone-eureka                  |                                    |
| property-resolver                        |                                    |
| proxy-host                               | 代理地址                               |
| proxy-password                           | 代理密码                               |
| proxy-port                               | 代理端口                               |
| proxy-user-name                          | 代理用户名                              |
| region                                   |                                    |
| register-with-eureka                     | 是否注册到eureka                        |
| registry-fetch-interval-seconds          | scheduler 缓存刷新定时任务 间隔时间，默认30秒      |
| registry-refresh-single-vip-address      | eureka server vip地址                |
| service-url                              |                                    |
| transport                                |                                    |
| Use-dns-for-fetching-service-urls        | 是否从dns获取服务地址，默认关闭                  |


## Eureka Dashboard

| element | descriptio        |
| :------ | ----------------- |
| enabled | 是否开启eureka仪表盘，默认  |
| path    | eureka仪表盘路径，默认'/' |

