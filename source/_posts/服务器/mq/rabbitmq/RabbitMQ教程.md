---
title: RabbitMQ入门学习教程
date: 2020-04-22 19:14:02
categories:
  - 服务器
  - mq
tags:
  - rabbitmq
---

# 一、RabbitMQ 介绍

RabbitMQ是一个消息代理 - 一个消息系统的媒介。它可以为你的应用提供一个通用的消息发送和接收平台，并且保证消息在传输过程中的安全。 

消息系统允许软件、应用相互连接和扩展．这些应用可以相互链接起来组成一个更大的应用，或者将用户设备和数据进行连接．消息系统通过将消息的发送和接收分离来实现应用程序的异步和解偶．

数据投递、非阻塞操作或推送通知。发布／订阅，异步处理，或者工作队列。所有这些都可以通过消息系统实现。 

# 二、RabbitMQ优势

- 可靠性

  RabbitMQ提供了多种技术可以让你在性能和可靠性之间进行权衡。这些技术包括持久性机制、投递确认、发布者证实和高可用性机制。 

- 灵活的路由

  消息在到达队列前是通过交换机进行路由的。RabbitMQ为典型的路由逻辑提供了多种内置交换机类型。如果你有更复杂的路由需求，可以将这些交换机组合起来使用，你甚至可以实现自己的交换机类型，并且当做RabbitMQ的插件来使用。 

- 集群

  在相同局域网中的多个RabbitMQ服务器可以聚合在一起，作为一个独立的逻辑代理来使用。 

- 联合

  对于服务器来说，它比集群需要更多的松散和非可靠链接。为此RabbitMQ提供了联合模型。 

- 高可用的队列

  在同一个集群里，队列可以被镜像到多个机器中，以确保当其中某些硬件出现故障后，你的消息仍然安全。 

- 多协议

  RabbitMQ 支持多种消息协议的消息传递。 

- 广泛的客户端

  编程语言几乎都有与其相适配的RabbitMQ客户端。 

- 追踪

  如果你的消息系统有异常行为，RabbitMQ还提供了追踪的支持，让你能够发现问题所在。 

- 插件系统

  RabbitMQ附带了各种各样的插件来对自己进行扩展。 


# 三、AMQP 协议

AMQP（高级消息队列协议）是一个网络协议。它支持符合要求的客户端应用（application）和消息中间件代理（messaging middleware broker）之间进行通信。 

消息代理（message brokers）从发布者（publishers）亦称生产者（producers）那儿接收消息，并根据既定的路由规则把接收到的消息发送给处理消息的消费者（consumers）。

由于AMQP是一个网络协议，所以这个过程中的发布者，消费者，消息代理 可以存在于不同的设备上。

## 3.1 AMQP 0-9-1 模型

![enter image description here](RabbitMQ教程/hello-world-example-routing.png) 

消息（message）被发布者（publisher）发送给交换机（exchange），交换机常常被比喻成邮局或者邮箱。然后交换机将收到的消息根据路由规则分发给绑定的队列（queue）。最后AMQP代理会将消息投递给订阅了此队列的消费者，或者消费者按照需求自行获取。 

发布者（publisher）发布消息时可以给消息指定各种消息属性（message meta-data）。有些属性有可能会被消息代理（brokers）使用，然而其他的属性则是完全不透明的，它们只能被接收消息的应用所使用。 

AMQP模块包含了一个消息确认（message acknowledgements）的概念：

当一个消息从队列中投递给消费者后（consumer），消费者会通知一下消息代理（broker），这个可以是自动的也可以由处理消息的应用的开发者执行。当“消息确认”被启用的时候，消息代理不会完全将消息从队列中删除，直到它收到来自消费者的确认回执（acknowledgement）。 

队列，交换机和绑定统称为AMQP实体（AMQP entities）。 

## 3.2 JMI和AMQP

MQ是消息通信的模型，并发具体实现。现在实现MQ的有两种主流方式：AMQP、JMS。

两者间的区别和联系：

- JMS是定义了统一的接口，来对消息操作进行统一；AMQP是通过规定协议来统一数据交互的格式

