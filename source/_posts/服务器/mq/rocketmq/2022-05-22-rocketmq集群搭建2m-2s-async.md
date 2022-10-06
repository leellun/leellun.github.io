---
title: rocketmq集群搭建2m-2s-async
date: 2022-05-22 18:14:02
categories:
  - 服务器
  - mq
tags:
  - rocketmq
---

# 一、rocketmq集群模式介绍

 RocketMQ支持多种集群策略

        2m-2s-async(本文采用模式)-2主2从异步刷盘(吞吐量较大，但是消息可能丢失)
    
        2m-2s-sync：2主2从同步刷盘(吞吐量会下降，但是消息更安全)
    
        2m-noslave ：2主无从(单点故障)，然后还可以直接配置broker.conf，进行单点环境配置
    
        dledger：用来实现主从切换的。集群中的节点会基于Raft协议随机选举出一个leader，
其他的就都是follower。通常正式环境都会采用这种方式来搭建集群。

# 二、配置文件说明

## 2.1 namesrv配置文件

```
vi namesrv.properties
```

内容：

```
listenPort=8876
```

## 2.2 broker配置文件

```
vi broker-a.properties 
```

内容：

```
#所属集群名字，名字一样的节点就在同一个集群内
brokerClusterName=rocketmq-cluster
#broker名字，名字一样的节点就是一组主从节点。
brokerName=broker-a
#brokerid,0就表示是Master，>0的都是表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=node1:9876;node1:9876;node1:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/app/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/app/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/app/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/app/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/app/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/app/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=ASYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量 
#sendMessageThreadPoolNums=128

#拉消息线程池数量
#pullMessageThreadPoolNums=128

#acl权限开启
aclEnable=true
```

```
vi broker-a-s.properties
```

修改brokerId和brokerRole

```
brokerId=1
brokerRole=SLAVE
```

# 三、环境准备

| 节点  | nameServer节点 | broker节点          |
| ----- | -------------- | ------------------- |
| node1 | nameserver     |                     |
| node2 | nameserver     | broker-a,broker-b-s |
| node3 | nameserver     | broker-a,broker-b-s |

# 四、启动服务

启动nameserver服务：

```
nohup sh bin/mqnamesrv -c conf/namesrv.properties &
```

启动broker服务

```
nohup sh bin/mqbroker -n localhost:8876 -c conf/broker.conf &
```

