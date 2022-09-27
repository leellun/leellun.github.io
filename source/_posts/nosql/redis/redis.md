---
title: redis基础学习文档
date: 2020-01-06 20:14:02
categories:
  - 服务器
  - redis
tags:
  - redis 
---

# 一、介绍

Redis:Remote DIctionary Server(远程字典服务器)，是完全开源免费的，用C语言编写的，遵守BSD协议，
是一个高性能的(key/value)分布式内存数据库，基于内存运行并支持持久化的NoSQL数据库，是当前最热门的NoSql数据库之一,也被人们称为数据结构服务器。

优点：

（1）Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用

（2）Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储

（3）Redis支持数据的备份，即master-slave模式的数据备份

相关应用：

（1）内存存储和持久化：redis支持异步将内存中的数据写到硬盘上，同时不影响继续服务

（2）取最新N个数据的操作，如：可以将最新的10条评论的ID放在Redis的List集合里面

（3）模拟类似于HttpSession这种需要设定过期时间的功能

（4）发布、订阅消息系统

（5）定时器、计数器

# 二、Redis的安装

下载安装

```
wget -c http://download.redis.io/releases/redis-5.0.0.tar.gz
make && make install
```

安装目录查看

查看默认安装目录：usr/local/bin

redis-benchmark:性能测试工具

redis-check-aof：修复有问题的AOF文件

redis-check-dump：修复有问题的dump.rdb文件

redis-cli：客户端，操作入口

redis-sentinel：redis集群使用

redis-server：Redis服务器启动命令

启动服务

（1）修改redis.conf文件将里面的daemonize no 改成 yes，让服务在后台启动

（2）将默认的redis.conf拷贝到自己定义好的一个路径下，比如/myconf

（3）通过命令redis-server /myconf启动服务

（4）redis-cli 连接测试

服务关闭

单实例关闭：redis-cli shutdown

多实例关闭，指定端口关闭:redis-cli -p 6379 shutdown

# 三、Redis服务

**服务核心**

单进程

单进程模型来处理客户端的请求。对读写等事件的响应是通过对epoll函数的包装来做到的。Redis的实际处理速度完全依靠主进程的执行效率。

epoll是Linux内核为处理大批量文件描述符而作了改进的epoll，是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。

**服务端操作**

默认16个数据库，类似数组下表从零开始，初始默认使用零号库。

设置数据库的数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id  databases 16。

默认端口是6379

**数据库操作**

select命令切换数据库：select  [index]

dbsize查看当前数据库的key的数量

flushdb：清空当前库

flushall：清空全部库

统一密码管理，16个库都是同样密码，要么都OK要么一个也连接不上

Redis索引都是从零开始

**更多操作**

http://redisdoc.com/

# 四、Redis数据类型

https://blog.csdn.net/weixin_40205234/article/details/124614720

Redis的五大数据类型:

（1）string（字符串）
（2）hash（哈希，类似java里的Map）
（3）list（列表）
（4）set（集合）
（5）zset(sorted set：有序集合)

```
String：如果存储数字的话，是用int类型的编码;如果存储非数字，小于等于39字节的字符串，是embstr；大于39个字节，则是raw编码。
List：如果列表的元素个数小于512个，列表每个元素的值都小于64字节（默认），使用ziplist编码，否则使用linkedlist编码
Hash：哈希类型元素个数小于512个，所有值小于64字节的话，使用ziplist编码,否则使用hashtable编码。
Set：如果集合中的元素都是整数且元素个数小于512个，使用intset编码，否则使用hashtable编码。
Zset：当有序集合的元素个数小于128个，每个元素的值小于64字节时，使用ziplist编码，否则使用skiplist（跳跃表）编码
```

## Redis 键(key)

**key值查看：**

```
（1）keys * 显示所有的key
（2）exists key 判断某个key是否存在
（3）move key db   将当前库的key移至db库
（4）expire key 秒钟  为给定的key设置过期时间
（5）ttl key 查看还有多少秒过期，-1表示永不过期，-2表示已过期
（6）type key 查看你的key是什么类型
（7）del key删除key
```

更多操作：http://redisdoc.com/database/index.html

## Redis字符串(String)

>
> string是redis最基本的类型，一个key对应一个value。
>
>
> string类型是二进制安全的。意思是redis的string可以包含任何数据。比如jpg图片或者序列化的对象 。
>
> string类型是Redis最基本的数据类型，一个redis中字符串value最多可以是512M。
>
> 应用场景：共享session、分布式锁，计数器、限流。
>
> 内部编码有3种，int（8字节长整型）/embstr（小于等于39字节字符串）/raw（大于39个字节字符串）