- JMS限定了必须使用Java语言；AMQP只是协议，不规定实现方式，因此是跨语言的。

- JMS规定了两种消息模型；而AMQP的消息模型更加丰富

JMS两种消息模型：

- P2P（Point to Point）点对点模式
- Publish/Subscribe(Pub/Sub)发布订阅模式

# 四、RabbitMQ安装

## 4.1 软件安装

因为RabbitMQ是基于erlang语言开发的，所以必须安装erlang

```
//安装依赖
yum -y install gcc glibc-devel make ncurses-devel openssl-devel xmlto perl wget gtk2-devel binutils-devel

//-------下面是安装RabbitMQ的依赖环境 因为RabbitMQ是基于erlang语言开发的，所以必须安装erlang----
//下载erlang
wget http://erlang.org/download/otp_src_22.0.tar.gz
tar -zxvf otp_src_22.0.tar.gz
cd otp_src_22.0
./configure --prefix=/usr/local/erlang
make install

//添加安装环境
echo 'export PATH=$PATH:/usr/local/erlang/bin' >> /etc/profile
source /etc/profile

//测试erl
erl
```

安装RabbitMQ

```
//下载
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.7.15/rabbitmq-server-generic-unix-3.7.15.tar.xz
yum install -y xz
xz -d rabbitmq-server-generic-unix-3.7.15.tar.xz
tar -xvf rabbitmq-server-generic-unix-3.7.15.tar
mv rabbitmq_server-3.7.15/ /usr/local/
mv /usr/local/rabbitmq_server-3.7.15  /usr/local/rabbitmq

//添加系统环境
echo 'export PATH=$PATH:/usr/local/rabbitmq/sbin' >> /etc/profile
source /etc/profile

//配置目录
mkdir /etc/rabbitmq
```

配置文件：https://github.com/rabbitmq/rabbitmq-server/blob/master/deps/rabbit/docs/rabbitmq.conf.example

```
/etc/rabbitmq/rabbitmq.conf
```

```
# 监听端口
listeners.tcp.default = 5672
# web端口
management.tcp.port = 15672
management.tcp.ip   = 0.0.0.0
```

systemctl服务配置：

```
[Unit]
Description=rabbitmq
After=network.target

[Service]
Type=forking
User=root
Group=root
ExecStart=/usr/local/rabbitmq/sbin/rabbitmq-server -detached
ExecStop=/usr/local/rabbitmq/sbin/rabbitmqctl stop
PrivateTmp=true
[Install]
WantedBy=multi-user.target
```

## 4.2 启动测试

默认账号密码：guest guest（这个账号只允许本机访问） 

```
# 启动
rabbitmq-server -detached
# 停止
rabbitmqctl stop
# 状态
rabbitmqctl status
# web插件
rabbitmq-plugins enable rabbitmq_management
```

# 五、用户管理

```
# 查看所有用户
rabbitmqctl list_users
# 添加一个用户
rabbitmqctl add_user leellun 123456
# 配置权限
rabbitmqctl set_permissions -p "/" leellun ".*" ".*" ".*"
# 查看用户权限
rabbitmqctl list_user_permissions leellun
# 设置tag 加入administrator用户组
rabbitmqctl set_user_tags leellun administrator
# 删除用户（安全起见，删除默认用户）
rabbitmqctl delete_user guest
```

# 六、Virtual Hosts配置

vhost是rabbitmq分配权限的最小细粒度。 vhost中包含许多的exchanges，queues，bingdings 。

virtual host只是起到一个命名空间的作用，所以可以多个user共同使用一个virtual host，文章开头写的vritual_host = '/'，这个是系统默认的， 

```
# 查看所有虚拟主机
rabbitmqctl list_vhosts
# 添加vhost
rabbitmqctl add_vhost test_vhost
# 配置权限
rabbitmqctl set_permissions -p "/test_vhost" leellun ".*" ".*" ".*"
```

# 七、RabbitMQ Java 客户端

本库的5.x系列的编译和运行需要JDK 8。

RabbitMQ Java 客户端使用com.rabbitmq.client作为它的顶级包。关键的类和接口有：

