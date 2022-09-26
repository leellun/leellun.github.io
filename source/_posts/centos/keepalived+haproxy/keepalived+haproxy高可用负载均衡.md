---
title: keepalived+haproxy高可用负载均衡
date: 2021-05-06 20:14:02
categories:
  - 服务器
  - 高可用
tags:
  - keepalived
  - haproxy
---

# 一、keepalived简介

Keepalived是基于vrrp协议的一款高可用软件。Keepailived有一台主服务器和多台备份服务器，在主服务器和备份服务器上面部署相同的服务配置，使用一个虚拟IP地址对外提供服务，当主服务器出现故障时，虚拟IP地址会自动漂移到备份服务器。

# 二、keepalived工作原理

keepalived是以VRRP协议为实现基础的。

VRRP（Virtual Router Redundancy Protocol，虚拟路由器冗余协议），VRRP是为了解决静态路由的高可用。VRRP的基本架构虚拟路由器由多个路由器组成，每个路由器都有各自的IP和共同的VRID(0-255)，其中一个VRRP路由器通过竞选成为MASTER，占有VIP，对外提供路由服务，其他成为BACKUP，MASTER以IP组播（组播地址：224.0.0.18）形式发送VRRP协议包，与BACKUP保持心跳连接，若MASTER不可用（或BACKUP接收不到VRRP协议包），则BACKUP通过竞选产生新的MASTER并继续对外提供路由服务，从而实现高可用。

keepalived主要有三个模块，分别是core、check和vrrp。core模块为keepalived的核心，负责主进程的启动、维护以及全局配置文件的加载和解析。check负责健康检查，包括常见的各种检查方式。vrrp模块是来实现VRRP协议的。 

# 三、HAProxy简介

HAProxy是一个免费的负载均衡软件，可以运行于大部分主流的Linux操作系统上。

HAProxy提供了L4(TCP)和L7(HTTP)两种负载均衡能力，具备丰富的功能。HAProxy的社区非常活跃，版本更新快速。最关键的是，HAProxy具备媲美商用负载均衡器的性能和稳定性。

HAProxy如今已成为免费负载均衡软件的首选 ，特别是keepalived+haproxy这套组合已经成为高可用的推荐方案了。

# 四、HAProxy的核心功能

- 负载均衡：L4和L7两种模式，支持RR/静态RR/LC/IP Hash/URI Hash/URL_PARAM Hash/HTTP_HEADER Hash等丰富的负载均衡算法
- 健康检查：支持TCP和HTTP两种健康检查模式
- 会话保持：对于未实现会话共享的应用集群，可通过Insert Cookie/Rewrite Cookie/Prefix Cookie，以及上述的多种Hash方式实现会话保持
- SSL：HAProxy可以解析HTTPS协议，并能够将请求解密为HTTP后向后端传输
- HTTP请求重写与重定向
- 监控与统计：HAProxy提供了基于Web的统计信息页面，展现健康状态和流量数据。基于此功能，使用者可以开发监控程序来监控HAProxy的状态

# 五、HAProxy的关键特性

- 采用单线程、事件驱动、非阻塞模型，减少上下文切换的消耗，能在1ms内处理数百个请求。并且每个会话只占用数KB的内存。
- 大量精细的性能优化，如O(1)复杂度的事件检查器、延迟更新技术、Single-buffereing、Zero-copy forwarding等等，这些技术使得HAProxy在中等负载下只占用极低的CPU资源。
- HAProxy大量利用操作系统本身的功能特性，使得其在处理请求时能发挥极高的性能，通常情况下，HAProxy自身只占用15%的处理时间，剩余的85%都是在系统内核层完成的。
- HAProxy作者在8年前（2009）年使用1.4版本进行了一次测试，单个HAProxy进程的处理能力突破了10万请求/秒，并轻松占满了10Gbps的网络带宽。

# 六 、keepalived与HAProxy安装

```
yum install -y keepalived
yum install -y haproxy
```

# 七、keepalived配置文件

/etc/keepalived/keepalived.conf  

