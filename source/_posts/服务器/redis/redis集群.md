---
title: redis集群
date: 2020-01-22 20:14:02
categories:
  - 服务器
  - redis
tags:
  - redis 
---

# redis集群

# 一、介绍

Redis 集群使用数据分片（sharding）而非一致性哈希（consistency hashing）来实现： 一个 Redis 集群包含 16384 个哈希槽（hash slot）， 数据库中的每个键都属于这 16384 个哈希槽的其中一个， 集群使用公式 CRC16(key) % 16384 来计算键 key 属于哪个槽， 其中 CRC16(key) 语句用于计算键 key 的 CRC16 校验和 。

集群中的每个节点负责处理一部分哈希槽。 举个例子， 一个集群可以有三个哈希槽， 其中：

- 节点 A 负责处理 `0` 号至 `5500` 号哈希槽。
- 节点 B 负责处理 `5501` 号至 `11000` 号哈希槽。
- 节点 C 负责处理 `11001` 号至 `16384` 号哈希槽。

## 二、Redis 集群中的主从复制

为了使得集群在一部分节点下线或者无法与集群的大多数（majority）节点进行通讯的情况下， 仍然可以正常运作， Redis 集群对节点使用了主从复制功能： 集群中的每个节点都有 `1` 个至 `N` 个复制品（replica）， 其中一个复制品为主节点（master）， 而其余的 `N-1` 个复制品为从节点（slave）。

在之前列举的节点 A 、B 、C 的例子中， 如果节点 B 下线了， 那么集群将无法正常运行， 因为集群找不到节点来处理 `5501` 号至 `11000` 号的哈希槽。

另一方面， 假如在创建集群的时候（或者至少在节点 B 下线之前）， 我们为主节点 B 添加了从节点 B1 ， 那么当主节点 B 下线的时候， 集群就会将 B1 设置为新的主节点， 并让它代替下线的主节点 B ， 继续处理 `5501` 号至 `11000` 号的哈希槽， 这样集群就不会因为主节点 B 的下线而无法正常运作了。

不过如果节点 B 和 B1 都下线的话， Redis 集群还是会停止运作。

## 三、Redis 集群的一致性保证（guarantee）

Redis 集群**不保证数据的强一致性**（strong consistency）： 在特定条件下， Redis 集群可能会丢失已经被执行过的写命令。

使用异步复制（asynchronous replication）是 Redis 集群可能会丢失写命令的其中一个原因。 考虑以下这个写命令的例子：

- 客户端向主节点 B 发送一条写命令。
- 主节点 B 执行写命令，并向客户端返回命令回复。
- 主节点 B 将刚刚执行的写命令复制给它的从节点 B1 、 B2 和 B3 。

如你所见， 主节点对命令的复制工作发生在返回命令回复之后， 因为如果每次处理命令请求都需要等待复制操作完成的话， 那么主节点处理命令请求的速度将极大地降低 —— 我们必须在性能和一致性之间做出权衡。

redis集群丢失命令的情况：

1 异步复制

2 集群出现网络分裂， 并且一个客户端与至少包括一个主节点在内的少数（minority）实例被孤立。

# 四、集群创建

哨兵模式：https://blog.csdn.net/m0_58292366/article/details/125706041

下载：

```
wget -c http://download.redis.io/releases/redis-5.0.0.tar.gz
tar -zxf redis-5.0.0.tar.gz
cd redis-5.0.0
make && make install
```

最少选项的集群配置文件示例：

```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```

文件中的 `cluster-enabled` 选项用于开实例的集群模式， 而 `cluster-conf-file` 选项则设定了保存节点配置文件的路径， 默认值为 `nodes.conf` 。

**要让集群正常运作至少需要三个主节点**， 不过在刚开始试用集群功能时， 强烈建议使用六个节点： 其中三个为主节点， 而其余三个则是各个主节点的从节点。

## 4.1 创建六个以端口号为名字的子目录

```
mkdir cluster-test
cd cluster-test
mkdir 7000 7001 7002 7003 7004 7005
```

## 4.2  启动

分别在7000 7001 7002 7003 7004 7005

```
port=7000
cd /home/redis/cluster-test/$port
cat > redis.conf <<END
port $port
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
masterauth 123456
requirepass 123456
END
redis-server ./redis.conf &
```

## 4.3 集群模式

```
redis-cli --cluster create --cluster-replicas 1 192.168.66.11:7000 192.168.66.11:7001 \
192.168.66.11:7002 192.168.66.11:7003 192.168.66.11:7004 192.168.66.11:7005 -a 123456
```

–cluster create: 创建集群

–cluster-relicas: 集群副本数。 这里是1，是1主机1从机的模式，如果设置为2（即：2台从机）会失败。因为集群中至少要有3个主机，所以设置2台从机时，至少需要9个节点才可以。

最后的参数中列出全部的redis主机IP地址和端口号。

-a: 后边跟的是redis的密码

结果显示：

```
13256:S 26 Aug 2022 15:05:06.448 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
13256:S 26 Aug 2022 15:05:06.448 * Background AOF rewrite finished successfull
```

master-slave查看

```
127.0.0.1:7001> cluster nodes
19a354f414924288a06c6b3ba24d0bf4affe8700 127.0.0.1:7000@17000 master - 0 1661497854077 1 connected 0-5460
1401b5c43f29453542d2c8d1c6ade3f2a3c9e146 127.0.0.1:7002@17002 master - 0 1661497854780 3 connected 10923-16383
d54c8707df777faab6cfa88a3b173bfd1c2e81bb 127.0.0.1:7004@17004 slave 19a354f414924288a06c6b3ba24d0bf4affe8700 0 1661497854000 1 connected
b4053809d6eedfbc72bb636b1439d1b32bb82fd3 127.0.0.1:7001@17001 myself,master - 0 1661497854000 2 connected 5461-10922
638e0107053358064fa6577b8cd3d82f6499d7b8 127.0.0.1:7003@17003 slave 1401b5c43f29453542d2c8d1c6ade3f2a3c9e146 0 1661497855284 4 connected
d898bda1abd3582078680805445b98f6c2a44cc2 127.0.0.1:7005@17005 slave b4053809d6eedfbc72bb636b1439d1b32bb82fd3 0 1661497855000 6 connected
```

## 4.4 客户端测试

```
redis-cli -c -p 7000
```