相关操作：

```
set/get/del/append/strlen  设置/获取/删除/追加/字符串长度
Incr/decr/incrby/decrby 数字操作，一定要是数字才能进行加减
getrange/setrange  返回指定范围内的字符串/指定范围字符串替换
setex(set with expire)键秒值/setnx(set if not exist)
mset/mget/msetnx 
getset(先get再set)
```

所有字符串类型操作：http://redisdoc.com/string/index.html

## Redis列表(List)

Redis 列表是简单的字符串列表，按照插入顺序排序。可以添加一个元素导列表的头部（左边）或者尾部（右边）。它的底层实际是个链表。

> 简介：列表（list）类型是用来存储多个有序的字符串，一个列表最多可以存储2^32-1个元素。
> 简单实用举例：lpush key value [value ...] 、lrange key start end
> 内部编码：ziplist（压缩列表）、linkedlist（链表）
> 应用场景：消息队列，文章列表

list应用场景参考以下：

```
lpush+lpop=Stack（栈）
lpush+rpop=Queue（队列）
lpsh+ltrim=Capped Collection（有限集合）
lpush+brpop=Message Queue（消息队列）
```

相关操作：

```
 lpush/rpush/lrange  插入表头/插入表尾/区间值返回
 lpop/rpop  移除并返回表头元素/移除并返回表尾元素
 lindex 按照索引下标获得元素(从上到下)
 llen 长度获取
 lrem 移除
 ltrim key 开始index 结束index，截取指定范围的值后再赋值给key
 rpoplpush 源列表 目的列表
 lset key index value
 linsert key  before/after 值1 值2
```


所有列表操作：http://redisdoc.com/list/index.html

## Redis集合(Set)

> 简介：集合（set）类型也是用来保存多个的字符串元素，但是不允许重复元素
> 简单使用举例：sadd key element [element ...]、smembers key
> 内部编码：intset（整数集合）、hashtable（哈希表）
> 注意点：smembers和lrange、hgetall都属于比较重的命令，如果元素过多存在阻塞Redis的可能性，可以使用sscan来完成。
> 应用场景：用户标签,生成随机数抽奖、社交需求。

相关操作：

```
 sadd/smembers/sismember
 scard，获取集合里面的元素个数
 srem key value 删除集合中元素
 srandmember key 某个整数(随机出几个数)
 spop key 随机出栈
 smove key1 key2 在key1里某个值      作用是将key1里的某个值赋给key2
差集：sdiff
交集：sinter
并集：sunion
```

所有Set集合操作：http://redisdoc.com/set/index.html

## Redis哈希(Hash)

> 简介：在Redis中，哈希类型是指v（值）本身又是一个键值对（k-v）结构
> 简单使用举例：hset key field value 、hget key field
> 内部编码：ziplist（压缩列表） 、hashtable（哈希表）
> 应用场景：缓存用户信息等。
> 注意点：如果开发使用hgetall，哈希元素比较多的话，可能导致Redis阻塞，可以使用hscan。而如果只是获取部分field，建议使用hmget。

相关操作：

 ```
  hset/hget/hmset/hmget/hgetall/hdel
  hlen
  hexists key 在key里面的某个值的key
  hkeys/hvals
  hincrby/hincrbyfloat
  hsetnx
 ```

所有Hash操作：http://redisdoc.com/hash/index.html

## Redis有序集合Zset(sorted set)

> - 简介：已排序的字符串集合，同时元素不能重复
> - 简单格式举例：`zadd key score member [score member ...]`，`zrank key member`
> - 底层内部编码：`ziplist（压缩列表）`、`skiplist（跳跃表）`
> - 应用场景：排行榜，社交需求（如用户点赞）

相关操作：

```
 zadd/zrange
 zrangebyscore key 开始score 结束score
 zrem key 某score下对应的value值，作用是删除元素
 zcard/zcount key score区间/zrank key values值，作用是获得下标值/zscore key 对应值,获得分数
 zrevrank key values值，作用是逆序获得下标值
 zrevrange
 zrevrangebyscore  key 结束score 开始score
```

所有操作：http://redisdoc.com/sorted_set/index.html

## Redis 的三种特殊数据类型

Geo：Redis3.2推出的，地理位置定位，用于存储地理位置信息，并对存储的信息进行操作。
HyperLogLog：用来做基数统计算法的数据结构，如统计网站的UV。
Bitmaps ：用一个比特位来映射某个元素的状态，在Redis中，它的底层是基于字符串类型实现的，可以把bitmaps成作一个以比特位为单位的数组

