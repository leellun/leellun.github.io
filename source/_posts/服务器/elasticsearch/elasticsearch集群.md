---
title: elasticsearch集群搭建
date: 2021-06-22 18:14:02
categories:
  - 服务器
  - elasticsearch
tags:
  - elasticsearch
---

# 一、简介

每台主机称作一个节点（Node）。

如图就是一个三节点的集群：

![image-20200910111009285](elasticsearch集群/20200910111010.png)

在图中，每个 Node 都有三个分片，其中 P 开头的代表 Primary 分片，即主分片，R 开头的代表 Replica  分片，即副本分片。所以图中主分片 1、2，副本分片 0 储存在 1 号节点，副本分片 0、1、2 储存在 2 号节点，主分片 0 和副本分片  1、2 储存在 3 号节点，一共是 3 个主分片和 6 个副本分片。同时我们还注意到 1 号节点还有个 MASTER  的标识，这代表它是一个主节点，它相比其他的节点更加特殊，它有权限控制整个集群，比如资源的分配、节点的修改等等。

这里就引出了一个概念就是节点的类型，我们可以将节点分为这么四个类型：

- 主节点：即 Master  节点。主节点的主要职责是和集群操作相关的内容，如创建或删除索引，跟踪哪些节点是群集的一部分，并决定哪些分片分配给相关的节点。稳定的主节点对集群的健康是非常重要的。默认情况下任何一个集群中的节点都有可能被选为主节点。索引数据和搜索查询等操作会占用大量的cpu，内存，io资源，为了确保一个集群的稳定，分离主节点和数据节点是一个比较好的选择。虽然主节点也可以协调节点，路由搜索和从客户端新增数据到数据节点，但最好不要使用这些专用的主节点。一个重要的原则是，尽可能做尽量少的工作。
- 数据节点：即 Data 节点。数据节点主要是存储索引数据的节点，主要对文档进行增删改查操作，聚合操作等。数据节点对 CPU、内存、IO 要求较高，在优化的时候需要监控数据节点的状态，当资源不够的时候，需要在集群中添加新的节点。
- 负载均衡节点：也称作 Client  节点，也称作客户端节点。当一个节点既不配置为主节点，也不配置为数据节点时，该节点只能处理路由请求，处理搜索，分发索引操作等，从本质上来说该客户节点表现为智能负载平衡器。独立的客户端节点在一个比较大的集群中是非常有用的，他协调主节点和数据节点，客户端节点加入集群可以得到集群的状态，根据集群的状态可以直接路由请求。
- 预处理节点：也称作 Ingest 节点，在索引数据之前可以先对数据做预处理操作，所有节点其实默认都是支持 Ingest 操作的，也可以专门将某个节点配置为 Ingest 节点。

以上就是节点几种类型，一个节点其实可以对应不同的类型，如一个节点可以同时成为主节点和数据节点和预处理节点，但如果一个节点既不是主节点也不是数据节点，那么它就是负载均衡节点。

## 1.1 主节点

```
node.master: true # 标志节点作为主节点
```

## 1.2 数据节点

```
node.data: true
```

## 1.3 负载均衡节点

该node服务器即不会被选作主节点，也不会存储任何索引数据。该服务器主要用 于查询负载均衡。在查询的时候，通常会涉及到从多个node服务器上查询数据，并请 求分发到多个指定的node服务器，并对各个node服务器返回的结果进行一个汇总处理， 最终返回给客户端。

```
node.master: false 
node.data: false
```

## 1.4 Ingest 节点

https://juejin.cn/post/6844903873153335309

ingest 节点是数据前置处理转换的节点，支持 pipeline管道设置，可以使用 ingest 对数据进行过滤、转换等操作。

原理：在实际文档索引发生之前，使用Ingest节点预处理文档。Ingest节点拦截批量和索引请求，它应用转换，然后将文档传递回索引或Bulk API。

```
node.ingest：false
```

Ingest实践：

（1）在数据写入阶段修改字段名

借助Ingest节点的 rename_hostname管道的预处理功能，实现了字段名称的变更：由hostname改成host。

```
PUT _ingest/pipeline/rename_hostname
{
  "processors": [
    {
        "field": "hostname",
        "target_field": "host",
        "ignore_missing": true
      }
    }
  ]
}

PUT server
POST server/values/?pipeline=rename_hostname
{
  "hostname": "myserver"
}
```

（2）在批量写入数据的时候，每条document插入实时时间戳

通过indexed*at管道的set处理器与ms-test的索引层面关联操作， ms-test索引每插入一篇document，都会自动添加一个字段index*at=最新时间戳。