- Channel: 代表 AMQP 0-9-1通道，并提供了大多数操作（协议方法）。

- Connection: 代表 AMQP 0-9-1 连接
- ConnectionFactory: 构建Connection实例
- Consumer: 代表消息的消费者
- DefaultConsumer: 消费者通用的基类
- BasicProperties: 消息的属性（元信息）
- BasicProperties.Builder: BasicProperties的构建器

## 7.1 常用api

```java
/**
* Channel
* 声明一个队列
* @param queue 队列名称
* @param durable 是否持久化，如果设置为true，服务器重启了队列仍然存在 
* @param exclusive 是否为独享队列（排他性队列），只有自己可见的队列，即不允许其它用户访问 
* @param autoDelete 当没有任何消费者使用时，自动删除该队列
* @param arguments 其他参数 
*/
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean 	autoDelete,Map<String, Object> arguments) throws IOException;   
```

如果exclusive声明为true，则该队列的特点是：

1、只对首次声明它的连接（Connection）可见  

2、会在其连接断开的时候自动删除。  一个线程用于消费，另一个线程在共享通道上推送是安全的。

```java
 /**
 * 限制服务器的数据量
 * @param prefetchSize  最大消息大小
 * @param prefetchCount RabbitMQ同时发送给消费者消息数量的最大值
 * @param global        是否使用整个通道
 */
void basicQos(int prefetchSize, int prefetchCount, boolean global) throws IOException;
/**
* 订阅消息并消费 
* @param queue 消息队列
* @param autoAck 是否自动应答
* @param callback 消息回调
*/
String basicConsume(String queue, boolean autoAck, Consumer callback) throws IOException;
 /**
* 发布一个消息
* @param exchange 指定转发器名称—-ExchangeName，这里用空字符串，就表示消息会交给默认的Exchange 
* @param routingKey 发布到哪个队列 
* @param props 和消息有关的其他配置参数，路由报头等 
* @param body 消息体
*/
void basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body) throws IOException;
/**
* 需要在每次处理完成一个消息后，手动向服务端发送一次应答。
* @param deliveryTag 当前消息的类似编号的号码，服务端为每一个消息生成的类似编号的号码 
* @param multiple 是否把小于当前deliveryTag的小于都应答了
*/
void basicAck(long deliveryTag, boolean multiple) throws IOException;
/**
* 删除队列
* @param queue 队列名称
* @param ifUnused 当未使用时，是否删除
* @param ifEmpty 当为empty时是否删除
*/
Queue.DeleteOk queueDelete(String queue, boolean ifUnused, boolean ifEmpty) throws IOException;
/**
* 清空消息
* @param queue 队列名称
*/
Queue.PurgeOk queuePurge(String queue) throws IOException;
```

## 7.2 maven配置

```xml
<dependencies>
    <dependency>
        <groupId>com.rabbitmq</groupId>
        <artifactId>amqp-client</artifactId>
        <version>5.9.0</version>
    </dependency>
	<!--rabbitmq依赖日志包-->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-log4j12</artifactId>
        <version>1.7.2</version>
    </dependency>
</dependencies>
```

## 7.3 连接和通道 的寿命

客户端connections是长连接。

Channels 虽然也是长期存活的，但是由于有大量的可恢复的协议错误会导致通道关闭，通道的存活期会比连接短一些。虽然每个操作都打开和关闭一个通道不是必须的操作，但是也不是不可行。有的选的情况下，还是优先考虑通道的复用为好。

已经关闭的通道不可以再被使用，也不会再接收到如消息投递之类的服务器事件。

## 7.4 由客户端提供的链接名称

