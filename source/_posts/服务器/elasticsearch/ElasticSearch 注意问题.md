---
title: ElasticSearch 安装注意问题
date: 2019-03-22 18:14:02
categories:
  - 服务器
  - elasticsearch
tags:
  - elasticsearch
---

1 如果系统配置 JAVA_HOME ，那么使用系统默认的 JDK ，如果没有配置使用自带的 JDK。

2 若启动失败，排查是否内存问题

config/jvm.options:

```
//JVM初始内存
Xms1g
//JVM最大内存
Xmx1g
```

3 出现max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]

```
vi /etc/sysctl.conf
# 添加下面配置：
vm.max_map_count=262144
# 并执行命令：
sysctl -p
# 查看
sysctl -a|grep vm.max_map_count
```

4 出现max file descriptors [4096] for elasticsearch process likely too low, increase to at least [65536]

```
修改/etc/security/limits.conf文件，添加或修改如下行：
*        hard    nofile           65536
*        soft    nofile           65536
```

5 出现max number of threads [1024] for user [hadoop] is too low, increase to at least [2048]

```
vi /etc/security/limits.d/90-nproc.conf
找到如下内容：
* soft nproc 1024
#修改为
* soft nproc 2048
```

6 ERROR: [1] bootstrap checks failed
[]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured

```
//在elasticsearch的config目录下，修改elasticsearch.yml配置文件
cluster.initial_master_nodes: ["node-1"] #这里的node-1为node-name配置的值
```



7 补：启动异常：ERROR: bootstrap checks failed
system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk

```
问题原因：因为Centos6不支持SecComp，而ES5.2.1默认bootstrap.system_call_filter为true进行检测，所以导致检测失败，失败后直接导致ES不能启动。详见 ：https://github.com/elastic/elasticsearch/issues/22899
```