# 五、redis配置文件

https://www.cnblogs.com/zhang-ke/p/5981108.html

```properties
#配置大小单位,开头定义了一些基本的度量单位，只支持bytes，不支持bit
#单位大小写不敏感

###################################GENERAL通用###################################
#包含其它配置文件
include /path/to/local.conf
#是否在后台执行，yes：后台运行；no：不是后台运行
daemonize yes
#redis的进程文件
pidfile /var/run/redis_6379.pid
#指定 redis 只接收来自于该 IP 地址的请求，如果不进行设置，那么将处理所有请求
bind 127.0.0.1
#redis监听的端口号。
port 6379
# 此参数为设置客户端空闲超过timeout，服务端会断开连接，为0则服务端不会主动断开连接，不能小于0。
timeout 0
#指定了服务端日志的级别。级别包括：debug（很多信息，方便开发、测试），verbose（许多有用的信息，但是没有debug级别信息多），notice（适当的日志级别，适合生产环境），warn（只有非常重要的信息）
loglevel notice
#指定了记录日志的文件。空字符串的话，日志会打印到标准输出设备。后台运行的redis标准输出是/dev/null。
logfile /var/log/redis/redis-server.log
#数据库的数量，默认使用的数据库是DB 0。可以通过”SELECT “命令选择一个db
databases 16
#设置tcp的backlog，backlog其实是一个连接队列，backlog队列总和=未完成三次握手队列 + 已经完成三次握手队列。在高并发环境下你需要一个高backlog值来避免客户端连接问题。注意Linux内核会将这个值减小到/proc/sys/net/core/somaxconn的值，所以需要确认增大somaxconn和tcp_max_syn_backlog两个值来达到想要的效果
tcp-backlog 511
#单位为秒，如果设置为0，则不会进行Keepalive检测，建议设置成60 
tcp-keepalive 60
#是否把日志输出到syslog中
syslog-enabled
#指定syslog里的日志标志
syslog-ident
#指定syslog设备，值可以是USER或LOCAL0-LOCAL7
syslog-facility
#设置密码 ./redis-cli -a 123456登录
#requirepass 123456

###################################SNAPSHOTTING快照###################################
# RDB是整个内存的压缩过的Snapshot
# 快照配置
# 注释掉“save”这一行配置项就可以让保存数据库功能失效
# 设置sedis进行数据库镜像的频率。
# 900秒（15分钟）内至少1个key值改变（则进行数据库保存--持久化） 
# 300秒（5分钟）内至少10个key值改变（则进行数据库保存--持久化） 
# 60秒（1分钟）内至少10000个key值改变（则进行数据库保存--持久化）
# 如果想禁用RDB持久化的策略，只要不设置任何save指令，或者给save传入一个空字符串参数也可以
save 900 1
save 300 10
save 60 10000

#当RDB持久化出现错误后，是否依然进行继续进行工作，yes：不能进行工作，no：可以继续进行工作，可以通过info中的rdb_last_bgsave_status了解RDB持久化是否有错误
stop-writes-on-bgsave-error yes
# 对于存储到磁盘中的快照，可以设置是否进行压缩存储。如果是的话，redis会采用LZF算法进行压缩。如果你不想消耗CPU来进行压缩的话，可以设置为关闭此功能
rdbcompression yes
#是否校验rdb文件。从rdb格式的第五个版本开始，在rdb文件的末尾会带上CRC64的校验和。这跟有利于文件的容错性，但是在保存rdb文件的时候，会有大概10%的性能损耗，所以如果你追求高性能，可以关闭该配置。
rdbchecksum yes
#rdb文件的名称
dbfilename dump.rdb
#数据目录，数据库的写入会在这个目录。rdb、aof文件也会写在这个目录
dir /var/lib/redis

################################### LIMITS ####################################

# 设置能连上redis的最大客户端连接数量。默认是10000个客户端连接。由于redis不区分连接是客户端连接还是内部打开文件或者和slave连接等，所以maxclients最小建议设置到32。如果超过了maxclients，redis会给新的连接发送’max number of clients reached’，并关闭连接。
# maxclients 10000
#redis配置的最大内存容量。当内存满了，需要配合maxmemory-policy策略进行处理。注意slave的输出缓冲区是不计算在maxmemory内的。所以为了防止主机内存使用完，建议设置的maxmemory需要更小一些。
# maxmemory <bytes>

#内存容量超过maxmemory后的处理策略。
#volatile-lru：利用LRU算法移除设置过过期时间的key。
#volatile-random：随机移除设置过过期时间的key。
#volatile-ttl：移除即将过期的key，根据最近过期时间来删除（辅以TTL）
#allkeys-lru：利用LRU算法移除任何key。
#allkeys-random：随机移除任何key。
#noeviction：不移除任何key，只是返回一个写错误。
#上面的这些驱逐策略，如果redis没有合适的key驱逐，对于写命令，还是会返回错误。redis将不再接收写请求，只接收get请求。写命令包括：set setnx setex append incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby getset mset msetnx exec sort。
# maxmemory-policy noeviction

#lru检测的样本数。使用lru或者ttl淘汰算法，从需要淘汰的列表中随机选择sample个key，选出闲置时间最长的key移除。
# maxmemory-samples 5

############################## APPEND ONLY MODE ###############################
#默认redis使用的是rdb方式持久化，这种方式在许多应用中已经足够用了。但是redis如果中途宕机，会导致可能有几分钟的数据丢失，根据save来策略进行持久化，Append Only File是另一种持久化方式，可以提供更好的持久化特性。Redis会把每次写入的数据在接收后都写入 appendonly.aof 文件，每次启动时Redis都会先把这个文件的数据读入内存里，先忽略RDB文件。
appendonly yes
#aof文件名
appendfilename "appendonly.aof"

#aof持久化策略的配置
#no表示不执行fsync，由操作系统保证数据同步到磁盘，速度最快。
#always表示每次写入都执行fsync，以保证数据同步到磁盘。
#everysec表示每秒执行一次fsync，可能会导致丢失这1s数据。
appendfsync everysec
# 重写时是否可以运用Appendfsync，用默认no即可，保证数据安全性。
no-appendfsync-on-rewrite no

#设置允许重写的最小aof文件大小，避免了达到约定百分比但尺寸仍然很小的情况还要重写
auto-aof-rewrite-min-size 64mb
#aof自动重写配置。当目前aof文件大小超过上一次重写的aof文件大小的百分之多少进行重写，即当aof文件增长到一定大小的时候Redis能够调用bgrewriteaof对日志文件进行重写。当前AOF文件大小是上次日志重写得到AOF文件大小的二倍（设置为100）时，自动启动新的日志重写过程。
auto-aof-rewrite-percentage 100
```



