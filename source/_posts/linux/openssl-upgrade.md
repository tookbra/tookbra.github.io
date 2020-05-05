---
title: openssl upgrade
date: 2017-03-01 20:09:49
tags:
    - linux
    - aliyun
categories:
    - linux
---

![openssl](/images/linux/openssl.png)
## 前景
新服务器，新气象，拿到服务器后对系统自带的openssl升级
<!-- more -->
## 升级openssl
备份系统自带openssl文件
```shell
mv /usr/bin/openssl /usr/bin/openssl.bak
```

安装相关依赖
```shell
yum install zlib-devel
```
下载openssl文件，检测运行环境编译并安装
```shell
wget https://www.openssl.org/source/openssl-1.1.0e.tar.gz
tar xvf openssl-1.1.0e.tar.gz && cd openssl-1.1.0e
./config shared zlib-dynamic
make & make install
```
软链接文件地址
```shell
ln -s /usr/local/bin/openssl /usr/bin/openssl
ln -s /usr/local/include/openssl/ /usr/include/openssl
ln -s /usr/local/lib64/libssl.so.1.1 /usr/lib64/libssl.so.1.1
ln -s /usr/local/lib64/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1
ldconfig -v
```