为定位出现问题的应用或者实例 ，AMQP 0-9-1客户端链接可以提供一个自定义标识符，以遍在服务器日志 和管理界面中方便地对客户端进行区分。日志内容额管理界面中便会对标识符有所体现。标识符即为**客户端提供的连接名称**。名称可以用于标识应用或应用中特定的组件。 

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://userName:password@hostName:portNumber/virtualHost");
// 提供一个连接名称
Connection conn = factory.newConnection("app:audit component:event-consumer");
```

![1617592290829](RabbitMQ教程/1617592290829.png)

# 八、RabbitMQ的常见队列模型

## 8.1 直连模型

![20190420132307859](RabbitMQ教程/20190420132307859.png) 

生产者将消息发送到队列，消费者从队列中获取消息，队列是存储消息的缓冲区。。

生产者:

```java
public class SimplePublisher {
    public static final String QUEUE_NAME="sdfsdfsdfssf1";
    public static void main(String[] args) throws Exception{
        Connection connection=RabbitMQConnectionUtil.getConnection();
        Channel channel=connection.createChannel();
        channel.queueDeclare(QUEUE_NAME,true,false,false,null);
        String message="hello java!";
        channel.basicPublish("",QUEUE_NAME,null,message.getBytes());
    }
}
```

消费者:

```java
public class SimpleConsumer {
    public static final String QUEUE_NAME = "sdfsdfsdfssf1";

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.basicConsume(QUEUE_NAME, true, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("message receive:" + new String(body));
            }
        });
    }
}
```

## 8.2 work模式

多个消费者监听同一队列。消费者接收到消息后， 通过线程池异步消费。但是一个消息只能被一个消费者获取。

![20190420132651740](RabbitMQ教程/20190420132651740.png) 

生产者：

```java
public class WorkPublisher {
    public static final String QUEUE_NAME="workqueue";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        for(int i=0;i<10;i++){
            channel.basicPublish("",QUEUE_NAME,null,String.format("第%d条信息",i).getBytes());
            Thread.sleep(200);
        }
        channel.close();
        connection.close();
    }
}
```

消费者1:

```java
public class WorkConsumer {
    public static final String QUEUE_NAME="workqueue";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        // 同时最大处理的消息数量
        channel.basicQos(2);
        channel.basicConsume(QUEUE_NAME,true,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("消费者:"+new String(body));
            }
        });
    }
}
```

消费者2：

```java
public class WorkConsumer {
    public static final String QUEUE_NAME="workqueue";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        // 同时最大处理的消息数量
        channel.basicQos(2);
        //autoAck因为false，而最大的prefetchCount为2，所以这里只会打印2条消息
        channel.basicConsume(QUEUE_NAME,false,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("消费者:"+new String(body));
            }
        });
    }
}
```

## 8.3 订阅模式-fanout

![2019042013315438](RabbitMQ教程/2019042013315438.png) 

1） 可以有多个消费者
2） 每个消费者有自己的queue（队列）
3） 每个队列都要绑定到Exchange（交换机）
4） 生产者发送的消息，只能发送到交换机，交换机来决定要发给哪个队列，生产者无法决定。
5） 交换机把消息发送给绑定过的所有队列
6） 队列的消费者都能拿到消息。实现一条消息被多个消费者消费

生产者：

```java
public class FanoutPublisher {
    public static final String EXCHANGE="fanout_exchange";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.basicPublish(EXCHANGE,"",null,"这是一条fanout消息".getBytes());
        channel.close();
        connection.close();
    }
}
```

消费者:

```java
public class FanoutConsumer {
    public static final String EXCHANGE="fanout_exchange";
    public static final String QUEUE1="fanout_queue1";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE1,false,false,true,null);
        channel.exchangeDeclare(EXCHANGE,"fanout",true);
        channel.queueBind(QUEUE1,EXCHANGE,"");
        channel.basicConsume(QUEUE1,true,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("消费者1:"+new String(body));
            }
        });
    }
}
```

## 8.4 订阅模式-direct

![20190420133237576](RabbitMQ教程/20190420133237576.png) 

在Fanout模式中，一条消息，会被所有订阅的队列都消费。但是，在某些场景下，我们希望不同的消息被不同的队列消费。这时就要用到Direct类型的Exchange。

在Direct模型下：

队列与交换机的绑定，不能是任意绑定了，而是要指定一个RoutingKey（路由key）
消息的发送方在 向 Exchange发送消息时，也必须指定消息的 RoutingKey。
Exchange不再把消息交给每一个绑定的队列，而是根据消息的Routing Key进行判断，只有队列的Routingkey与消息的 Routing key完全一致，才会接收到消息

生产者：

```java
public class DirectPublisher {
    public static final String EXCHANGE="direct_exchange";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.basicPublish(EXCHANGE,"insert",null,"这是一条direct insert消息".getBytes());
        channel.basicPublish(EXCHANGE,"update",null,"这是一条direct update消息".getBytes());
        channel.basicPublish(EXCHANGE,"delete",null,"这是一条direct delete消息".getBytes());
        channel.close();
        connection.close();
    }
}
```

消费者1：

```java
public class DirectConsumer {
    public static final String EXCHANGE="direct_exchange";
    public static final String QUEUE1="direct_insert_delete";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE1,false,false,true,null);
        //绑定交换机和队列并指定RoutingKey为insert
        channel.queueBind(QUEUE1,EXCHANGE,"insert");
        //绑定交换机和队列并指定RoutingKey为delete
        channel.queueBind(QUEUE1,EXCHANGE,"delete");
        channel.basicConsume(QUEUE1,true,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("INSERT/DELETE:"+new String(body));
            }
        });
    }
}
```

消费者1打印消息

```
INSERT/DELETE:这是一条direct insert消息
INSERT/DELETE:这是一条direct delete消息
```



消费者2:

```java
public class DirectConsumer2 {
    public static final String EXCHANGE="direct_exchange";
    public static final String QUEUE1="direct_update";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE1,false,false,true,null);
        channel.queueBind(QUEUE1,EXCHANGE,"update");
        channel.basicConsume(QUEUE1,true,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("update:"+new String(body));
            }
        });
    }
}
```

消费者2打印消息

```
update:这是一条direct update消息
```

## 8.5 订阅模式-topic

![20190420133447136](RabbitMQ教程/20190420133447136.png) 

Topic类型的Exchange与Direct相比，都是可以根据RoutingKey把消息路由到不同的队列。只不过Topic类型Exchange可以让队列在绑定Routing key 的时候使用通配符！

通配符规则：

 \#：匹配一个或多个词
 *：匹配不多不少恰好1个词

举例：
 audit.#：能够匹配audit.irs.corporate`或者 audit.irs
 audit.*：只能匹配audit.irs

