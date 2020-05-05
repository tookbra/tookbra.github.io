---
title: centos加固
date: 2017-03-06 21:56:59
tags:
    - linux
    - aliyun
categories:
    - linux
---

![openssh](/images/linux/centos-logo.png)

# 前景
买了台aliyun香港节点ecs折腾下, 系统改加固的加固
<!-- more -->
## linux加固

1.更新系统
```shell
yum -y update
```

2.查看是否有人暴力破解
```shell
cat /var/log/secure|awk '/Failed/{print $(NF-3)}'|sort|uniq -c|awk '{print $2"="$1;}'
```

3.更改SSH端口，最好改为10000以上
修改 port对应的ssh端口号
```shell
vi /etc/ssh/ssh_config
vi /etc/ssh/sshd_config
```
重启sshd服务
```shell
service sshd restart
```

4.开放防火墙ssh端口
```shell
iptables -A INPUT -p tcp --dport 123456 -j ACCEPT
service iptables save
```
查看iptable 规则
```shell
iptables -L
```

5.删除不需要的账号和组，这里不要直接删除,把不需要的组和用户注释掉即可
adm、lp、shutdown、uucp、operator、games、gopher、video等

6.增加普通账号，并禁止root远程登录
```shell
useradd newuser  //添加新用户
passwd newuser  //修改密码
usermod -G10 newuser //将用户加入wheel组，允许使用 su – 命令提权成root

vi /etc/pam.d/su
#auth            required        pam_wheel.so use_uid 把这段话注释去掉
echo “SU_WHEEL_ONLY yes” >> /etc/login.defs  //禁止不在wheel组的用户使用su -命令
```

7.禁止root登陆
```shell
vi /etc/ssh/sshd_config
PermitRootLogin no  //禁止root远程登录
PermitEmptyPasswords no //禁止空密码登录
```

8.设备系统口令策略
```shell
vi /etc/login.defs
PASS_MAX_DAYS   90   //密码过期时间
PASS_MIN_LEN    12   //密码最小长度
```

9.修改帐户TMOUT值，设置自动注销时间
```shell
vi /etc/profile
HISTSIZE=10  //base历史保留条数
TMOUT=600    //自动注销时间
```

10.增加服务器文件描述符
```shell
vi /etc/security/limits.conf
*   -   nofile  65536       //在文本的最后一行添加
```

