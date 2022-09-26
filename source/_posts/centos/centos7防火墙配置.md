---
title: centos7防火墙配置
date: 2020-03-06 20:14:02
categories:
  - 服务器
  - linux
tags:
  - centos 
  - 防火墙
---

# 一、**防火墙服务**

```
yum install firewalld //防火墙安装
systemctl status firewalld  或者 firewall-cmd --state //防火墙状态查看
systemctl disable firewalld //停止
systemctl stop firewalld //禁用
systemctl start firewalld //启动
systemctl restart firewalld //重启

#selinux平时没怎么用过，所以直接 关掉了
setennforce 0 //零时关掉
vi /etc/sysconfig/selinux //配置文件永久关掉 
```

# 二、防火墙基本命令

**查看default zone和active zone**

还没做任何配置之前，default zone和active zone都应该是public

```
firewall-cmd --get-default-zone
firewall-cmd --get-active-zones
```

**查看当前开了哪些端口**

其实一个服务对应一个端口，每个服务对应/usr/lib/firewalld/services下面一个xml文件。

```
firewall-cmd --list-services
```

**查看还有哪些服务可以打开**

```
firewall-cmd --get-services
```

**查看所有打开的端口**

```
firewall-cmd --zone=public --list-ports
```

**更新防火墙规则**

```
firewall-cmd --reload
```

**添加一个服务到firewalld**

```
firewall-cmd --add-service=http //http换成想要开放的service
```

要永久开发一个service，需要加上 –permanent

```
firewall-cmd --permanent --add-service=http
```

**添加端口**

```
firewall-cmd --zone=public --add-port=443/tcp --permanent
```

**取消端口**

```
firewall-cmd --permanent --zone=public --remove-port=443/tcp
```

**防火墙配置重新加载**

```
firewall-cmd --reload
```

**查看所有zone**

```
firewall-cmd --list-all-zones
```

# 三、查看zone

```
public
  target: default
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: dhcpv6-client ssh
  ports:  22/tcp      //端口配置
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
target
可用于接受（accept）、拒绝（reject）或丢弃（drop）与任何规则（rule）(端口（port）、服务（etc）等)不匹配的每个包。在可信区域中使用ACCEPT target接收不匹配任何规则的每个包。在block zone中使用%%REJECT%%目标来拒绝（使用默认的firewalld拒绝类型）不匹配任何规则的每个包。DROP target用于drop zone中删除不匹配任何规则的每个包。如果没有指定目标，则将拒绝不匹配任何规则的每个包。
icmp-block-inversion
是一个可选标记，在区域配置中只能使用一次。此标志反转icmp块处理。区域中只接受启用的ICMP类型，而拒绝所有其他类型。
interfaces
是一个可选的标记，可以多次使用。它可用于将接口绑定到zone。
sources
是一个可选的标记，可以多次使用。它可用于将源地址、地址范围、MAC地址或ipset绑定到一个区域。
services
是一个可选的标记，可以多次使用，以启用多个服务条目。服务条目格式如下:
ports
是一个可选标记，可多次用于具有多个端口条目。端口条目的所有属性都是强制性的:
port="portid[-portid]"     #定义端口或者端口范围
protocol="tcp|udp|sctp|dccp"    #定义协议类型
```

# 四、防火墙开启问题

1 防火墙firewalld报错：ERROR: Exception DBusException: org.freedesktop.DBus.Error.AccessDenied:…

```
重启dbus服务：systemctl restart dbus
然后再：systemctl start firewalld
```