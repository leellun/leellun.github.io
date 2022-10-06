---
title: rocketmq伪集群搭建2m-2s-async
date: 2022-05-22 18:14:02
categories:
  - 服务器
  - mq
tags:
  - rocketmq
---

# 一、环境

nameserver节点

| 节点         | 端口 |
| ------------ | ---- |
| nameserver-1 | 8876 |
| nameserver-2 | 8878 |
| nameserver-3 | 8879 |

broker节点：

| 节点       | 端口  |
| ---------- | ----- |
| broker-a   | 10911 |
| broker-a-s | 10921 |
| broker-b   | 10931 |
| broker-b-s | 10941 |

# 二、配置

## 2.1 安装

```
wget -c wget -c https://archive.apache.org/dist/rocketmq/5.0.0/rocketmq-all-5.0.0-bin-release.zip
```

## 2.2 nameserver配置

nameserver-1：

```
vi namesrv.properties

listenPort=8876
```

nameserver-2：

```
vi namesrv-b.properties

listenPort=8878
```

nameserver-3：

```
vi namesrv-c.properties

listenPort=8879
```

## 2.3 broker配置

broker-a：

```
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
namesrvAddr=192.168.66.70:8876;192.168.66.70:8878;192.168.66.70:8879
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/app/rocketmq/store-a
#commitLog 存储路径
storePathCommitLog=/app/rocketmq/store-a/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/app/rocketmq/store-a/consumequeue
#消息索引存储路径
storePathIndex=/app/rocketmq/store-a/index
#checkpoint 文件存储路径
storeCheckpoint=/app/rocketmq/store-a/checkpoint
#abort 文件存储路径
abortFile=/app/rocketmq/store-a/abort
#限制的消息大小
maxMessageSize=65536
#acl权限开启
aclEnable=true
#Broker 对外服务的监听端口
listenPort=10911
```

broker-a-s：

```
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
namesrvAddr=192.168.66.70:8876;192.168.66.70:8878;192.168.66.70:8879
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/app/rocketmq/store-a-s
#commitLog 存储路径
storePathCommitLog=/app/rocketmq/store-a-s/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/app/rocketmq/store-a-s/consumequeue
#消息索引存储路径
storePathIndex=/app/rocketmq/store-a-s/index
#checkpoint 文件存储路径
storeCheckpoint=/app/rocketmq/store-a-s/checkpoint
#abort 文件存储路径
abortFile=/app/rocketmq/store-a-s/abort
#限制的消息大小
maxMessageSize=65536
#acl权限开启
aclEnable=true
listenPort=10921
```

broker-b：

```
brokerClusterName=DefaultCluster
brokerName=broker-b
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
namesrvAddr=192.168.66.70:8876;192.168.66.70:8878;192.168.66.70:8879
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/app/rocketmq/store-b
#commitLog 存储路径
storePathCommitLog=/app/rocketmq/store-b/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/app/rocketmq/store-b/consumequeue
#消息索引存储路径
storePathIndex=/app/rocketmq/store-b/index
#checkpoint 文件存储路径
storeCheckpoint=/app/rocketmq/store-b/checkpoint
#abort 文件存储路径
abortFile=/app/rocketmq/store-b/abort
#限制的消息大小
maxMessageSize=65536
#acl权限开启
aclEnable=true
listenPort=10931
```

broker-b-s:

```
brokerClusterName=DefaultCluster
brokerName=broker-b
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
namesrvAddr=192.168.66.70:8876;192.168.66.70:8878;192.168.66.70:8879
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/app/rocketmq/store-b-s
#commitLog 存储路径
storePathCommitLog=/app/rocketmq/store-b-s/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/app/rocketmq/store-b-s/consumequeue
#消息索引存储路径
storePathIndex=/app/rocketmq/store-b-s/index
#checkpoint 文件存储路径
storeCheckpoint=/app/rocketmq/store-b-s/checkpoint
#abort 文件存储路径
abortFile=/app/rocketmq/store-b-s/abort
#限制的消息大小
maxMessageSize=65536
#acl权限开启
aclEnable=true
listenPort=10941
```

# 三、启动服务

启动nameserver

```
nohup sh bin/mqnamesrv -c conf/namesrv.properties &
nohup sh bin/mqnamesrv -c conf/namesrv-b.properties &
nohup sh bin/mqnamesrv -c conf/namesrv-c.properties &
```

启动broker

```
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-a.properties &
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-a-s.properties &
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-b.properties &
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-b-s.properties &
```

# 四、测试

发送端：

```
public class SyncProducer {
    public static final String NAMESRVADDR="192.168.66.70:8876;192.168.66.70:8878;192.168.66.70:8879";
    private static final String CONSUMER_GROUP1="please_rename_unique_group_name";
    private static final String TopicTest="TopicTest";
    public static void main(String[] args) throws Exception {
        // 实例化消息生产者Producer
        DefaultMQProducer producer = new DefaultMQProducer(CONSUMER_GROUP1);
        // 设置NameServer的地址
        producer.setNamesrvAddr(NAMESRVADDR);
        // 启动Producer实例
        producer.start();
        for (int i = 0; i < 100; i++) {
            // 创建消息，并指定Topic，Tag和消息体
            Message msg = new Message(TopicTest,
                    "TagA" /* Tag */,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );
            // 发送消息到一个Broker
            SendResult sendResult = producer.send(msg);
            // 通过sendResult返回消息是否成功送达
            System.out.printf("%s%n", sendResult);
        }
        // 如果不再发送消息，关闭Producer实例。
        producer.shutdown();
    }
}
```

消费端：

```
public class Consumer {
    public static final String NAMESRVADDR="192.168.66.70:8876;192.168.66.70:8878;192.168.66.70:8879";
    private static final String CONSUMER_GROUP1="please_rename_unique_group_name";
    private static final String TopicTest="TopicTest";
    public static void main(String[] args) throws InterruptedException, MQClientException {

        // 实例化消费者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(CONSUMER_GROUP1);

        // 设置NameServer的地址
        consumer.setNamesrvAddr(NAMESRVADDR);

        // 订阅一个或者多个Topic，以及Tag来过滤需要消费的消息
        consumer.subscribe(TopicTest, "*");
        // 注册回调实现类来处理从broker拉取回来的消息
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                // 标记该消息已经被成功消费
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 启动消费者实例
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
```

# 更多源码：

https://github.com/leellun/rocketmq-learn.git

