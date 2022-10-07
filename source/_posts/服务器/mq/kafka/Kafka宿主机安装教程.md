---
title: kafka基本教程
date: 2020-09-28 17:14:02
categories:
  - 服务器
  - mq
tags:
  - kafka  
---

# 一、下载安装

```
wget -c https://archive.apache.org/dist/kafka/3.1.0/kafka_2.12-3.1.0.tgz
tar -zxf kafka_2.12-3.1.0.tgz
```

# 二、kafka配置

配置文件config/server.properties

```
# broker.id是kafka broker的编号，集群里每个broker的id需不同。我是从0开始。
broker.id=0
# listeners是监听地址，需要提供外网服务的话，要设置本地的IP地址
listeners=PLAINTEXT://192.168.66.11:9092
# log.dirs是日志目录，需要设置
log.dirs=/tmp/kafka-logs
# 设置Zookeeper集群地址，我是在同一个服务器上搭建了kafka和Zookeeper，所以填的本地地址
zookeeper.connect=localhost:2181
# 为新建Topic的默认Partition数量，partition数量提升，一定程度上可以提升并发性
num.partitions=1
# 内部__consumer_offsets和__transaction_state两个topic，分组元数据的复制因子，为了保证可用性，在生产上建议设置大于1。
# default.replication.factor为kafka保存消息的副本数，如果一个副本失效了，另一个还可以继续提供服务，是在自动创建topic时的默认副本数，可以设置为3
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

#数据的保存时间
log.retention.hours=168
```

设置副本数3

```
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=3
default.replication.factor=3
```

# 三、服务管理

## 3.1 zk服务启动

kafka可以使用外部zk也可以使用自带zk

```
# 启动zk
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
# 关闭zk
bin/zookeeper-server-stop.sh -daemon config/zookeeper.properties
```

## 3.2 启动及停止kafka 

```
# 启动
bin/kafka-server-start.sh -daemon config/server.properties
# 关闭 
bin/kafka-server-stop.sh config/server.properties
#启动日志查看
tail -f logs/server.log
```

# 四、测试

1 创建topic

因为我的broker只部署了一个所以指定副本1个，分区数3

```
bin/kafka-topics.sh --create --bootstrap-server 192.168.66.11:9092  --replication-factor 1 --partitions 3 --topic test 
```

2 查看主题

```
bin/kafka-topics.sh --list --bootstrap-server 192.168.66.11:9092 
```

3 发送消息

```
bin/kafka-console-producer.sh --broker-list 192.168.66.11:9092 --topic test
```

4 接收消息

```
bin/kafka-console-consumer.sh --bootstrap-server 192.168.66.11:9092 --topic test --from-beginning
```

5 查看特定主题的详细信息

```
 bin/kafka-topics.sh --bootstrap-server 192.168.66.11:9092 --describe  --topic test   
```

6 删除主题

```
bin/kafka-topics.sh --bootstrap-server 192.168.66.11:9092 --delete  --topic test
```





学习源码：

https://github.com/leelun/kafkaexamples.git