# 六、redis持久化

## RDB

**原理**

在指定的时间间隔内将内存中的数据集快照写入磁盘，也就是行话讲的Snapshot快照，它恢复时是将快照文件直接读到内存里。

Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方式要比AOF方式更加的高效。RDB的缺点是最后一次持久化后的数据可能丢失。

fork的作用是复制一个与当前进程一样的进程。新进程的所有数据（变量、环境变量、程序计数器等）数值都和原进程一致，但是是一个全新的进程，并作为原进程的子进程。

**保存方案**

rdb 保存的是dump.rdb文件。

save命令只管保存，其它不管，全部阻塞。

Redis会在后台异步进行快照操作，快照同时还可以响应客户端请求。可以通过lastsave命令获取最后一次成功执行快照的时间。

**RDB持久化恢复方案**

将备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务即可，CONFIG GET dir获取目录，指定配置文件的dir 路径，dir默认当前目录。

**优势与劣势**

优势：

（1）适合大规模的数据恢复

（2）对数据完整性和一致性要求不高

劣势：

（1）在一定间隔时间做一次备份，所以如果redis意外down掉的话，就会丢失最后一次快照后的所有修改。

（2）fork的时候，内存中的数据被克隆了一份，大致2倍的膨胀性需要考虑

动态停止保存方法：

动态所有停止RDB保存规则的方法：redis-cli config set save ""

## AOF（Append Only File）

**原理**

以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下来(读操作不记录)，只许追加文件但不可以改写文件，redis启动之初会读取该文件重新构建数据，换言之，redis重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作。

**配置文件**

Aof保存的是appendonly.aof文件,可以在redis.conf修改文件名。

修改默认的appendonly no，改为yes

```properties
appendonly yes
# The name of the append only file (default: "appendonly.aof")
appendfilename "appendonly.aof"
#always表示每次写入都执行fsync，以保证数据同步到磁盘。
# appendfsync always
#everysec表示每秒执行一次fsync，可能会导致丢失这1s数据。
appendfsync everysec
#no表示不执行fsync，由操作系统保证数据同步到磁盘，速度最快。
# appendfsync no
```

