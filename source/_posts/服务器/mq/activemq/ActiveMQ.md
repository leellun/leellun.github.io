---
title: ActiveMQ介绍与架构说明
date: 2020-09-29 19:14:02
categories:
  - 服务器
  - mq
tags:
  - activemq
---

# 一、介绍

Apache ActiveMQ是Apache软件基金会所研发的开放源代码消息中间件；由于ActiveMQ是一个纯Java程序，因此只需要操作系统支持Java虚拟机，ActiveMQ便可执行。

# 二、ActiveMQ持久化

在ActiveMQ 中，持久化是指对消息数据的持久化。在ActiveMQ 中，默认的消息是保存在内存中的。当内存容量不足的时候，或ActiveMQ 正常关闭的时候，会将内存中的未处理的消息持久化到磁盘中。具体的持久化策略由配置文件中的具体配置决定。
ActiveMQ 的默认存储策略是kahadb。如果使用JDBC 作为持久化策略，则会将所有的需要持久化的消息保存到数据库中。
所有的持久化配置都在conf/activemq.xml 中配置，配置信息都在broker 标签内部定义。

## 2.1 kahadb

kahadb是ActiveMQ 默认的持久化策略。kahadb 是一个文件型数据库。是使用内存+文件保证数据的持久化的。kahadb 可以限制每个数据文件的大小。不代表总计数据容量。

```
<persistenceAdapter>
	<!-- directory:保存数据的目录; journalMaxFileLength:保存消息的文件大小-->
	<kahaDB directory="${activemq.data}/kahadb" journalMaxFileLength="16mb"/>
</persistenceAdapter>
```

特性：

1、日志形式存储消息；

2、消息索引以B-Tree 结构存储，可以快速更新；

3、完全支持JMS 事务；

4、支持多种恢复机制；

## 2.2 AMQ

AMQ 也是一个文件型数据库，消息信息最终是存储在文件中。内存中也会有缓存数据。并且只适用于5.3 版本之前。

性能高于JDBC，写入消息时，会将消息写入日志文件，由于是顺序追加写，性能很高。为了提升性能，创建消息主键索引，并且提供缓存机制，进一步提升性能。每个日志文件的大小都是有限制的（默认32m，可自行配置）。当超过这个大小，系统会重新建立一个文件。当所有的消息都消费完成，系统会删除这个文件或者归档。
主要的缺点是AMQ Message 会为每一个Destination 创建一个索引，如果使用了大量的Queue，索引文件的大小会占用很多磁盘空间。而且由于索引巨大，一旦Broker（ActiveMQ 应用实例）崩溃，重建索引的速度会非常慢。
虽然AMQ 性能略高于Kaha DB 方式，但是由于其重建索引时间过长，而且索引文件占用磁盘空间过大，所以已经不推荐使用。

```
<persistenceAdapter>
	<!-- directory:保存数据的目录; maxFileLength:保存消息的文件大小-->
	<amqPersistenceAdapter directory="${activemq.data}/amq" maxFileLength="32mb"/>
</persistenceAdapter>
```

## 2.3 JDBC

dataSource 指定持久化数据库的bean，createTablesOnStartup 是否在启动的时候创建数据表，默认值是true，这样每次启动都会去创建数据表了，一般是第一次启动的时候设置为true，之后改成false。

```
<broker brokerName="test-broker" persistent="true"
xmlns="http://activemq.apache.org/schema/core">
    <persistenceAdapter>
        <jdbcPersistenceAdapter dataSource="#mysql-ds" createTablesOnStartup="false"/>
    </persistenceAdapter>
</broker>
<bean id="mysql-ds" class="org.apache.commons.dbcp.BasicDataSource"
destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url"
    value="jdbc:mysql://localhost/activemq?relaxAutoCommit=true"/>
    <property name="username" value="activemq"/>
    <property name="password" value="activemq"/>
    <property name="maxActive" value="200"/>
    <property name="poolPreparedStatements" value="true"/>
</bean>
```

数据库配置：

配置成功后，需要在数据库中创建对应的database，否则无法访问。表格ActiveMQ 可以自动创建。

**activemq_msgs** 

用于存储消息，Queue 和Topic 都存储在这个表中。

字段：

