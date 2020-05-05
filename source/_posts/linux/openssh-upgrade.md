---
title: openssh upgrade
date: 2017-03-01 20:09:39
tags:
    - linux
    - aliyun
categories:
    - linux
---
![openssh](/images/linux/openssh.png)
# 前景
买了台aliyun香港节点ecs折腾下, 系统改加固的加固
<!-- more -->
## 升级openssh

1.备份系统自带openssh文件
```shell
cp -rf /etc/ssh /etc/ssh.bak
```
2.安装openssh所需依赖
```shell
yum install gcc openssl-devel pam-devel rpm-build
```
3.下载openssh文件，检测运行环境编译并安装

**这里我去掉了pam模块**
```shell
wget http://mirror.internode.on.net/pub/OpenBSD/OpenSSH/portable/openssh-7.4p1.tar.gz
tar zxvf openssh-7.4p1.tar.gz
 ./configure --prefix=/usr --sysconfdir=/etc/ssh --with-zlib --with-md5-passwords --enable-shared
make & make install
service sshd restart
```
4.解决警告
![openssh-warn](/images/linux/openssh-error.png)
```shell

sed -i '/^GSSAPICleanupCredentials/s/GSSAPICleanupCredentials yes/#GSSAPICleanupCredentials yes/' /etc/ssh/sshd_config
sed -i '/^GSSAPIAuthentication/s/GSSAPIAuthentication yes/#GSSAPIAuthentication yes/' /etc/ssh/sshd_config
sed -i '/^GSSAPIAuthentication/s/GSSAPIAuthentication no/#GSSAPIAuthentication no/' /etc/ssh/sshd_config
```
或者 在configure加入 --with-kerberos5=/usr/lib64/libkrb5.so

5.开机运行
```shell
chkconfig sshd on
```

6.问题
```shell
configure: error: PAM headers not found
需要安装pam-devel的rpm包
yum install  –y  pam-devel
```

```shell
Permissions 0640 for '/etc/ssh/ssh_host_ed25519_key' are too open
权限问题导致
chmod 0600 /etc/ssh/*
```