**AOF启动/修复/恢复**

将有数据的aof文件复制一份保存到对应目录(config get dir)，然后重启redis进行加载而恢复数据。

如果AOP文件存在数据异常，可以使用redis-check-aof --fix进行修复。

**rewrite**

AOF采用文件追加方式，文件会越来越大为避免出现此种情况，新增了重写机制,当AOF文件的大小超过所设定的阈值时，Redis就会启动AOF文件的内容压缩，只保留可以恢复数据的最小指令集.可以使用命令bgrewriteaof

重写原理：

AOF文件持续增长而过大时，会fork出一条新进程来将文件重写(也是先写临时文件最后再rename)，遍历新进程的内存中数据，每条记录有一条的Set语句。重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件，这点和快照有点类似。

触发机制：

Redis会记录上次重写时的AOF大小，默认配置是当AOF文件大小是上次rewrite后大小的一倍且文件大于64M时触发。

**AOF优势与劣势**

优势：

（1）每修改同步：appendfsync always   同步持久化 每次发生数据变更会被立即记录到磁盘  性能较差但数据完整性比较好.
（2）每秒同步：appendfsync everysec    异步操作，每秒记录   如果一秒内宕机，有数据丢失,但数据丢失小。
（3）不同步：appendfsync no   从不同步，性能好。

劣势：

（1）相同数据集的数据而言aof文件要远大于rdb文件，恢复速度慢于rdb
（2）aof运行效率要慢于rdb,每秒同步策略效率较好，不同步效率和rdb相同

## 持久化方案总结

如果只希望数据在服务器运行的时候存在,可以不使用任何持久化方式.

方案对比：

RDB持久化方式能够在指定的时间间隔能对你的数据进行快照存储。
AOF持久化方式记录每次对服务器写的操作,当服务器重启的时候会重新执行这些命令来恢复原始的数据,AOF命令以redis协议追加保存每次写的操作到文件末尾.Redis还能对AOF文件进行后台重写,使得AOF文件的体积不至于过大

同时开启两种持久化方式：

在这种情况下,当redis重启的时候会优先载入AOF文件来恢复原始的数据,因为在通常情况下AOF文件保存的数据集要比RDB文件保存的数据集要完整.
RDB的数据不实时，同时使用两者时服务器重启也只会找AOF文件。那要不要只使用AOF呢？
作者建议不要，因为RDB更适合用于备份数据库(AOF在不断变化不好备份)，快速重启，而且不会有AOF可能潜在的bug，留着作为一个万一的手段。

# 七、Redis事务

可以一次执行多个命令，本质是一组命令的集合。一个事务中的所有命令都会序列化，按顺序地串行化执行而不会被其它命令插入，不许加塞。一个队列中，一次性、顺序性、排他性的执行一系列命令。

**操作流程**

（1）开启：以MULTI开始一个事务
（2）入队：将多个命令入队到事务中，接到这些命令并不会立即执行，而是放到等待执行的事务队列里面
（3）执行：由EXEC命令触发事务

## 事务

**基本命令**

http://redisdoc.com/transaction/index.html

```
DISCARD：取消事务，放弃执行事务块内的所有命令。
EXEC：执行所有事务块内的命令 
MULTI ：标记一个事务块的开始。 
WATCH ：监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。 
UNWATCH：取消 WATCH 命令对所有 key 的监视。 
```

### 执行情况

#### 基本操作

（1）全部正常执行，即事务中的命令全部执行成功

（2）放弃事务，即放弃在事务中的所有操作

（3）全部执行失败（事务中队列中的执行操作，有语法错误）。

（4）部分执行（若队列中命令无语法错误，则全部执行，执行过程报错的不影响其它命令，例如：一个字符串k1="vvsdf"，执行操作 incr  k1不会报语法错误，但会报执行错误）

#### watch监控

 悲观锁(Pessimistic Lock), 顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。

 乐观锁(Optimistic Lock), 顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量，

乐观锁策略:提交版本必须大于记录当前版本才能执行更新。

Watch原理：

Watch指令，类似乐观锁，事务提交时，如果Key的值已被别的客户端改变，比如某个list已被别的客户端push/pop过了，整个事务队列都不会被执行。
通过WATCH命令在事务执行之前监控了多个Keys，倘若在WATCH之后有任何Key的值发生了变化，EXEC命令执行的事务都将被放弃，同时返回Nullmulti-bulk应答以通知调用者事务执行失败。

Watch执行流程：

（1）WATCH  监视一个(或多个) key