ID：自增的数据库主键
CONTAINER：消息的Destination
MSGID_PROD：消息发送者客户端的主键
MSG_SEQ：是发送消息的顺序，MSGID_PROD+MSG_SEQ 可以组成JMS 的MessageID
EXPIRATION：消息的过期时间，存储的是从1970-01-01 到现在的毫秒数
MSG：消息本体的Java 序列化对象的二进制数据
PRIORITY：优先级，从0-9，数值越大优先级越高

**activemq_acks** 

用于存储订阅关系。如果是持久化Topic，订阅者和服务器的订阅关系在这个表保存。

字段：

CONTAINER：消息的Destination
SUB_DEST：如果是使用Static 集群，这个字段会有集群其他系统的信息
CLIENT_ID：每个订阅者都必须有一个唯一的客户端ID 用以区分
SUB_NAME：订阅者名称
SELECTOR：选择器，可以选择只消费满足条件的消息。条件可以用自定义属性实现，
可支持多属性AND 和OR 操作
LAST_ACKED_ID：记录消费过的消息的ID。

**activemq_lock** 

在集群环境中才有用，只有一个Broker 可以获得消息，称为Master Broker，其他的只能作为备份等待Master Broker 不可用，才可能成为下一个Master Broker。这个表用于记录哪个Broker 是当前的Master Broker。只有在消息必须保证有效，且绝对不能丢失的时候。使用JDBC 存储策略。如果消息可以容忍丢失，或使用集群/主备模式保证数据安全的时候，建议使用levelDB或Kahadb。

## 2.4 google leveldb

内存数据库存储

# 三、ActiveMQ 消息

## 3.1 API

```
send(Message message);发送消息到默认目的地，就是创建Producer 时指定的目的地。
send(Destination destination, Message message); 发送消息到指定目的地，Producer不建议绑定目的地。也就是创建Producer 的时候，不绑定目的地。session.createProducer(null)。
send(Message message, int deliveryMode, int priority, long timeToLive);发送消息到默认目的地， 且设置相关参数。deliveryMode- 持久化方式（ DeliveryMode.PERSISTENT|DeliveryMode.NON_PERSISTENT）。priority-优先级。timeToLive-消息有效期（单位毫秒）。
send(Destination destination, Message message, int deliveryMode, int priority, long timeToLive); 发送消息到指定目的地，且设置相关参数。
```

如果开启了session，需要手动提交commit

```java
Session session = connection.createSession(true, Session.CLIENT_ACKNOWLEDGE);
......
session.commit();
```

## 3.2 发送消息过期处理方式

消息过期后，默认会将失效消息保存到“死信队列（ActiveMQ.DLQ）”。不持久化的消息，在超时后直接丢弃，不会保存到死信队列中。死信队列名称可配置，死信队列中的消息不能恢复。死信队列是在activemq.xml 中配置的。

## 3.3 消息优先级

broker 可以建议权重较高的消息将会优先发送给Consumer，但priority 并不能决定消息传送的严格顺序(order)。

JMS 标准中约定priority 可以为0~9 的整数数值，值越大表示权重越高，默认值为4。但activeMQ 中各个存储器对priority 的支持并非完全一样。比如JDBC 存储器可以支持0~9，因为JDBC 存储器可以基于priority 对消息进行排序和索引化；但是对于kahadb/levelDB等这种基于日志文件的存储器而言，priority 支持相对较弱，只能识别三种优先级(LOW: <4,NORMAL: =4,HIGH: > 4)。

##### priority配置

在broker 端，默认是不存储priority 信息的，我们需要手动开启，修改activemq.xml 配
置文件，在broker 标签的子标签policyEntries 中增加下述配置：

```
<policyEntry queue=">" prioritizedMessages="true"/>
```

如果通道中Consumer 的消费速度足够快，那么消息几乎不需要从存储文件中Paged In(获取)，直接就能从内存的cache 中获取即可，这种情况下，priority 可以担保“全局顺序”；不过，如果消费者滞后太多，cache 已满，就会
触发新接收的消息直接保存在磁盘中，那么此时，priority 就没有那么有效了。

在Queue 中，prefetch 的消息列表默认将会采用“轮询”的方式。(roundRobin，注意并不是roundRobinDispatch)[备注：因为Queue 不支持任何DispatchPolicy]，依次添加到每个consumer的pending buffer 中， 比如有m1-m2-m3-m4 四条消息， 有C1-C2 两个消费者， 那么:m1->C1,m2->C2,m3->C1,m4->C2。这种轮序方式，会对基于权重的消息发送有些额外的影响，假如四条消息的权重都不同，但是(m1,m3)->C1，事实上m2 的权重>m3,对于C1 而言，它似乎丢失了“顺序性”。

