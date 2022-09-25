---
title: Centos查看IPVS和安装IPVS
date: 2021-06-03 21:14:02
categories:
  - 服务器
tags:
  - centos
  - ipvs
---

# 一、查看IPVS

检查IPVS是否正确安装

```
lsmod | grep ip_vs 或 ps -ef | grep ip_vs
```

查看内核有哪些模块

```
insmod / modprobe 加载驱动
rmmod 卸载驱动
lsmod 查看系统中所有已经被加载了的所有的模块以及模块 间的依赖关系
modinfo 获得模块的信息 查看已经加载的驱动模块的信息： lsmod 能够显示驱动的大小以及被谁使用

cat /proc/modules 能够显示驱动模块大小、在内核空间中的地址
cat /proc/devices 只显示驱动的主设备号，且是分类显示
/sys/modules 下面存在对应的驱动的目录，目录下包含驱动的分段信息等等。
```

# 二、安装IPVS

Centos7已经自带了LVS，因此只需要安装LVS管理程序 ipvsadm(理解为ipvs admin)并配置即可。

安装前查看

```
[root@k8s-node01 ~]# lsmod | grep ip_
ip_set                 45799  0 
ip_tables              27126  5 iptable_security,iptable_filter,iptable_mangle,iptable_nat,iptable_raw
```

开始安装

```
yum install ipset ipvsadm -y 
ipvsadm -l -n

cat >> /etc/sysconfig/modules/ipvs.modules << EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_sh
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- nf_conntrack_ipv4
EOF

chmod +x /etc/sysconfig/modules/ipvs.modules
sh /etc/sysconfig/modules/ipvs.modules
```

安装后查看

```
[root@k8s-node02 ~]# lsmod | grep ip_
ip_set                 45799  0 
ip_vs_wrr              12697  0 
ip_vs_rr               12600  0 
ip_vs_sh               12688  0 
ip_vs                 145458  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139264  7 ip_vs,nf_nat,nf_nat_ipv4,nf_nat_ipv6,xt_conntrack,nf_conntrack_ipv4,nf_conntrack_ipv6
ip_tables              27126  5 iptable_security,iptable_filter,iptable_mangle,iptable_nat,iptable_raw
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
```
# 三、ipvsadm 命令详解 

```
-C 清除表中所有的记录
-A --add-service在服务器列表中新添加一条新的虚拟服务器记录
-t 表示为tcp服务
-u 表示为udp服务
-s --scheduler 使用的调度算法， rr | wrr | lc | wlc | lblb | lblcr | dh | sh | sed | nq 默认调度算法是 wlc
ipvsadm -a -t 192.168.3.187:80 -r 192.168.200.10:80 -m -w 1
-a --add-server 在服务器表中添加一条新的真实主机记录
-t --tcp-service 说明虚拟服务器提供tcp服务
-u --udp-service 说明虚拟服务器提供udp服务
-r --real-server 真实服务器地址
-m --masquerading 指定LVS工作模式为NAT模式
-w --weight 真实服务器的权值
-g --gatewaying 指定LVS工作模式为直接路由器模式（也是LVS默认的模式）
-i --ipip 指定LVS的工作模式为隧道模式
-p 会话保持时间，定义流量呗转到同一个realserver的会话存留时间
```

# 四、查看ipvs的路由转发

iptables查看

```
iptables -t nat -nvL
```

ipvsadm -Ln

```
[root@k8s-master01 data5]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 10.0.0.10:6443               Masq    1      3          0         
TCP  10.96.0.10:53 rr
  -> 10.244.0.12:53               Masq    1      0          0         
  -> 10.244.0.13:53               Masq    1      0          0         
TCP  10.96.0.10:9153 rr
  -> 10.244.0.12:9153             Masq    1      0          0         
  -> 10.244.0.13:9153             Masq    1      0          0         
UDP  10.96.0.10:53 rr
  -> 10.244.0.12:53               Masq    1      0          0         
  -> 10.244.0.13:53               Masq    1      0          0  
```

**ipvsadm -Ln  --rate** 

```
[root@k8s-master01 ~]# ipvsadm -Ln --rate
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port                 CPS    InPPS   OutPPS    InBPS   OutBPS
  -> RemoteAddress:Port
```

--rate选项是显示速率信息

- CPS （current connection rate） 每秒连接数
- InPPS （current in packet rate） 每秒的入包个数
- OutPPS （current out packet rate） 每秒的出包个数
- InBPS （current in byte rate） 每秒入流量（字节）
- OutBPS （current out byte rate） 每秒入流量（字节）

**ipvsadm -L --stats** 

```
[root@k8s-master01 ~]# ipvsadm -L --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
```

--stats 选项是统计自该条转发规则生效以来的

- Conns （connections scheduled） 已经转发过的连接数
- InPkts （incoming packets） 入包个数
- OutPkts （outgoing packets） 出包个数
- InBytes （incoming bytes） 入流量（字节）
- OutBytes （outgoing bytes） 出流量（字节）