（2）MULTI标记一个事务块的开始

（3）执行操作

（4）情况一：监控了key，如果key被修改了，后面一个事务的执行失效， WATCH 取消对所有 key 的监视

情况二：执行exec 把MULTI队列中的命令全部执行完成，并且WATCH监控锁也会被取消掉。

## 分布式锁

[Redission](https://www.jianshu.com/p/67f700fad8b3)

## 特性

单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。

没有隔离级别的概念：队列中的命令没有提交之前都不会实际的被执行，因为事务提交前任何指令都不会被实际执行，也就不存在”事务内的查询要看到事务里的更新，在事务外查询不能看到”这个让人万分头痛的问题。

不保证原子性：redis同一个事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚。

# 八、Redis的发布订阅

## 简介

进程间的一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。

Redis 客户端可以订阅任意数量的频道。 

## 订阅发布消息图

例：频道 channel1 ， 以及订阅这个频道的三个客户端 —— client2 、 client5 和 client1 之间的关系图

![pubsub1](http://www.runoob.com/wp-content/uploads/2014/11/pubsub1.png) 

当有新消息通过 PUBLISH 命令发送给频道 channel1 时， 这个消息就会被发送给订阅它的三个客户端： 

![pubsub2](http://www.runoob.com/wp-content/uploads/2014/11/pubsub2.png) 

## 操作命令

http://redisdoc.com/pubsub/index.html

PSUBSCRIBE pattern [pattern ...] 订阅一个或多个符合给定模式的频道。
PUBSUB subcommand [argument [argument ...]] 查看订阅与发布系统状态。
PUBLISH channel message 将信息发送到指定的频道。
PUNSUBSCRIBE [pattern [pattern ...]] 退订所有给定模式的频道。
SUBSCRIBE channel [channel ...] 订阅给定的一个或多个频道的信息。
UNSUBSCRIBE [channel [channel ...]] 指退订给定的频道。



操作实例：

在redis客服端创建订阅频道channel1：

```
127.0.0.1:6379[1]> SUBSCRIBE channel1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "channel1"
3) (integer) 1
1) "message"
2) "channel1"
3) "hello channel1" //收到的第一条消息
1) "message"
2) "channel1"
3) "come here" //收到的第二条消息
```

重新开启一个redis客服端，然后在同一个频道 redisChat 发布消息，订阅者就能接收到消息。 

```
127.0.0.1:6379[1]> PUBLISH channel1 "hello channel1"
(integer) 1
127.0.0.1:6379[1]> PUBLISH channel1 "come here"
(integer) 1
```

# 九、Redis的复制(Master/Slave)

## 简介

Redis支持简单的主从（master-slave）复制功能，当主Redis服务器更新数据时能将数据自动同步到从Redis服务器 ,Master以写为主，Slave以读为主。

## 主要用途

读写分离、容灾恢复

## 配置搭建

### 原则

配从(库)不配主(库)，从库是不能进行修改操作的。

### 模式

http://redisdoc.com/replication/slaveof.html

每次与master断开之后，都需要重新连接，除非你配置进redis.conf文件

例如有3台redis服务器，分别是A、B、C。

#### 一主二从

以A为主(master)库，B、C为从库。

在B、C上分别执行slaveof 127.0.0.1 6379，然后A库上所有的更新操作都能在B、C中存在。

A:

```
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=70,lag=0
slave1:ip=127.0.0.1,port=6381,state=online,offset=70,lag=1
.....
127.0.0.1:6379> set k1 v1
OK
```

B:

```
127.0.0.1:6380> slaveof 127.0.0.1 6379
OK
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
...
127.0.0.1:6380> get k1
"v1"
```

C:

```
127.0.0.1:6380> slaveof 127.0.0.1 6379
OK
127.0.0.1:6381> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
...
127.0.0.1:6381> get k1
"v1"
```

#### 薪火相传

上一个Slave可以是下一个slave的Master，Slave同样可以接收其他slaves的连接和同步请求，那么该slave作为了链条中下一个的master,可以有效减轻master的写压力。
中途变更转向:会清除之前的数据，重新建立拷贝最新的。
A（master）为主库
B作为A库的slave，作为C库的master
C为B库的slave

A：

```
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=3290,lag=0
127.0.0.1:6379> set k2 v2
OK
```

B:

```
127.0.0.1:6380> slaveof 127.0.0.1 6379
OK
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
...
slave0:ip=127.0.0.1,port=6381,state=online,offset=3262,lag=1
127.0.0.1:6380> get k2
"v2
```

C:

```
127.0.0.1:6381> slaveof 127.0.0.1 6380
OK
127.0.0.1:6381> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6380
...
127.0.0.1:6381> get k2
"v2"
```

#### 反客为主

使当前数据库停止与其他数据库的同步，转成主数据库。

SLAVEOF no one

#### 哨兵模式(sentinel)

##### 介绍

反客为主的自动版，能够后台监控主机是否故障，如果故障了根据投票数自动将从库转换为主库。
例如：当原有的master挂了，剩下的slave会投票选举出新的master，如果之前的master库重新启动了，则会成为同步新选举出来的master的slave从库。
一组sentinel能同时监控多个Master。

##### 步骤

（1）在redis.conf对应目录下新建sentinel.conf文件。并且配置哨兵： 
sentinel monitor 自定义被监控redis库 127.0.0.1 6379 1
上面最后一个数字1，表示主机挂掉后salve投票看让谁接替成为主机，得票数多少后成为主机。
例如：监控A库
sentinel.conf:

```
sentinel monitor host6379 127.0.0.1 6379 1
```

（2）启动哨兵

```
redis-sentinel sentinel.conf 
```

## 复制原理

slave启动成功连接到master后会发送一个sync命令
Master接到命令启动后台的存盘进程，同时收集所有接收到的用于修改数据集命令，在后台进程执行完毕之后，master将传送整个数据文件到slave,以完成一次完全同步
全量复制：而slave服务在接收到数据库文件数据后，将其存盘并加载到内存中。
增量复制：Master继续将新的所有收集到的修改命令依次传给slave,完成同步，但是只要是重新连接master,一次完全同步（全量复制)将被自动执行

例：当slave第一次连上master会全量复制，后面进行的是增量复制

## 复制缺点

复制延时：

由于所有的写操作都是先在Master上操作，然后同步更新到Slave上，所以从Master同步到Slave机器有一定的延迟，当系统很繁忙的时候，延迟问题会更加严重，Slave机器数量的增加也会使这个问题更加严重。

# Jedis

## 基础

Jedis是 Redis 官方首选的 Java 客户端开发包。 

开发需要的jar包：commons-pool-1.6.jar、jedis-2.1.0.jar

maven：

```
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.1.0</version>
</dependency>
<dependency>
    <groupId>commons-pool</groupId>
    <artifactId>commons-pool</artifactId>
    <version>1.6</version>
</dependency>
```

## 常用操作

### 连接测试

```java
Jedis jedis = new Jedis("192.168.10.206",6379);
System.out.println(jedis.ping());
```

### 基本类型操作

```java
Jedis jedis = new Jedis("192.168.10.206",6379);
//获取 keys * 结果
Set<String> keys = jedis.keys("*");
for (Iterator iterator = keys.iterator(); iterator.hasNext();) {
    String key = (String) iterator.next();
    System.out.println(key);
}
//查看k2是否存在
System.out.println("jedis.exists====>"+jedis.exists("k2"));
//查看k1还有多少秒过期
System.out.println(jedis.ttl("k1"));
//查看k1值
System.out.println(jedis.get("k1"));
//设置k4=k4_redis
jedis.set("k4","k4_redis");
//多个字符串同时设置值
jedis.mset("str1","v1","str2","v2","str3","v3");
//获取多个字符串值
System.out.println(jedis.mget("str1","str2","str3"));

//list设置值 及获取
jedis.lpush("mylist","v1","v2","v3","v4","v5");
List<String> list = jedis.lrange("mylist",0,-1);
for (String element : list) {
    System.out.println(element);
}

//set设置值 及获取
jedis.sadd("orders","jd001");
jedis.sadd("orders","jd002");
jedis.sadd("orders","jd003");
Set<String> set1 = jedis.smembers("orders");
for (Iterator iterator = set1.iterator(); iterator.hasNext();) {
    String string = (String) iterator.next();
    System.out.println(string);
}
//set移除值
jedis.srem("orders","jd002");
//hash设值及获取
jedis.hset("hash1","userName","lisi");
System.out.println(jedis.hget("hash1","userName"));

//zset添加值
jedis.zadd("zset01",90d,"v4");
//zset获取全部值
Set<String> s1 = jedis.zrange("zset01",0,-1);
for (Iterator iterator = s1.iterator(); iterator.hasNext();) {
    String string = (String) iterator.next();
    System.out.println(string);
}
```

### 事务控制

```java
Jedis jedis = new Jedis("127.0.0.1", 6379);
int balance;// 可用余额
int amtToSubtract = 10;// 实刷额度
//watch监控
jedis.watch("balance");
balance = Integer.parseInt(jedis.get("balance"));
if (balance < amtToSubtract) {
    //取消watch监控
    jedis.unwatch();
    System.out.println("modify");
} else {
    //开启事务
    Transaction transaction = jedis.multi();
    transaction.decrBy("balance", amtToSubtract);
    transaction.incrBy("debt", amtToSubtract);
    //执行并提交事务 watch监控也会取消
    transaction.exec();
    System.out.println("success");
}
```

### 主从复制

```java
Jedis jedis_M = new Jedis("127.0.0.1",6379);
Jedis jedis_S = new Jedis("127.0.0.1",6380);
jedis_S.slaveof("127.0.0.1",6379);
```

### Jedis池

获取Jedis实例需要从JedisPool中获取,用完Jedis实例需要返还给JedisPool,如果Jedis在使用过程中出错，则也需要还给JedisPool

```java
public class JedisPoolUtil {
    private static volatile JedisPool jedisPool = null;

    private JedisPoolUtil(){}

    public static JedisPool getJedisPoolInstance(){
        if(null == jedisPool){
            synchronized (JedisPoolUtil.class){
                if(null == jedisPool){
                    JedisPoolConfig poolConfig = new JedisPoolConfig();
                    poolConfig.setMaxActive(1000);
                    poolConfig.setMaxIdle(32);
                    poolConfig.setMaxWait(100*1000);
                    poolConfig.setTestOnBorrow(true);

                    jedisPool = new JedisPool(poolConfig,"127.0.0.1",6379);
                }
            }
        }
        return jedisPool;
    }

    public static void release(JedisPool jedisPool,Jedis jedis){
        if(null != jedis){
            jedisPool.returnResourceObject(jedis);
        }
    }
```

## JedisPoolConfig配置

```properties
JedisPool的配置参数大部分是由JedisPoolConfig的对应项来赋值的。
maxActive：控制一个pool可分配多少个jedis实例，通过pool.getResource()来获取；如果赋值为-1，则表示不限制；如果pool已经分配了maxActive个jedis实例，则此时pool的状态为exhausted。
maxIdle：控制一个pool最多有多少个状态为idle(空闲)的jedis实例；
whenExhaustedAction：表示当pool中的jedis实例都被allocated完时，pool要采取的操作；默认有三种。
 WHEN_EXHAUSTED_FAIL --> 表示无jedis实例时，直接抛出NoSuchElementException；
 WHEN_EXHAUSTED_BLOCK --> 则表示阻塞住，或者达到maxWait时抛出JedisConnectionException；
 WHEN_EXHAUSTED_GROW --> 则表示新建一个jedis实例，也就说设置的maxActive无用；
maxWait：表示当borrow一个jedis实例时，最大的等待时间，如果超过等待时间，则直接抛JedisConnectionException；
testOnBorrow：获得一个jedis实例的时候是否检查连接可用性（ping()）；如果为true，则得到的jedis实例均是可用的；


testOnReturn：return 一个jedis实例给pool时，是否检查连接可用性（ping()）；


testWhileIdle：如果为true，表示有一个idle object evitor线程对idle object进行扫描，如果validate失败，此object会被从pool中drop掉；这一项只有在timeBetweenEvictionRunsMillis大于0时才有意义；


timeBetweenEvictionRunsMillis：表示idle object evitor两次扫描之间要sleep的毫秒数；


numTestsPerEvictionRun：表示idle object evitor每次扫描的最多的对象数；


minEvictableIdleTimeMillis：表示一个对象至少停留在idle状态的最短时间，然后才能被idle object evitor扫描并驱逐；这一项只有在timeBetweenEvictionRunsMillis大于0时才有意义；


softMinEvictableIdleTimeMillis：在minEvictableIdleTimeMillis基础上，加入了至少minIdle个对象已经在pool里面了。如果为-1，evicted不会根据idle time驱逐任何对象。如果minEvictableIdleTimeMillis>0，则此项设置无意义，且只有在timeBetweenEvictionRunsMillis大于0时才有意义；


lifo：borrowObject返回对象时，是采用DEFAULT_LIFO（last in first out，即类似cache的最频繁使用队列），如果为False，则表示FIFO队列；


其中JedisPoolConfig对一些参数的默认设置如下：
testWhileIdle=true
minEvictableIdleTimeMills=60000
timeBetweenEvictionRunsMillis=30000
numTestsPerEvictionRun=-1
```