##### 强顺序

```
<policyEntry queue=">" strictOrderDispatch="true"/>
```

strictOrderDispatch“严格顺序转发”，这是区别于“轮询”的一种消息转发手段；不过不要误解它为“全局严格顺序”，它只不过是将prefetch 的消息依次填满每个consumer 的pendingbuffer 。比如上述例子中， 如果C1-C2 两个消费者的buffer 尺寸为3 ， 那么(m1,m2,m3)->C1,(m4)->C2;当C1 填充完毕之后，才会填充C2。由此这种策略可以保证buffer中所有的消息都是“权重临近的”、有序的。(需要注意：strictOrderDispatch 并非是解决priority
消息顺序的问题而生，只是在使用priority 时需要关注它)。

##### 严格顺序

```
<policyEntry queue=">" prioritizedMessages="true" useCache="false" expireMessagesPeriod="0" ueuePrefetch="1"/>
```

useCache=false 来关闭内存，强制将所有的消息都立即写入文件(索引化，但是会降低消息的转发效率)；queuePrefetch=1 来约束每个consumer 任何时刻只有一个消息正在处理，那些消息消费之后，将会从文件中重新获取，这大大增加了消息文件操作的次数，不过每次读取肯定都是priority 最高的消息。

## 3.4 消息投递模式

点对点∶生产者向队列投递一条消息，只有一个消费者能够 监听得到这条消息（PTP） 

发布订阅∶生产者向队列投递一条消息，所有监听该队列的 消费者都能够监听得到这条消息（P/S） 

## 3.5 消息接收

#### 消息确认

Consumer 拉取消息后，如果没有做确认acknowledge，此消息不会从MQ 中删除。
如果消息被拉去到consumer 后，未确认，那么消息被锁定。如果consumer 关闭的时候仍旧没有确认消息，则释放消息锁定信息，消息将发送给其他的consumer 处理。

#### 消息过滤

设置过滤：
Session.createConsumer(Destination destination, String messageSelector);
过滤信息为字符串，语法类似SQL92 中的where 子句条件信息。可以使用诸如AND、OR、IN、NOT IN 等关键字（详细内容可以查看javax.jms.Message 的帮助文档）。
注意：消息的生产者在发送消息的的时候，必须设置可过滤的属性信息，所有的属性信息设置方法格式:setXxxxProperty(String name, T value)。其中方法名中的Xxxx 是类型，如setObjectProperty/setStringProperty 等。

```java
factory=new ActiveMQConnectionFactory("admin","admin","tcp://192.168.3.44:61616");
connection=factory.createConnection();
session=connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
destination=session.createQueue("filtration");
producer=session.createProducer(destination);
connection.start();
for(int i=0;i<100;i++) {
    message=session.createObjectMessage(i);
    message.setBooleanProperty("validate", true);
    message.setIntProperty("index", i);
    producer.send(message,DeliveryMode.PERSISTENT,i%9+1,2000);
}
```

```java
consumer = session.createConsumer(destination,"validate=true and index=1");
consumer.setMessageListener(new MessageListener() {
    public void onMessage(Message message) {
        try {
            //確認方法，代表consumer已經處理消息
            message.acknowledge();
            ObjectMessage oMessage=(ObjectMessage) message;
            Object data=oMessage.getObject();
            System.out.println(data);
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
});
```

# 四、ActiveMQ 集群

### Master-Slave

主从模式是一种高可用解决方案。在ZooKeeper 中注册若干ActiveMQ Broker，其中只有一个Broker 提供对外服务（Master），其他Broker 处于待机状态（Slave）。当Master 出现故障导致宕机时，通过ZooKeeper 内部的选举机制，选举出一台Slave 替代Master 继续对外提供服务。

#### ZooKeeper

| 主机            | 服务端口 | 投票端口 | 选举端口 |
| --------------- | -------- | -------- | -------- |
| 192.168.159.130 | 2181     | 2881     | 3881     |
| 192.168.159.130 | 2182     | 2882     | 3882     |
| 192.168.159.130 | 2183     | 2883     | 3883     |

#### ActiveMQ

