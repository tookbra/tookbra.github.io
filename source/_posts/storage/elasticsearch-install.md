---
title: elasticsearch安装
date: 2017-08-23 23:24:54
tags:
    - elasticsearch
    - linux
categories:
    - elasticsearch
---

### 下载elasticsearch，解压
``` shell
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.2.tar.gz
tar zxf elasticsearch-5.5.2.zip && mv elasticsearch-5.5.2 /usr/local/elasticsearch
cd /usr/local/elasticsearch
```

### 配置elasticsearch
``` shell
vi config/elasticsearch.yml
```

``` shell
# 集群节点名称
cluster.name: es-cluster
# 节点名称
node.name: 10
# 数据存放路径
path.data: /data/elasticsearch
# 日志存放路径
path.logs: /var/log/elasticsearch
# 是否在启动时锁定内存
bootstrap.memory_lock: false
# 绑定地址
network.host: 0.0.0.0
# 端口
http.port: 9200
```

### 启动elasticsearch
如果用root用户启动elasticsearch，会提示错误
``` java
[2017-08-24T02:16:04,076][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [10] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:127) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:114) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:67) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:122) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.cli.Command.main(Command.java:88) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:91) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:84) ~[elasticsearch-5.5.2.jar:5.5.2]
Caused by: java.lang.RuntimeException: can not run elasticsearch as root
	at org.elasticsearch.bootstrap.Bootstrap.initializeNatives(Bootstrap.java:106) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Bootstrap.setup(Bootstrap.java:194) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:351) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:123) ~[elasticsearch-5.5.2.jar:5.5.2]
	... 6 more
```

我们新建个elasticsearch用户来启动
``` shell
useradd elasticsearch
chown -R elasticsearch:elasticsearch .
#创建对应的目录
mkdir -p /var/log/elasticsearch
mkdir -p /data/elasticsearch
#启动
./bin/elasticsearch
```
启动日志
``` java
[es@db-node1 elasticsearch]$ ./bin/elasticsearch
[2017-08-24T02:35:37,649][INFO ][o.e.n.Node               ] [10] initializing ...
[2017-08-24T02:35:37,764][INFO ][o.e.e.NodeEnvironment    ] [10] using [1] data paths, mounts [[/data (/dev/vdb1)]], net usable_space [46.3gb], net total_space [49gb], spins? [possibly], types [ext4]
[2017-08-24T02:35:37,764][INFO ][o.e.e.NodeEnvironment    ] [10] heap size [1015.6mb], compressed ordinary object pointers [true]
[2017-08-24T02:35:37,766][INFO ][o.e.n.Node               ] [10] node name [10], node ID [5ErZDLQ7RUuxwtJF_LUpyg]
[2017-08-24T02:35:37,766][INFO ][o.e.n.Node               ] [10] version[5.5.2], pid[7959], build[b2f0c09/2017-08-14T12:33:14.154Z], OS[Linux/3.10.0-514.26.2.el7.x86_64/amd64], JVM[Oracle Corporation/Java HotSpot(TM) 64-Bit Server VM/1.8.0_144/25.144-b01]
[2017-08-24T02:35:37,766][INFO ][o.e.n.Node               ] [10] JVM arguments [-Xms1g, -Xmx1g, -XX:+UseConcMarkSweepGC, -XX:CMSInitiatingOccupancyFraction=75, -XX:+UseCMSInitiatingOccupancyOnly, -XX:+AlwaysPreTouch, -Xss1m, -Djava.awt.headless=true, -Dfile.encoding=UTF-8, -Djna.nosys=true, -Djdk.io.permissionsUseCanonicalPath=true, -Dio.netty.noUnsafe=true, -Dio.netty.noKeySetOptimization=true, -Dio.netty.recycler.maxCapacityPerThread=0, -Dlog4j.shutdownHookEnabled=false, -Dlog4j2.disable.jmx=true, -Dlog4j.skipJansi=true, -XX:+HeapDumpOnOutOfMemoryError, -Des.path.home=/usr/local/elasticsearch]
[2017-08-24T02:35:39,575][INFO ][o.e.p.PluginsService     ] [10] loaded module [aggs-matrix-stats]
[2017-08-24T02:35:39,575][INFO ][o.e.p.PluginsService     ] [10] loaded module [ingest-common]
[2017-08-24T02:35:39,575][INFO ][o.e.p.PluginsService     ] [10] loaded module [lang-expression]
[2017-08-24T02:35:39,576][INFO ][o.e.p.PluginsService     ] [10] loaded module [lang-groovy]
[2017-08-24T02:35:39,576][INFO ][o.e.p.PluginsService     ] [10] loaded module [lang-mustache]
[2017-08-24T02:35:39,576][INFO ][o.e.p.PluginsService     ] [10] loaded module [lang-painless]
[2017-08-24T02:35:39,576][INFO ][o.e.p.PluginsService     ] [10] loaded module [parent-join]
[2017-08-24T02:35:39,576][INFO ][o.e.p.PluginsService     ] [10] loaded module [percolator]
[2017-08-24T02:35:39,576][INFO ][o.e.p.PluginsService     ] [10] loaded module [reindex]
[2017-08-24T02:35:39,576][INFO ][o.e.p.PluginsService     ] [10] loaded module [transport-netty3]
[2017-08-24T02:35:39,576][INFO ][o.e.p.PluginsService     ] [10] loaded module [transport-netty4]
[2017-08-24T02:35:39,577][INFO ][o.e.p.PluginsService     ] [10] no plugins loaded
[2017-08-24T02:35:42,921][INFO ][o.e.d.DiscoveryModule    ] [10] using discovery type [zen]
[2017-08-24T02:35:44,116][INFO ][o.e.n.Node               ] [10] initialized
[2017-08-24T02:35:44,117][INFO ][o.e.n.Node               ] [10] starting ...
[2017-08-24T02:35:44,443][INFO ][o.e.t.TransportService   ] [10] publish_address {45.77.98.243:9300}, bound_addresses {[::]:9300}
[2017-08-24T02:35:44,468][INFO ][o.e.b.BootstrapChecks    ] [10] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
[2017-08-24T02:35:47,632][INFO ][o.e.c.s.ClusterService   ] [10] new_master {10}{5ErZDLQ7RUuxwtJF_LUpyg}{unEEP3OYTZyNtBK3v_Jk1A}{45.77.98.243}{45.77.98.243:9300}, reason: zen-disco-elected-as-master ([0] nodes joined)
[2017-08-24T02:35:47,729][INFO ][o.e.g.GatewayService     ] [10] recovered [0] indices into cluster_state
[2017-08-24T02:35:47,731][INFO ][o.e.h.n.Netty4HttpServerTransport] [10] publish_address {45.77.98.243:9200}, bound_addresses {[::]:9200}
[2017-08-24T02:35:47,731][INFO ][o.e.n.Node               ] [10] started
```

### 测试
``` txt
curl http://127.0.0.1:9200
{
  "name" : "10",
  "cluster_name" : "es-cluster",
  "cluster_uuid" : "6G4jVotkRdqaLnRKv4jz5Q",
  "version" : {
    "number" : "5.5.2",
    "build_hash" : "b2f0c09",
    "build_date" : "2017-08-14T12:33:14.154Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
}
```

### 错误解决
``` shell
<font color=red>[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]</font>
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 131072" >> /etc/security/limits.conf
echo "* soft nproc 2048" >> /etc/security/limits.conf
echo "* hard nproc 4096" >> /etc/security/limits.conf

<font color=red>[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]</font>
echo "vm.max_map_count = 655360" >> /etc/sysctl.conf
sysctl -p
```



