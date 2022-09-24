---
title: kubernetes基础控制器
date: 2021-06-30 20:24:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - 控制器
---

**ReplicationController**

ReplicationController 用来确保容器应用的副本数始终保持在用户定义的副本数,即如果有容器异常退出，会自动创建新的Pod来替代；而如果异常多出来的容器也会自动回收。
在新版本的Kubernetes 中建议使用ReplicaSet取代ReplicationController。

**ReplicaSet**

ReplicaSet 和ReplicationController 没有本质的不同,只是名字不一样,并且ReplicaSet支持集合式的selector
虽然ReplicaSet 可以独立使用，但一般还是建议使用Deployment来自动管理ReplicaSet ，这样就无需担心跟其他机制的不兼容问题(比如ReplicaSet不支持rolling-update，但Deployment 支持)

**Deployment**

Deployment Pod和ReplicaSet提供了一个声明式定义(declarative)方法,用来替代以前的ReplicationController来方便的管理应用。

Deployment 是Kubenetes v1.2 引入的新概念，引入的目的是为了更好的解决Pod 的编排问题，Deployment 内部使用了Replica Set 来实现。

典型的应用场景包括：

- 定义Deployment 来创建Pod和ReplicaSet
- 滚动升级和回滚应用
- 扩容和缩容
- 暂停和继续Deployment

**Horizontal Pod Autoscaling**

Horizontal Pod Autoscaling 仅适用于Deployment和ReplicaSet ,在V1版本中仅支持根据Pod的CPU利用率扩所容,在vlalpha版本中,支持根据内存和用户自定义的metric扩缩容。

**StatefulSet**

StatefulSet是为了解决有状态服务的问题(对应Deployments和ReplicaSets是为无状态服务而设计) ,其应用场景包括:

- 稳定的持久化存储,即Pod重新调度后还是能访问到相同的持久化数据,基于PVC来实现
- 稳定的网络标志,即Pod重新调度后其PodName和HostNameT变,基于Headless Service(即没有Cluster IP的Service)来实现
- 有序部署,有序扩展,即Pod是有顺序的,在部署或者扩展的时候要依据定义的顺序依次依次进行(即从0到N-1,在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态)基于init containers来实现
- 有序收缩,有序删除(即从N-1到0)

**DaemonSet**

DaemonSet确保全部(或者一些)Node上运行一个Pod的副本。当有Node加入集群时,也会为他们新增一个Pod。当有Node从集群移除时,这些Pod也会被回收。删除DaemonSet将会删除它创建的所有Pod
使用DaemonSet的一些典型用法:

- 运行集群存储daemon,例如在每个Node上运行 glusterd,ceph.
- 在每个Node上运行日志收集daemon,例如fluentd、 logstash
- 在 Node上运行监控 daemon,例如Prometheus Node Exporter

**Job**

Job负责批处理任务,即仅执行一次的任务,它保证批处理任务的一个或多个Pod成功结束

**Cron Job **

管理基于时间的Job,即:

- 在给定时间点只运行一次
- 周期性地在给定时间点运行