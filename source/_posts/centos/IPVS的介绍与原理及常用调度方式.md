---
title: Centos查看IPVS和安装IPVS
date: 2021-06-03 22:14:02
categories:
  - 服务器
  - linux
tags:
  - centos
  - ipvs
---

# IPVS的介绍与原理及常用调度方式

# 一、介绍

 IPVS (IP Virtual Server)是在 Netfilter 上层构建的，并作为 Linux 内核的一部分，实现传输层负载均衡。IPVS是LVS（Linux Virtual Server）项目重要组成部分，目前包含于官方Linux Kernel，IPVS也是一个开源软件。

IPVS的主要用途是为web服务器提供一个“前端”，处理入向连接请求。然后这些请求会发给各个http服务器来处理相应。IPVS本身就是填在linux内核中的应用，所以开销很小，性能也很好。

# 二、工作原理

 IPVS通过建立一个或多个虚拟服务器来工作，它会将工作服务器都分配给该服务器，并为该服务器创建一个linux前端。这个linux前端将入向请求先聚集到该服务器，再将每个请求的负载分发给虚拟服务一部分。然后由这些服务器处理这些请求；加入到这个虚拟服务的工作服务器越来越多，每台服务器要处理的前端分配过来的请求就越少。

IPVS 集成在 LVS（Linux Virtual Server，Linux 虚拟服务器）中，它在主机上运行，并在物理服务器集群前作为负载均衡器。IPVS 可以将基于 TCP 和 UDP 服务的请求定向到真实服务器，并使真实服务器的服务在单个IP地址上显示为虚拟服务。

# 三、IPVS常用的调度方式

1 基于轮询的调度方法

 这些调度方法将会以轮转的方式在服务器间分发连接。

**轮询调度rr**

工作机制：默认的调度算法，它会创建一个服务器序列，然后按序分配连接
优点：简单易懂，效率高。
缺点：未考虑服务器负载，所以可能一台服务器会因大量请求造成过载，而另一台高容量服务器处于空闲状态
适用情形：当你确信请求的复杂度相同时，这样，没有请求会给服务器带来很大的负载

**加权轮询调度wrr**

工作机制：所有工作服务器都会被预分配一个权值，权重越高服务器被认为处理更大负载。调度器会继续用轮询的方法，

按序向权值较高的服务器分配更多的负载
优点：采用同样的简单的轮询方法，但支持系统特定服务器分配更多负载。
缺点：需要用户驱动的加权，从而需要人工干预；没有考虑到不同的连接可能会带来不同的负载
适用情形：当你确信请求的复杂程度相同，并熟知每台服务器处理请求的能力时，这样，可以有效地分配权值，并高效地分发服务器负载.

2 基于最小连接的调度方法

调度方法会将连接分发给活动连接最小的工作服务器.

**最小连接调度lc**

工作机制：由连接数最小的服务器来处理下一个连接请求。
优点：平衡基于服务器上的实际连接数；减小了对有多个较长活动连接的单台服务器的依赖性。
缺点：未考虑每台服务器的处理能力，并熟知每天服务器处理请求的能力时，这样可以有效地分配权值，并高效地分发服务器负载。

**加权最小连接调度wlc**

工作机制：连接数最小、权重比最高的服务器处理下一个请求。它给出的算法是连接数除以权值
优点：平衡基于服务器上的实际连接数；减小了对有多个较长活动连接的单台服务器的依赖性；支持给服务器分配权重，又提供了一个一层可配置性。
缺点：需要人工干预权值;同时，在计算下个连接请求分给哪台服务器时开销也很大。
适用情形：当你确信请求的复杂相同，并熟知每天服务器处理请求的能力时。这样可以有效地分给权值，并高效分发服务器负载。

3 基于局部性的调度方法

 这种方法的基本原则是最小化服务器之间的共享缓存，从而避免增加服务器的负载。

 **基于局部性的最小连接调度lblc**

 工作机制：只要服务器没有过载并处于可用状态，便可将针对同一个终端用户的连接任务分配同一台服务器。否则，将它分配    给任务最少服务器。
