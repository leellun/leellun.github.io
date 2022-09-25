---
title: Centos 网络配置
date: 2020-03-05 21:14:02
categories:
  - 服务器
tags:
  - centos 
  - 网络配置
---

# 网络配置文件列表：

| /etc/resolv.conf                       | 域名解析器                                           |
| -------------------------------------- | ---------------------------------------------------- |
| /etc/networks                          | 网络名和网络地址之间的映射关系                       |
| /etc/hosts                             | 存放的是域名与ip的对应关系，域名与主机名没有任何关系 |
| /etc/hostname                          | 存放的是主机名                                       |
| /etc/sysconfig/network                 | 该文件用来指定服务器上的网络配置信息                 |
| /etc/sysconfig/network-scripts/ifcfg-* | 配置为互联网网卡                                     |

# /etc/resolv.conf

域名解析器

resolv.conf的关键字主要有四个，分别是：

- nameserver    //定义DNS服务器的IP地址
- domain       //定义本地域名
- search        //定义域名的搜索列表
- sortlist        //对返回的域名进行排序

例如：

```
domain  youhang.com
search   www.youhang.site   youhang.site
nameserver 202.102.192.68
nameserver 202.102.192.69
```

# /etc/networks

网络名和网络地址之间的映射关系

/etc/networks文件定义了网络名和网络地址之间的映射关系，下面是/etc/networks文件内容的示例。

```
default 0.0.0.0 
loopback 127.0.0.0 
link-local 169.254.0.0 
test   192.168.0.0
```

# /etc/hosts

存放的是域名与ip的对应关系，域名与主机名没有任何关系

```
::1 localhost localhost.localdomain localhost6 localhost6.localdomain6
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
172.21.175.115 iZ2zejf5uriy00xpdxwkw8Z iZ2zejf5uriy00xpdxwkw8Z
```

/etc/hostname

存放的是主机名

```
localhost.domain
```

# /etc/sysconfig/network

该文件用来指定服务器上的网络配置信息，下面是一个示例： 

```
NETWORK=yes/no　　　　 网络是否被配置；
FORWARD_IPV4=yes/no　　　　是否开启IP转发功能
HOSTNAME=hostname hostname表示服务器的主机名 
GAREWAY=gw-ip　　　　 gw-ip表示网络网关的IP地址
GAREWAYDEV=gw-dev　　 gw-dw表示网关的设备名，如：etho等 
```

# /etc/sysconfig/network-scripts/ifcfg-*

**ip配置文件**

```
TYPE=Ethernet　　#配置为互联网网卡
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static #设置获取IP地址方式，默认是DHCP，设置为静态时改为static
DEFROUTE=yes
IPADDR=192.168.10.2  #设置的静态ip地址
NETMASK=255.255.255.0 # 子网掩码
NETWORK=192.168.10.0  
GATEWAY=192.168.10.254 # 网关地址
DNS1=114.114.114.114　　#配置DNS解析
ONBOOT=no　　#开启自动重启网卡
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33　　　#配置网卡名字
DEVICE=ens33　　#网卡硬件名字
UUID=d24e50a3-27c3-4410-a2b2-1d5a0c985435
IPV6_PRIVACY=no
```

**路由配置文件**

/etc/sysconfig/network-scripts/route-IFACE

# 关闭ipv6

1 编辑**/etc/sysctl.conf**配置，增加**net.ipv6.conf.all.disable_ipv6=1** 

2 编辑/**etc/sysconfig/network**配置，增加 **NETWORKING_IPV6=no**，保存并退出 

3 编辑/**etc/sysconfig/network-scripts/ifcfg-eno16777736**，确保**IPV6INIT=no**，ifcfg-eno16777736是根据自己机器的 

4 关闭防火墙的开机自启动 

**systemctl disable ip6tables.service** 

5 执行**sysctl -p或者reboot重启命令** 

# 路由添加

## 1 **使用route 命令添加**  

route(选项)(参数) 

选项

```
-A：设置地址类型；
-C：打印将Linux核心的路由缓存；
-v：详细信息模式；
-n：不执行DNS反向查找，直接显示数字形式的IP地址；
-e：netstat格式显示路由表；
-net：到一个网络的路由表；
-host：到一个主机的路由表。
```

参数

```
Add：增加指定的路由记录；
Del：删除指定的路由记录；
Target：目的网络或目的主机；
gw：设置默认网关；
mss：设置TCP的最大区块长度（MSS），单位MB；
window：指定通过路由表的TCP连接的TCP窗口大小；
dev：路由记录所表示的网络接口。
```

例子：

```
//添加到主机的路由
# route add –host 192.168.8.11 dev eth0
# route add –host 192.168.8.12 gw 192.168.8.1
 
//添加到网络的路由
# route add –net 192.168.8.11 netmask 255.255.255.0 dev eth0
# route add –net 192.168.8.11 netmask 255.255.255.0 gw 192.168.1.1
# route add –net 192.168.8.0/24 dev eth1
# route add 10.15.150.0/24 via 192.168.150.253 dev eth0
 
//添加默认网关
# route add default gw 192.168.8.1
 
//删除路由
# route del –host 192.168.8.11 dev eth0
 
//屏蔽一条路由,增加一条屏蔽的路由，目的地址为224.x.x.x将被拒绝。
route add -net 224.0.0.0 netmask 240.0.0.0 reject
```

## 2 使用ip命令来添加、删除路由 

```
ip route add default via 172.16.10.2 dev eth0
ip route add 172.16.1.0/24 via 172.16.10.2 dev eth0
```

## 3 **设置永久路由** 

1.在/etc/rc.local里添加

```
route add -net 192.168.60.0/24 dev eth0
route add -net 192.168.63.0/24 gw 192.168.63.254
```

2.在/etc/sysconfig/network里添加到末尾

 ```
GATEWAY=gw-ip
或
GATEWAY=gw-dev
 ```

 3./etc/sysconfig/static-routes :

```
any net 192.168.63.0/24 gw 192.168.63.254

any net 10.250.228.128 netmask 255.255.255.192 gw 10.250.228.129

```

