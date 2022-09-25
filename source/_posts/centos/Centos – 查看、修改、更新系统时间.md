---
title: Centos – 查看、修改、更新系统时间
date: 2020-03-03 21:14:02
categories:
  - 服务器
tags:
  - centos 
---

# 一、查看系统时间

```
[root@k8s-master01 ~]# date
2022年 07月 10日 星期日 13:10:18 CST
```

# 二、手动修改系统时间

（1）设置时间

`date -s "20190712 18:30:50"`

（2）保存设置
`hwclock --systohc`

# 三、通过网络同步时间

（1）安装 ntpdate 命令

```
yum install -y ntpdate
```

（2）开始同步

```
ntpdate 0.asia.pool.ntp.org
# 强制把系统时间写入CMOS
clock -w
```

若上面的时间服务器不可用，也可以改用如下服务器进行同步：
`time.nist.gov`
`time.nuri.net`
`0.asia.pool.ntp.org`
`1.asia.pool.ntp.org`
`2.asia.pool.ntp.org`
`3.asia.pool.ntp.org`
`ntp.aliyun.com`

（3）将系统时间同步到硬件，防止系统重启后时间被还原。

```
ntpdate 0.asia.pool.ntp.org
hwclock --systohc
```

