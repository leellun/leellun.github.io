---
title: elasticsearch配置文件
date: 2019-03-24 18:19:02
categories:
  - 服务器
  - elasticsearch
tags:
  - elasticsearch
---

# 一、配置文件

```
# ---------------------------------- Cluster -----------------------------------
# 集群名称
#cluster.name: my-application
# ------------------------------------ Node ------------------------------------
# 节点名称
#node.name: node-1
#
# 向节点添加自定义属性
#node.attr.rack: r1
# ----------------------------------- Paths ------------------------------------
# 存储数据的目录路径(多个位置用逗号分隔):
#path.data: /path/to/data
# 日志文件
#path.logs: /path/to/logs
#
# ----------------------------------- Memory -----------------------------------
#启动时锁定内存
#bootstrap.memory_lock: true
#
# 在启动时锁定内存确保将堆大小设置为系统可用内存的一半左右，并且允许进程的所有者使用这个限制。
#
# 当系统交换内存时，Elasticsearch的性能很差
#
# ---------------------------------- Network -----------------------------------
# 默认情况下，Elasticsearch只能在本地主机上访问。设置一个不同的地址在网络上暴露这个节点:
#network.host: 192.168.0.1
#
# 默认情况下，Elasticsearch在它的第一个空闲端口上侦听HTTP通信，从9200开始查找。在这里设置一个特定的HTTP端口:
#
#http.port: 9200
#
# --------------------------------- Discovery 发现 ----------------------------------
#
# 当该节点启动时，通过一个初始主机列表执行发现:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#discovery.seed_hosts: ["host1", "host2"]
#
# 使用一组符合主条件的初始节点引导集群:
#cluster.initial_master_nodes: ["node-1", "node-2"]
#
# --------------------------------- Readiness ----------------------------------
#
# 在本地主机上启用未验证的TCP准备端点
#
#readiness.port: 9399
#
# ---------------------------------- Various -----------------------------------
#
# 允许通配符删除索引:
#
#action.destructive_requires_name: false

#----------------------- BEGIN SECURITY AUTO CONFIGURATION -----------------------
#
# 为配置Elasticsearch安全特性，已在26-08-2022 09:42:48自动生成如下配置、TLS证书和密钥
#
# --------------------------------------------------------------------------------

# 启用安全特性
xpack.security.enabled: true

xpack.security.enrollment.enabled: true

# 对HTTP API客户端连接(如Kibana、Logstash、Agents)启用加密
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

# 启用集群节点之间的加密和相互认证功能
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
# 启用集群节点之间的加密和相互认证仅使用当前节点创建新集群后续仍可加入集群
cluster.initial_master_nodes: ["k8s-master01"]

# 允许来自任何地方的HTTP API连接 连接是加密的，需要用户身份验证
http.host: 0.0.0.0

# 允许其他节点从加密和相互验证连接的任何地方加入集群
#transport.host: 0.0.0.0
# 启动时会去更新地图的一些数据库
ingest.geoip.downloader.enabled: false
#----------------------- END SECURITY AUTO CONFIGURATION -------------------------

```