优点：支持来自同一源ip地址的用户连到同一台主机上，从而避免加剧服务器端的负载；即这些用户使用的缓存数据不必复制到多台web服务器上。
缺点：由于nat技术的存在，单个ip地址背后可能有大量用户，可能会对单台服务器造成过载。
适用情形：当来自共享数据的同一ip的多个连接请求会给服务器带来大量的负载，比如出现庞大cookie或当量会话数据的情况。

# 四、三种代理模式

- VIP:virtual IP，LVS服务器上接收外网数据包的网卡IP地址。
- DIP:director IP，LVS服务器上转发数据包到realserver的网卡IP地址。
- RIP:realserver(常简称为RS)上接收Director转发数据包的IP，即提供服务的服务器IP。
- CIP:客户端的IP。

1 NAT

就是传统的NAT，进出流量都需要经过调度器，调度器会选择一个目的服务器，将进入流量的目标IP改写为负载均衡到的目标服务器，同时源IP地址也会改为调度器IP地址。

保留IP地址（10.0.0.0/255.0.0.0、172.16.0.0/255.128.0.0和192.168.0.0/255.255.0.0）[64, 65, 66]，这些地址不在Internet上使用，而是专门为内部网络预留的。当内部网络中的主机要访问Internet或被Internet访问时，就需要采用网络地址转换（Network Address Translation, 以下简称NAT），将内部地址转化为Internets上可用的外部地址。

NAT的工作原理是报文头（目标地址、源地址和端口等）被正确改写后，客户相信它们连接一个IP地址，而不同IP地址的服务器组也认为它们是与客户直接相连的。由此，可以用NAT方法将不同IP地址的并行网络服务变成在一个IP地址上的一个虚拟服务。

VS/NAT的体系结构如下图所示。在一组服务器前有一个调度器，它们是通过Switch/HUB相连接的（意思就是说真实服务器可以和IPVS负载均衡器不处于同一个子网）。这些服务器提供相同的网络服务、相同的内容，即不管请求被发送到哪一台服务器，执行结果是一样的。服务的内容可以复制到每台服务器的本地硬盘上，可以通过网络文件系统（如NFS）共享，也可以通过一个分布式文件系统来提供。

![img](IPVS的介绍与原理及常用调度方式.assets/vs-nat.jpg)nat

2 VS/TUN模式

采用NAT技术时，由于请求和响应报文都必须经过调度器地址重写，当客户请求越来越多时，调度器的处理能力将成为瓶颈。为了解决这个问题，调度器把请求报文通过IP隧道转发至真实服务器，而真实服务器将响应直接返回给客户，所以调度器只处理请求报文。由于一般网络服务响应报文比请求报文大许多，采用VS/TUN技术后，调度器得到极大的解放。

![img](IPVS的介绍与原理及常用调度方式.assets/image-23.png)

3 VS/TUN

VS/TUN模式下，调度器对数据包的处理是使用IP隧道技术进行二次封装。VS/DR模式和VS/TUN模式很类似，只不过调度器对数据包的处理是改写数据帧的目标MAC地址，通过链路层来负载均衡。

![img](IPVS的介绍与原理及常用调度方式.assets/image-24.png)

# 五、ipvs模式和iptables模式的区别

1  ipvs模式的扩展性和性能更好

以ipvs依赖iptables的场景来说明，ipvs使用ipset来存储流量的源或目标地址，而ipset使用哈希表作为基础数据结构，这样可以确保iptables规则的数量是恒定的。iptables模式下针对不同ip需要一条一条地添加规则，但对于ipset只需要将对应的ip加入到ipset集合中即可，因此iptables规则可以保持不变，数量也很少。

 IPVS 专门用于负载均衡，并使用更高效的数据结构（哈希表），允许几乎无限的规模扩张。

2 ipvs模式支持更多调度算法

如上面 ”三、IPVS常用的调度方式“