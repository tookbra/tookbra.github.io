---
title: Mysql 压缩包安装
date: 2017-08-11 13:55:07
tags:
    - mysql
    - linux
categories:
    - mysql
---

### 安装相关依赖库
``` bash
yum install libaio*
```

### 创建mysql用户组
``` bash
groupadd -r mysql && useradd -r -g mysql -s /bin/false -M mysql
```

### 创建相关目录
``` bash
mkdir -p /var/data/mysql
mkdir -p /var/log/mariadb
mkdir -p /var/run/mariadb
mkdir -p /var/log/mysql
mkdir -p /var/lib/mysql

touch /var/log/mariadb/mariadb.log
touch /var/run/mariadb/mariadb.pid

chown -R mysql:mysql /var/data/mysql /var/log/mariadb /var/run/mariadb /var/log/mysql /var/lib/mysql
```

### 下载mysql
``` bash
wget -c https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.19-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.7.19-linux-glibc2.12-x86_64.tar.gz /usr/local
cd /usr/local
tar zxf mysql-5.7.19-linux-glibc2.12-x86_64.tar.gz && mv mysql-5.7.19-linux-glibc2.12-x86_64 mysql
cd mysql
chown -R mysql:mysql .
```

### 初始化mysql
``` bash
./bin/mysqld --user=mysql --basedir=/usr/local/mysql/ --datadir=/var/data/mysql/ --initialize
```
输出:
``` shell
2017-08-14T02:56:29.504709Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2017-08-14T02:56:30.035422Z 0 [Warning] InnoDB: New log files created, LSN=45790
2017-08-14T02:56:30.126494Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2017-08-14T02:56:30.187786Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: 2c23a9cd-809c-11e7-9abe-56000083708e.
2017-08-14T02:56:30.188914Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2017-08-14T02:56:30.189840Z 1 [Note] A temporary password is generated for root@localhost: #L+dgK=hu7k!
```

### 启动mysql服务、修改root密码访问权限
``` bash
cp support-files/mysql.server /etc/init.d/mysqld

service mysqld start
/usr/local/mysql/bin/mysql -uroot -p
SET PASSWORD = PASSWORD('123456');
grant all privileges on *.* to root@'%' identified by '123456';
flush privileges;
```

### my.cnf配置
``` bash
[client]
no-beep
socket =/var/lib/mysql/mysql.sock
port=3306

[mysql]
default-character-set=utf8

[mysqld]
datadir=/var/data/mysql
socket=/var/lib/mysql/mysql.sock
port=3306
open_files_limit=65535
symbolic-links=0
character-set-server=utf8
default-storage-engine=INNODB
server-id=1
max_connections=2000
query_cache_size=0
table_open_cache=2000
tmp_table_size=246M
thread_cache_size=300

slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
log-error = /var/log/mysql/error.log
long_query_time = 0.1

[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid
```