生产者:

```java
public class TopicPublisher {
    public static final String EXCHANGE="topic_exchange";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE,"topic",true);
        channel.basicPublish(EXCHANGE,"item.delete.product",null,"topic类型".getBytes());
        channel.close();
        connection.close();
    }
}
```

消费者：

```java
public class TopicConsumer {
    public static final String EXCHANGE="topic_exchange";
    public static final String QUEUE1="topic_queue";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE1,false,false,true,null);
        channel.queueBind(QUEUE1,EXCHANGE,"item.#");
        channel.basicConsume(QUEUE1,true,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("RECEIVERD:"+new String(body));
            }
        });
    }
}
```

## 8.6 headers模式

根据消息的headers来匹配对应的队列，在消息接收回调中指定headers， 可以是Map<String, Object>、String可变数组类型的keys等 

```java
public class HeadersPublisher {
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        Map<String,Object> headers=new HashMap<String, Object>();
        headers.put("test","testheaders1");
        AMQP.BasicProperties props=new AMQP.BasicProperties.Builder().headers(headers).build();
        channel.basicPublish("","headersqueue",props,"headers类型".getBytes());
        channel.close();
        connection.close();
    }
}
public class HeadersConsumer {
    public static final String QUEUE1="headersqueue";
    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        Map<String,Object> headers=new HashMap<String, Object>();
        headers.put("test","testheaders");
        channel.queueDeclare(QUEUE1,true,false,false,headers);
        channel.basicConsume(QUEUE1,true,headers,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("INSERT/DELETE:"+new String(body));
            }
        });
    }
}
```

# 九、存储机制

一个队列的内部存储其实是由5个子队列来流转运作的，队列中的消息可以有4种不同的状态。

RabbitMQ存储层包含两个部分：队列索引和消息存储。

RabbitMQ消息有两种类型：持久化消息和非持久化消息，这两种消息都会被写入磁盘。
持久化消息在到达队列时写入磁盘，同时会内存中保存一份备份，当内存吃紧时，消息从内存中清除。这会提高一定的性能。非持久化消息一般只存于内存中，当内存吃紧时会被换入磁盘，以节省内存空间。