```
! Configuration File for keepalived

global_defs {#global_defs区域，主要是配置故障发生时的通知对象以及机器标识
   notification_email { #notification_email 故障发生时给谁发邮件通知。
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc #notification_email_from 通知邮件从哪个地址发出。
   smtp_server 192.168.200.1 #smpt_server 通知邮件的smtp地址。
   smtp_connect_timeout 30 #smtp_connect_timeout 连接smtp服务器的超时时间。
   router_id LVS_DEVEL #router_id 标识本节点的字条串，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
   vrrp_skip_check_adv_addr 
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance VI_1 {#vrrp_instance用来定义对外提供服务的VIP区域及其相关属性。
    state MASTER #可以是MASTER或BACKUP，不过当其他节点keepalived启动时会将priority比较大的节点选举为MASTER，因此该项其实没有实质用途。
    interface eth0 #节点固有IP（非VIP）的网卡，用来发VRRP包。
    virtual_router_id 51 #virtual_router_id 取值在0-255之间，用来区分多个instance的VRRP组播。
    priority 100  #用来选举master的，要成为master，那么这个选项的值最好高于其他机器50个点，该项取值范围是1-255（在此范围之外会被识别成默认值100）。
    advert_int 1 #advert_int 发VRRP包的时间间隔，即多久进行一次master选举（可以认为是健康查检时间间隔）。
    authentication {#authentication 认证区域，认证类型有PASS和HA（IPSEC），推荐使用PASS（密码只识别前8位）
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress { #vip,虚拟IP地址池，可以有多个IP，每个IP占一行，不需要指定子网掩码。注意：这个IP必须与我们的设定的vip保持一致。
        192.168.200.16
        192.168.200.17
        192.168.200.18
    }
}

virtual_server 192.168.200.100 443 {  //VIP地址，要和vrrp_instance模块中的                virtual_ipaddress地址一致
    delay_loop 6 # 健康检查时间间隔 延迟轮询时间（单位秒）
    lb_algo rr #lvs调度算法rr|wrr|lc|wlc|lblc|sh|dh 
    lb_kind NAT #负载均衡转发规则NAT|DR|RUN 
    persistence_timeout 50#会话保持时间 
    protocol TCP#使用的协议 

    real_server 192.168.201.100 443 {#真正提供服务的服务器。
        weight 1 #权重
        SSL_GET {
            url {
              path / #path 请求real serserver上的路径。
              digest ff20ad2481f97b1754ef3e12ecd3a9cc #digest/status_code 分别表示用genhash算出的结果和http状态码。genhash -s 10.0.0.102 -p 80 -u /hello
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
virtual_server 10.10.10.2 1358 {  //VIP地址，要和vrrp_instance模块中的                virtual_ipaddress地址一致
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    sorry_server 192.168.200.200 1358 #当所有real server宕掉时，sorry server顶替。

    real_server 192.168.200.2 1358 {
        weight 1
        HTTP_GET {
            url {
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url {
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url {
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.200.3 1358 {
        weight 1
        HTTP_GET {
            url {
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334c
            }
            url {
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
virtual_server 10.10.10.3 1358 {
    delay_loop 3
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.200.4 1358 {
        weight 1
        HTTP_GET {
            url {
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url {
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url {
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.200.5 1358 {
        weight 1
        HTTP_GET {
            url {
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url {
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url {
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

### static_ipaddress和static_routes区域

static_ipaddress和static_routes区域配置的是是本节点的IP和路由信息。如果你的机器上已经配置了IP和路由，那么这两个区域可以不用配置。

### virtual_server_group和virtual_server区域

virtual_server_group一般在超大型的LVS中用到

-  virtualhost 用来给HTTP_GET和SSL_GET配置请求header的。 
- notify_up/down 当real server宕掉或启动时执行的脚本。 
- 健康检查的方式，N多种方式。 
- connect_port 健康检查，如果端口通则认为服务器正常。 
- connect_timeout,nb_get_retry,delay_before_retry分别表示超时时长、重试次数，下次重试的时间延迟。 

### vrrp_script区域

用来做健康检查的，当时检查失败时会将`vrrp_instance`的`priority`减少相应的值。 

```
vrrp_script chk_http_port {
    script "</dev/tcp/127.0.0.1/80"
    interval 1
    weight -10
}
```

## 配置文件ip作用

服务启动前

```
2: eno16777736: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:53:83:ed brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.101/24 brd 10.0.0.255 scope global noprefixroute eno16777736
       valid_lft forever preferred_lft forever
    inet6 fd56:a9ae:cb0f::c0f/128 scope global noprefixroute dynamic 
       valid_lft 39757sec preferred_lft 39757sec
    inet6 fd56:a9ae:cb0f:0:20c:29ff:fe53:83ed/64 scope global noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe53:83ed/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