```
PUT _ingest/pipeline/indexed_at
{
  "description": "Adds indexed_at timestamp to documents",
  "processors": [
    {
      "set": {
        "field": "_source.indexed_at",
        "value": "{{_ingest.timestamp}}"
      }
    }
  ]
}
PUT ms-test
{
  "settings": {
    "index.default_pipeline": "indexed_at"
  }
}
POST ms-test/_doc/1
{"title":"just testing"}
```

## 1.5 组合方式

组合一：

该node服务器只作为一个数据节点，只用于存储索引数据。使该node服务器功能单一，只用于数据存储和数据查询，降低其资源消耗率。

```yaml
node.master: false 
node.data: true
```

组合二：

该node服务器只作为一个主节点，但不存储任何索引数据。该node服务器将使用 自身空闲的资源，来协调各种创建索引请求或者查询请求，讲这些请求合理分发到相关 的node服务器上。

```yaml
node.master: true 
node.data: false
```

组合三：

该node服务器即不会被选作主节点，也不会存储任何索引数据。该服务器主要用 于查询负载均衡。在查询的时候，通常会涉及到从多个node服务器上查询数据，并请 求分发到多个指定的node服务器，并对各个node服务器返回的结果进行一个汇总处理， 最终返回给客户端。

```yaml
node.master: false 
node.data: false
```

组合四：

这种组合表示这个节点即有成为主节点的资格，又存储数据，这个时候如果某个节点被选举成为了真正的主节点，那么他还要存储数据，这样对于这个节点的压力就比较大了。elasticsearch默认每个节点都是这样的配置，在测试环境下这样做没问题。实际工作中建议不要这样设置，这样相当于主节点和数据节点的角色混合到一块了。

```yaml
node.master: true 
node.data: true
```

# 二、集群搭建

学习配置方案 3node——3 master——3 data ，为了快速学习的简单配置，正式环境不建议，最好master node做分离。当达到一定的集群数目是最好还单独独立出来一个负载均衡节点+ingest节点

## 2.1 创建数据目录和日志目录

```sh
mkdir /opt/es/data1
mkdir /var/log/es/log1
```

## 2.2 配置文件

集群配置中最重要的两项是`node.name`与`network.host`，每个节点都必须不同。其中`node.name`是节点名称主要是在Elasticsearch自己的日志加以区分每一个节点信息。`discovery.seed_hosts`是集群中的节点信息，可以使用IP地址、可以使用主机名(必须可以解析)

| 节点   | ip            | 角色        |
| ------ | ------------- | ----------- |
| node-1 | 192.168.66.11 | master data |
| node-2 | 192.168.66.21 | master data |
| node-3 | 192.168.66.22 | master data |

编辑 `elasticsearch.yml`

高版本：

```
roles [transform, data_frozen, master, remote_cluster_client, data, ml, data_content, data_hot, data_warm, data_cold, ingest]
```

```
cluster.name: my-application  # 集群名
node.name: node-1 #节点名
node.master: true # 是否主节点
node.data: true #是否数据节点
path.data: /opt/es/data1 #数据
path.logs: /var/log/es/log1 #日志
network.host: 192.168.66.11 # 绑定host
http.port: 9200 # 端口
transport.port: 9300 # 集群内部端口
transport.compress: true #压缩开启
discovery.seed_hosts: ["192.168.66.11","192.168.66.21","192.168.66.22"] # 集群扫描
cluster.initial_master_nodes: ["node-1","node-2","node-3"] # 初始化master节点 启动的时候会执行一次
http.cors.enabled: true # 跨域支持
http.cors.allow-origin: "*"
```

node-2（变动配置）：

```
node.name: node-2
network.host: 192.168.66.21
```

node-3（变动配置）：

```
node.name: node-3
network.host: 192.168.66.22
```

可能会出现geoip-database的错误，这里加入这个配置可以解决

```
ingest.geoip.downloader.enabled: false
```

配置的时候注意防火墙

## 2.3 伪集群方案

一个node，只要内存允许可以防止n个es节点

节点1：

```
cluster.name: my-application
node.name: node-1
node.master: true
node.data: true
path.data: /opt/es/data1
path.logs: /var/log/es/log1
network.host: 192.168.66.11
http.port: 9201
transport.port: 9301
transport.compress: true
discovery.seed_hosts: ["192.168.66.11:9300","192.168.66.11:9301","192.168.66.11:9301"....,"192.168.66.11:930n"] 
cluster.initial_master_nodes: ["node-1","node-2","node-3"...,"node-n"] 
http.cors.enabled: true 
http.cors.allow-origin: "*"
```

节点n：

