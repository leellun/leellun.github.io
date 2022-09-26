---
title: keepalived配置文件
date: 2021-05-05 20:14:02
categories:
  - 服务器
  - 高可用
tags:
  - keepalived
---

# 一、配置文件说明

## **第一部分：全局定义块**

1、email通知。作用：有故障，发邮件报警。
2、Lvs负载均衡器标识（lvs_id）。在一个网络内，它应该是唯一的。
3、花括号“{}”。用来分隔定义块，因此必须成对出现。如果写漏了，keepalived运行时，不会得到预期的结果。由于定义块内存在嵌套关系，因此很容易遗漏结尾处的花括号，这点要特别注意。

```
global_defs {#global_defs区域，主要是配置故障发生时的通知对象以及机器标识
   notification_email { # 故障发生时给谁发邮件通知。
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc # 指定发件人
   smtp_server 192.168.200.1 #通知邮件的smtp地址。
   smtp_connect_timeout 30 #连接smtp服务器的超时时间。
   router_id LVS_DEVEL #运行keepalived机器的一个标识
   vrrp_skip_check_adv_addr 
 #  vrrp_strict #意为严格遵循 vrrp 协议，我们平时使用需要将其注释。
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
```

vrrp_strict如果不注释需要开启防火墙

## **第二部分：vrrp_sync_group**

确定失败切换（FailOver）包含的路由实例个数。即在有2个负载均衡器的场景，一旦某个负载均衡器失效，需要自动切换到另外一个负载均衡器的实例是哪些？ 实例组group{}至少包含一个vrrp实例

```
vrrp_sync_group VG_1{ #监控多个网段的实例
    group {
　　　　VI_1 #实例名
　　　　VI_2
　　　　......
    }
    notify_master /path/xx.sh #指定当切换到master时，执行的脚本
    netify_backup /path/xx.sh #指定当切换到backup时，执行的脚本
    notify_fault "path/xx.sh VG_1" #故障时执行的脚本
    notify /path/xx.sh 
    smtp_alert #使用global_defs中提供的邮件地址和smtp服务器发送邮件通知
}
```

## 第三部分：vrrp_instance

虚拟路由实例配置

```
vrrp_instance VI_1 {
    state BACKUP #指定那个为master，那个为backup，如果设置了nopreempt这个值不起作用，主备考priority决定
    interface eth0 #设置实例绑定的网卡
    dont_track_primary #忽略vrrp的interface错误（默认不设置）
    track_interface{ #设置额外的监控，里面那个网卡出现问题都会切换
        eth0
        eth1
    }
    mcast_src_ip #发送多播包的地址，如果不设置默认使用绑定网卡的primary ip
    garp_master_delay #在切换到master状态后，延迟进行gratuitous ARP请求
    virtual_router_id 50 #VPID标记
    priority 99 #优先级，高优先级竞选为master
    advert_int 1 #检查间隔，默认1秒
    nopreempt #设置为不抢占 注：这个配置只能设置在backup主机上，而且这个主机优先级要比另外一台高
    preempt_delay #抢占延时，默认5分钟
    debug #debug级别
    authentication { #设置认证
        auth_type PASS #认证方式
        auth_pass 111111 #认证密码
    }
    virtual_ipaddress { #设置vip
        192.168.202.200
    }
}
```

## 第四部分：virtual_server

虚拟服务器virtual_server定义块 ，虚拟服务器定义是keepalived框架最重要的项目了，是keepalived.conf必不可少的部分。 该部分是用来管理LVS的，是实现keepalive和LVS相结合的模块。ipvsadm命令可以实现的管理在这里都可以通过参数配置实现，注意：real_server是被包含在viyual_server模块中的，是子模块。