服务启动后

```
2: eno16777736: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:53:83:ed brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.101/24 brd 10.0.0.255 scope global noprefixroute eno16777736
       valid_lft forever preferred_lft forever
    inet 192.168.200.16/32 scope global eno16777736
       valid_lft forever preferred_lft forever
    inet 192.168.200.17/32 scope global eno16777736
       valid_lft forever preferred_lft forever
    inet 192.168.200.18/32 scope global eno16777736
       valid_lft forever preferred_lft forever
    inet6 fd56:a9ae:cb0f::c0f/128 scope global noprefixroute dynamic 
       valid_lft 39636sec preferred_lft 39636sec
    inet6 fd56:a9ae:cb0f:0:20c:29ff:fe53:83ed/64 scope global noprefixroute 
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe53:83ed/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

# 八、haproxy配置文件

**/etc/haproxy/haproxy.cfg** 

```
#全局配置
global
    log         127.0.0.1 local2 

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000 
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http #模式为http
    log                     global #日志记录为全局的设置
    option                  httplog #记录访问日志
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s  #从连接创建开始到从客户端读取完整HTTP请求的超时时间，用于避免类DoS攻击
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s 
    timeout check           10s
    maxconn                 3000 #最大的连接数为3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend  main *:5000
    acl url_static       path_beg       -i /static /images /javascript /stylesheets
    acl url_static       path_end       -i .jpg .gif .png .css .js

    use_backend static          if url_static
    default_backend             app
#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend static 
    balance     roundrobin #负载均衡模式为轮询
    server      static 127.0.0.1:4331 check 

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend app
    balance     roundrobin
    server  app1 127.0.0.1:5001 check
    server  app2 127.0.0.1:5002 check
    server  app3 127.0.0.1:5003 check
    server  app4 127.0.0.1:5004 check

```

# 九、使用Keepalived实现HAProxy高可用

尽管HAProxy非常稳定，但仍然无法规避操作系统故障、主机硬件故障、网络故障甚至断电带来的风险。所以必须对HAProxy实施高可用方案。 

所以需要利用Keepalived实现的HAProxy热备方案 。

### 原理

在两台HAProxy的主机上分别运行着一个Keepalived实例，这两个Keepalived争抢同一个虚IP地址，两个HAProxy也尝试去绑定这同一个虚IP地址上的端口。 显然，同时只能有一个Keepalived抢到这个虚IP，抢到了这个虚IP的Keepalived主机上的HAProxy便是当前的MASTER。 Keepalived内部维护一个权重值，权重值最高的Keepalived实例能够抢到虚IP。同时Keepalived会定期check本主机上的HAProxy状态，状态OK时权重值增加。

### keepalived配置文件

```
global_defs {
    router_id LVS_DEVEL  #虚拟路由名称
}
#HAProxy健康检查配置
vrrp_script chk_haproxy {
    script "killall -0 haproxy"  #使用killall -0检查haproxy实例是否存在，性能高于ps命令
    interval 2   #脚本运行周期
    weight 2   #每次检查的加权权重值
}
#虚拟路由配置
vrrp_instance VI_1 {
    state MASTER           #本机实例状态，MASTER/BACKUP，备机配置文件中请写BACKUP
    interface enp0s25      #本机网卡名称，使用ifconfig命令查看
    virtual_router_id 51   #虚拟路由编号，主备机保持一致
    priority 101           #本机初始权重，备机请填写小于主机的值（例如100）
    advert_int 1           #争抢虚地址的周期，秒
    virtual_ipaddress {
        192.168.8.201      #虚地址IP，主备机保持一致
    }
    track_script {
        chk_haproxy        #对应的健康检查配置
    }
}
```

killall 

```
yum install -y psmisc
```

# 十、CentOS7下配置防火墙放过Keepalived

```
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT --destination 224.0.0.18 --protocol vrrp -j ACCEPT firewall-cmd --di
```