```
node.name: node-[n]
path.data: /opt/es/data[n]
path.logs: /var/log/es/log[n]
http.port: 920[n]
transport.port: 930[n]
```

## 2.4 启动

```
elasticsearch -d
```

## 2.5 接口测试

```
GET  /_cat
/_cat/health
/_cat/nodes
/_cat/master
/_cat/indices
/_cat/allocation 
/_cat/shards 
/_cat/shards/{index}
/_cat/thread_pool
/_cat/segments 
/_cat/segments/{index}
```

## 2.6 helm 集群安装

```
helm repo add elastic https://helm.elastic.co
```



# 三、集群分析

## 3.1 需要多大规模的集群？

- 当前的数据量有多大？数据增长情况如何？
- 你的机器配置如何？cpu、多大内存、多大硬盘容量？

**推算的依据：**

ES JVM heap 最大可以设置32G 。
30G heap 大概能处理的数据量 10 T。如果内存很大如128G，可在一台机器上运行多个ES节点实例。

备注：集群规划满足当前数据规模+适量增长规模即可，后续可按需扩展。

**两类应用场景：**

A. 用于构建业务搜索功能模块，且多是垂直领域的搜索。数据量级几千万到数十亿级别。一般2-4台机器的规模。
B. 用于大规模数据的实时OLAP（联机处理分析），经典的如ELK Stack，数据规模可能达到千亿或更多。几十到上百节点的规模。

## 3.2 集群中的节点角色如何分配？

可以有主节点、数据节点、协调节点

一个节点可以充当一个或多个角色，默认三个角色都有

协调节点：一个节点只作为接收请求、转发请求到其他节点、汇总各个节点返回数据等功能的节点。就叫协调节点

 如何分配：

- A. 小规模集群，不需严格区分。
- B. 中大规模集群（十个以上节点），应考虑单独的角色充当。特别并发查询量大，查询的合并量大，可以增加独立的协调节点。角色分开的好处是分工分开，不互影响。如不会因协调角色负载过高而影响数据节点的能力。

## 3.3 避免脑裂

5.x版本：

```
discovery.zen.minimum_master_nodes: (有master资格节点数/2) + 1 

discovery.zen.ping.multicast.enabled: false —— 关闭多播发现机制，默认是关闭的

discovery.zen.ping.unicast.hosts: ["master1", "master2", "master3"] —— 配置单播发现的主节点ip地址，其他从节点要加入进来，就得去询问单播发现机制里面配置的主节点我要加入到集群里面了，主节点同意以后才能加入，然后主节点再通知集群中的其他节点有新节点加入

discovery.zen.ping_timeout: 30（默认值是3秒）——其他节点ping主节点多久时间没有响应就认为主节点不可用了
discovery.zen.minimum_master_nodes: 2 —— 选举主节点时需要看到最少多少个具有master资格的活节点，才能进行选举
```

7+.x版本：

```
discovery.seed_hosts: ["192.168.66.11:9300","192.168.66.11:9301","192.168.66.11:9301"....,"192.168.66.11:930n"] 
cluster.initial_master_nodes: ["node-1","node-2","node-3"...,"node-n"] 
```

## 3.4 索引应该设置多少个分片？

说明：分片数指定后不可变，除非重索引。分片对应的存储实体是索引，分片并不是越多越好

分片多浪费存储空间、占用资源、影响性能

**分片过多的影响：**

- 每个分片本质上就是一个Lucene索引, 因此会消耗相应的文件句柄, 内存和CPU资源。
- 每个搜索请求会调度到索引的每个分片中. 如果分片分散在不同的节点倒是问题不太. 但当分片开始竞争相同的硬件资源时, 性能便会逐步下降。
- ES使用词频统计来计算相关性. 当然这些统计也会分配到各个分片上. 如果在大量分片上只维护了很少的数据, 则将导致最终的文档相关性较差。

**分片设置的可参考原则：**

- ElasticSearch推荐的最大JVM堆空间是30~32G, 所以把你的分片最大容量限制为30GB, 然后再对分片数量做合理估算. 例如, 你认为你的数据能达到200GB, 推荐最多分配7到8个分片。
- 在开始阶段, 一个好的方案是根据你的节点数量按照1.5~3倍的原则来创建分片. 例如,如果你有3个节点, 则推荐你创建的分片数最多不超过9(3x3)个。当性能下降时，增加节点，ES会平衡分片的放置。
- 对于基于日期的索引需求, 并且对索引数据的搜索场景非常少. 也许这些索引量将达到成百上千, 但每个索引的数据量只有1GB甚至更小. 对于这种类似场景, 建议只需要为索引分配1个分片。如日志管理就是一个日期的索引需求，日期索引会很多，但每个索引存放的日志数据量就很少。