![img](RabbitMQ教程/1323383-20200730003807268-172253195.png)

## 9.1 队列索引：rabbit_queue_index（下文简称index）

index维护队列的落盘消息的信息，如存储地点、是否已被交付给消费者、是否已被消费者ack等。每个队列都有相对应的index。

index使用顺序的段文件来存储，后缀为.idx，文件名从0开始累加，每个段文件中包含固定的segment_entry_count条记录，默认值是16384。每个index从磁盘中读取消息的时候，至少要在内存中维护一个段文件，所以设置queue_index_embed_msgs_below值得时候要格外谨慎，一点点增大也可能会引起内存爆炸式增长。

## 9.2 消息存储：rabbit_msg_store（下文简称store）

store以键值的形式存储消息，所有队列共享同一个store，每个节点有且只有一个。
从技术层面上说，store还可分为msg_store_persistent和msg_store_transient，前者负责持久化消息的持久化，重启后消息不会丢失；后者负责非持久化消息的持久化，重启后消息会丢失。通常情况下，两者习惯性的被当作一个整体。

store使用文件来存储，后缀为.rdq，经过store处理的所有消息都会以追加的方式写入到该文件中，当该文件的大小超过指定的限制（file_size_limit）后，将会关闭该文件并创建一个新的文件以供新的消息写入。文件名从0开始进行累加。在进行消息的存储时，RabbitMQ会在ETS（Erlang Term Storage）表中记录消息在文件中的位置映射和文件的相关信息。

消息（包括消息头、消息体、属性）可以直接存储在index中，也可以存储在store中。最佳的方式是较小的消息存在index中，而较大的消息存在store中。这个消息大小的界定可以通过queue_index_embed_msgs_below来配置，默认值为4096B。当一个消息小于设定的大小阈值时，就可以存储在index中，这样性能上可以得到优化（可理解为数据库的覆盖索引和回表）。
读取消息时，先根据消息的ID（msg_id）找到对应存储的文件，如果文件存在并且未被锁住，则直接打开文件，从指定位置读取消息内容。如果文件不存在或者被锁住了，则发送请求由store进行处理。

删除消息时，只是从ETS表删除指定消息的相关信息，同时更新消息对应的存储文件和相关信息。在执行消息删除操作时，并不立即对文件中的消息进行删除，也就是说消息依然在文件中，仅仅是标记为垃圾数据而已。当一个文件中都是垃圾数据时可以将这个文件删除。当检测到前后两个文件中的有效数据可以合并成一个文件，并且所有的垃圾数据的大小和所有文件（至少有3个文件存在的情况下）的数据大小的比值超过设置的阈值garbage_fraction（默认值0.5）时，才会触发垃圾回收，将这两个文件合并，执行合并的两个文件一定是逻辑上相邻的两个文件。

# 十、队列结构

通常队列由rabbit_amqqueue_process和backing_queue这两部分组成，rabbit_amqqueue_process负责协议相关的消息处理，即接收生产者发布的消息、向消费者交付消息、处理消息的确认（包括生产端的confirm和消费端的ack）等。backing_queue是消息存储的具体形式和引擎，并向rabbit_amqqueue_process提供相关的接口以供调用。

如果消息投递的目的队列是空的，并且有消费者订阅了这个队列，那么该消息会直接发送给消费者，不会经过队列这一步。当消息无法直接投递给消费者时，需要暂时将消息存入队列，以便重新投递。

RabbitMQ的队列消息有4种状态：

- alpha：消息索引和消息内容都存内存，最耗内存，很少消耗CPU
- beta：消息索引存内存，消息内容存磁盘
- gama：消息索引内存和磁盘都有，消息内容存磁盘
- delta：消息索引和内容都存磁盘，基本不消耗内存，消耗更多CPU和I/O操作

消息存入队列后，不是固定不变的，它会随着系统的负载在队列中不断流动，消息的状态会不断发送变化。持久化的消息，索引和内容都必须先保存在磁盘上，才会处于上述状态中的一种，gama状态只有持久化消息才会有的状态。