```
virtual_server 192.168.202.200 23 {        //VIP地址，要和vrrp_instance模块中的                virtual_ipaddress地址一致
　　　　delay_loop 6 #健康检查时间间隔 
　　　　lb_algo rr #lvs调度算法rr|wrr|lc|wlc|lblc|sh|dh 
　　　　lb_kind DR #负载均衡转发规则NAT|DR|RUN 
　　　　persistence_timeout 5 #会话保持时间 
　　　　protocol TCP #使用的协议 
　　　　persistence_granularity <NETMASK> #lvs会话保持粒度 
　　　　virtualhost <string> #检查的web服务器的虚拟主机（host：头） 
　　　　sorry_server<IPADDR> <port> #备用机，所有realserver失效后启用


    real_server 192.168.200.5 23 {             //RS的真实IP地址
            weight 1 #默认为1,0为失效
            inhibit_on_failure #在服务器健康检查失效时，将其设为0，而不是直接从ipvs中删除 
            notify_up <string> | <quoted-string> #在检测到server up后执行脚本
            notify_down <string> | <quoted-string> #在检测到server down后执行脚本
            
        TCP_CHECK {                    //常用
            connect_timeout 3 #连接超时时间
            nb_get_retry 3 #重连次数
            delay_before_retry 3 #重连间隔时间
            connect_port 23  健康检查的端口的端口
            bindto <ip>   
          }

        HTTP_GET | SSL_GET{          //不常用
            url{ #检查url，可以指定多个
                 path /
                 digest <string> #检查后的摘要信息 genhash -s 10.0.0.102 -p 80 -u /hello
                 status_code 200 #检查的返回状态码
            }
            connect_port <port> 
            bindto <IPADD>
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 2
        }

        SMTP_CHECK{                 //不常用
            host{
                connect_ip <IP ADDRESS>
                connect_port <port> #默认检查25端口
                bindto <IP ADDRESS>
            }
            connect_timeout 5
            retry 3
            delay_before_retry 2
            helo_name <string> | <quoted-string> #smtp helo请求命令参数，可选
        }
 
        MISC_CHECK{                 //不常用
            misc_path <string> | <quoted-string> #外部脚本路径
            misc_timeout #脚本执行超时时间
            misc_dynamic #如设置该项，则退出状态码会用来动态调整服务器的权重，返回0 正常，不修改；返回1，

检查失败，权重改为0；返回2-255，正常，权重设置为：返回状态码-2
        }
    }
}
```

## 第五部分：vrrp_script

服务监控脚本。keepalived只能做到对网络故障和keepalived本身的监控，即当出现网络故障或者keepalived本身出现问题时，进行切换。但是这些还不够，我们还需要监控keepalived所在服务器上的**其他业务进程，**比如说nginx，keepalived+nginx实现nginx的负载均衡高可用。

  

```
vrrp_script check_haproxy{

    script "/home/check.sh" #监控脚本
    interval 3              #定时执行脚本间隔
    weight -20              #脚本权重
    #keepalived会定时执行脚本并对脚本执行的结果进行分析，动态调整vrrp_instance的优先级。
    #如果脚本执行结果为0，并且weight配置的值大于0，则优先级相应的增加
    #如果脚本执行结果非0，并且weight配置的值小于0，则优先级相应的减少

}
```

# 生产环境配置文件实例

```
# cat /etc/keepalived/keepalived.conf 
! Configuration File for keepalived

global_defs {
        notification_email {
                49000448@qq.com
        }
        notification_email_from Alexandre.Cassen@firewall.loc
                smtp_server 10.0.0.1
                smtp_connect_timeout 30
                router_id LVS_2
}

vrrp_instance VI_1 {
        state BACKUP
        interface eth0
        virtual_router_id 55
        priority 100
        advert_int 1
        authentication {
             auth_type PASS
             auth_pass 1111
        }
        virtual_ipaddress {
             192.168.220.110/24
        }

# 这东西还没发现实用，有时有效，有时就连不上了
virtual_server 192.168.220.110 80 {
        delay_loop 6
        lb_algo wrr
        lb_kind DR
        nat_mask 255.255.255.0
        persistence_timeout 300
        protocol TCP


        real_server 192.168.220.129 80 {
            weight 1
            TCP_CHECK {
                        connect_timeout 8
                        nb_get_retry 3
                        delay_before_retry 3
                        connect_port 80
            }
        }


        real_server 192.168.220.138 80 {
            weight 1
            TCP_CHECK {
                        connect_timeout 8
                        nb_get_retry 3
                        delay_before_retry 3
                        connect_port 80
            }
        }
}
```