## 3.5 分片应该设置几个副本？

说明：副本数是可以随时调整的！

副本的用途：备份数据保证高可用数据不丢失，高并发的时候参与数据查询，一般一个分片有1-2个副本即可保证高可用。

集群规模没变的情况下副本过多会有什么影响？

副本多浪费存储空间、占用资源、影响性能

**副本设置基本原则：**

- 为保证高可用，副本数设置为2即可。要求集群至少要有3个节点，来分开存放主分片、副本。
- 如发现并发量大时，查询性能会下降，可增加副本数，来提升并发查询能力。

注意：新增副本时主节点会自动协调，然后拷贝数据到新增的副本节点

参考文章：

https://learnku.com/articles/40400

https://www.cnblogs.com/leesmall/p/9220535.html

https://www.cnblogs.com/sparkdev/p/11044662.html

https://blog.csdn.net/zuodaoyong/article/details/104719508

https://www.cnblogs.com/rsapaper/p/9675225.html

# 四、elk案例

## 4.1 日志同步

- 代码输出日志

  ```java
      @Test
      public void contextLoads() throws InterruptedException {
          while (true) {
              Random random = new Random();
              int i = random.nextInt(1000);
              if (i % 2 == 0) {
                  log.info("我是info，当前生成的随机数是 {}", i);
              } else {
                  log.error("我是error，当前生成的随机数是 {}", i);
              }
              if (i == 99) {
                  break;
              }
              Thread.sleep(100);
          }
      }
  ```

  

- logstash同步到ES

  ```sh
  input {
    file {
      path => "H:/es/**/*.log"
      start_position => beginning
      #增加标签
      stat_interval => 1
    }
  }
  
  filter {
    grok {
      match => ["message", "%{TIMESTAMP_ISO8601:log_time}\ %{GREEDYDATA:level}\ \[%{GREEDYDATA:thread}\]\ \[%{GREEDYDATA:app_name}\]\ %{GREEDYDATA:package_name}\:%{NUMBER:code_line}\ \-%{GREEDYDATA:info}"]
    }
    mutate {
      gsub => ["info", "\r", ""]
      remove_field => ["message", "@timestamp", "host"]
    }
  }
  
  output {
    stdout {
      codec => rubydebug
    }
    elasticsearch {
      hosts => ["39.102.41.53:9200"]
      index => "log-%{+YYYY.MM.dd}"
    }
  }
  ```

  

- kibana显示

## 4.2 mysql数据同步

将数据同步到ES

- 代码持续生成数据存库

- 配置logstash

  ```sh
  input {
   jdbc {
  	  jdbc_connection_string => "jdbc:mysql://127.0.0.1:3306/article?characterEncoding=UTF8"
  	  jdbc_user => "root"
  	  jdbc_password => "yangdeshi"
  	  jdbc_driver_library => "H:/logstash-7.3.0/config/mysql-connector-java-5.1.47.jar"
  	  jdbc_driver_class => "com.mysql.jdbc.Driver"
  	  #使用其它字段追踪，而不是用时间
      use_column_value => true
      #追踪的字段
      tracking_column => id
      record_last_run => true
      #上一个sql_last_value值的存放文件路径, 必须要在文件中指定字段的初始值
      #指定文件,来记录上次执行到的 tracking_column 字段的值
  	  #比如上次数据库有 10000 条记录,查询完后该文件中就会有数字 10000 这样的记录,下次执行 SQL 查询可以从 10001 条处开始.
  	  #我们只需要在 SQL 语句中 WHERE MY_ID > :sql_last_value 即可. 其中 :sql_last_value 取得就是该文件中的值(10000).
      last_run_metadata_path => "H:/logstash-7.3.0/config/station_parameter"
  	  #是否开启分页
  	  #jdbc_paging_enabled => "true"
  	  #每页条数
  	  #jdbc_page_size => "100"
  	  #以下对应着要执行的sql的绝对路径。
  	  #sql太长写在文件中，这里指定路径statement_filepath => "文件路径"
  	  statement_filepath => "H:/logstash-7.3.0/config/article.sql"
  	  #定时执行，最小一分钟一次
  	  schedule => "* * * * *"
    }
  }
  
  output {
    stdout {
      codec => rubydebug
    }
    elasticsearch {
      hosts => ["39.102.41.53:9200"]
      index => "article-%{+YYYY.MM.dd}"
    }
  }
  ```

- 查看同步情况

```
helm install elasticsearch ../../elasticsearch --namespace=gulimall
```