在运行时，RabbitMQ会根据消息传递的速度定期计算一个当前内存中能够保存的最大消息数量（target_ram_count），如果alpha状态的消息数量大于此值，则会引起消息的状态转换，多余的消息可能会转换到beta、gama或者delta状态。区分这4种状态的主要作用是满足不同的内存和CPU需求。

对于普通没有设置优先级和镜像的队列来说，backing_queue的默认实现是rabbit_variable_queue，其内部通过5个子队列Q1、Q2、delta、Q3、Q4来体现消息的各个状态。

![img](RabbitMQ教程/1323383-20200730004139693-1457985890.png)

消费者获取消息也会引起消息的状态转换。

- 当消费者获取消息时，首先会从Q4中获取消息，如果获取成功则返回。
- 如果Q4为空，则尝试从Q3中获取消息，系统首先会判断Q3是否为空，如果为空则返回队列为空，即此时队列中无消息。
- 如果Q3不为空，则取出Q3中的消息，进而再判断此时Q3和Delta中的长度，如果都为空，则可以认为 Q2、Delta、 Q3、Q4 全部为空，此时将Q1中的消息直接转移至Q4，下次直接从 Q4 中获取消息。
- 如果Q3为空，Delta不为空，则将Delta的消息转移至Q3中，下次可以直接从Q3中获取消息。在将消息从Delta转移到Q3的过程中，是按照索引分段读取的，首先读取某一段，然后判断读取的消息的个数与Delta中消息的个数是否相等，如果相等，则可以判定此时Delta中己无消息，则直接将Q2和刚读取到的消息一并放入到Q3中，如果不相等，仅将此次读取到的消息转移到Q3。

通常在负载正常时，如果消费速度大于生产速度，对于不需要保证可靠不丢失的消息来说，极有可能只会处于alpha状态。对于持久化消息，它一定会进入gamma状态，在开启publisher confirm机制时，只有到了gamma 状态时才会确认该消息己被接收，若消息消费速度足够快、内存也充足，这些消息也不会继续走到下一个状态。

在系统负载较高时，消息若不能很快被消费掉，这些消息就会进入到很深的队列中去，这样会增加处理每个消息的平均开销。因为要花更多的时间和资源处理“堆积”的消息，如此用来处理新流入的消息的能力就会降低，使得后流入的消息又被积压到很深的队列中，继续增大处理每个消息的平均开销，继而情况变得越来越恶化，使得系统的处理能力大大降低。

要避免流控机制触发，服务端默认配置是当内存使用达到40%，磁盘空闲空间小于50M，即启动内存报警，磁盘报警；报警后服务端触发流控（flowcontrol）机制。

一般地，当发布端发送消息速度快于订阅端消费消息的速度时，队列中堆积了大量的消息，导致报警，就会触发流控机制。
触发流控机制后，RabbitMQ服务端接收发布来的消息会变慢，使得进入队列的消息减少；
与此同时RabbitMQ服务端的消息推送也会受到极大的影响，测试发现，服务端推送消息的频率会大幅下降，等待下一次推送的时间，有时等1分钟，有时5分钟，甚至30分钟。

一旦触发流控，将导致RabbitMQ服务端性能恶化，推送消息也会变得非常缓慢；
因此要做好数据设计，使得发送速率和接收速率保持平衡，而不至于引起服务器堆积大量消息，进而引发流控。通过增加服务器集群节点，增加消费者，来避免流控发生，治标不治本，而且成本高。

应对这一问题一般有3种措施：

1. 增加prefetch_count的值，即一次发送多条消息给消费者，加快消息被消费的速度。
2. 采用multiple ack，降低处理 ack 带来的开销
3. 流量控制

# 十一、案例

rabbitmq练习：

https://github.com/leelun/rabbitmq-examples

Spring RabbitMQ结合：

https://github.com/leelun/springmvc-learn

Spring Boot RabbitMQ结合：

https://github.com/leelun/springboot-learn

Spring Cloud RabbitMQ结合：

https://github.com/leelun/springcloud-learn