| 主机M-S         | 通讯端口 | 服务端口 | jetty 端口 |
| --------------- | -------- | -------- | ---------- |
| 192.168.159.130 | 62626    | 61616    | 8161       |
| 192.168.159.130 | 62627    | 61617    | 8162       |
| 192.168.159.130 | 62628    | 61618    | 8163       |

#### ActiveMQ配置

##### 1 修改jetty 端口

conf/jetty.xml,分别指定8161、8162、8163

```
<bean id="jettyPort" class="org.apache.activemq.web.WebConsolePort" init-method="start">
    <!-- the default port number for the web console -->
    <property name="port" value="8161"/>
</bean>
```

##### 2 统一所有主从节点Broker命名

修改conf/activemq.xml文件，将broker的命名统一。

```
<broker xmlns="http://activemq.apache.org/schema/core" brokerName="mq-cluster"
dataDirectory="${activemq.data}">
```

##### 3 修改持久化配置

修改conf/activemq.xml 文件。修改broker 标签中子标签persistenceAdapter 相关内容。
replicas ：当前主从模型中的节点数量。
bind ：主从实例之间的通讯端口。
zkAddress 属性代表ZooKeeper 地址。
zkPath 是ActiveMQ 主从信息保存到ZooKeeper 中的路径。
hostname 为ActiveMQ 实例安装Linux 的主机名。

```xml
<persistenceAdapter>
    <!-- <kahaDB directory="${activemq.data}/kahadb"/> -->
    <replicatedLevelDB
        directory="${activemq.data}/levelDB"
        replicas="3"
        bind="tcp://0.0.0.0:62626"
        zkAddress="192.168.159.130:2181,192.168.159.130:2182,192.168.159.130:2183"
        zkPath="/activemq/leveldb-stores"
        hostname="mq-server"
    />
</persistenceAdapter>
```

##### 4 修改服务端口

 修改ActiveMQ 对外提供的服务端口。原默认端口为61616。
修改conf/activemq.xml 配置文件。修改broker 标签中子标签transportConnectors 的相关配置。

```xml
<transportConnectors>
    <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
    <transportConnector name="openwire"
    uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=1048576
    00"/>
    <transportConnector name="amqp"
    uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857
    600"/>
    <transportConnector name="stomp"
    uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=1048
    57600"/>
    <transportConnector name="mqtt"
    uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857
    600"/>
    <transportConnector name="ws"
    uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=1048576
    00"/>
</transportConnectors>
```

#### 主从启动

将三个ActiveMQ 实例分别启动。${activemq-home}/bin/active start。启动后，可以查看
日志文件，检查启动状态，日志文件为${activemq-home}/data/activemq.log。

#### 状态查看

### 集群

在所有的ActiveMQ 节点中的conf/activemq.xml 中增加下述配置：（每个主从模型中的networkConnector 都指向另外一个主从模型）

```xml
<networkConnectors>
    <networkConnector uri="static://(tcp://ip:port,tcp://ip:port)" duplex="false">
    </networkConnector>
</networkConnectors>
```

注意配置顺序，Networks 相关配置必须在持久化相关配置之前。如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://activemq.apache.org/schema/core">
    <broker xmlns="http://activemq.apache.org/schema/core" brokerName="mq-cluster"
            dataDirectory="${activemq.data}" >
        <networkConnectors>
            <networkConnector uri=" static://(tcp://ip:port,tcp://ip:port)"/>
        </networkConnectors>
        <persistenceAdapter>
            < replicatedLevelDB directory = "xxx"/>
        </persistenceAdapter>
    </broker>
</beans>
```

如： 主从模型1 - 192.168.159.129 主从模型2 - 192.168.159.130,在主从模型1 的所有节点activemq.xml 配置文件中增加标签：

```xml
<networkConnectors>
    <networkConnector
      uri="static://(tcp://192.168.159.130:61616,tcp://192.168.159.130:61617)"/>
</networkConnectors>
```

在模型2 中所有节点增加配置：

```xml
<networkConnectors>
    <networkConnector                uri="static://(tcp://192.168.159.129:61616,tcp://192.168.159.129:61617)"/>
</networkConnectors>
```

## Spring集成ActiveMQ

https://github.com/leelun/activemqdemo

activemq-producer-spring  为ActiveMQ消息的发送，activemq-consumer-spring为ActiveMQ消息的接收

