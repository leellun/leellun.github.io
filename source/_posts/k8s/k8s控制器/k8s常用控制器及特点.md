---
title: k8s常用控制器及特点
date: 2021-08-01 20:24:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - 控制器
---

# 什么是控制器

Kubernetes 中内建了很多controller (控制器),这些相当于一个状态机,用来控制Pod的具体状态和行为

# 控制器类型

- ReplicationController 和 ReplicaSet
- Deployment
- DaemonSet
- StateFulSet
- Job/CronJob
- Horizontal Pod Autoscaling

## ReplicationController 和 ReplicaSet

ReplicationController (RC)用来确保容器应用的副本数始终保持在用户定义的副本数,即如果有容器异常退出,会自动创建新的Pod来替代;而如果异常多出来的容器也会自动回收;

在新版本的Kubernetes 中建议使用ReplicaSet 来取代ReplicationController. ReplicaSet 跟ReplicationController 没有本质的不同,只是名字不一样,并且ReplicaSet支持集合式的selector;

虽然ReplicaSet 可以独立使用，但一般还是建议使用Deployment来自动管理ReplicaSet ，这样就无需担心跟其他机制的不兼容问题(比如ReplicaSet不支持rolling-update，但Deployment 支持)

## Deployment

Deployment为Pod和ReplicaSet提供了一个声明式定义(declarative)方法,用来替代以前的ReplicationController来方便的管理应用。

Deployment 是Kubenetes v1.2 引入的新概念，引入的目的是为了更好的解决Pod 的编排问题，Deployment 内部使用了Replica Set 来实现。

典型的应用场景包括;

- 定义Deployment 来创建Pod和ReplicaSet
- 滚动升级和回滚应用
- 扩容和缩容
- 暂停和继续Deployment

## DaemonSet

DaemonSet确保全部(或者一些) Node上运行一个Pod的副本。当有Node加入集群时,也会为他们新增一个Pod.当有Node从集群移除时,这些Pod也会被回收。删除DaemonSet将会删除它创建的所有Pod

使用DaemonSet的一些典型用法:

- 运行集群存储daemon,例如在每个Node上运行 glusterd,ceph
- 在每个Node上运行日志收集daemon,例如fluentd, logstash
- 在每个Node上运行监控daemon,例如Prometheus Node Exporter, collectd, Datadog 代理、New Relic 代理,或 Ganglia gmond

## Job

Job负责批处理任务,即仅执行一次的任务,它保证批处理任务的一个或多个Pod成功结束

## CronJob

CronJob管理基于时间的Job,即:

- 在给定时间点只运行一次
- 周期性地在给定时间点运行

使用前提条件:林当前使用的Kubernetes集群,版本>=1.8 (对Cronjob).对于先前版本的集群,版本<1.8,启动API Server时,通过传递选项-runtime-config=batch/v2alphal=true可以开启batch/v2alpha API**

典型的用法如下所示:

- 在给定的时间点调度Job运行
- 创建周期性运行的Job,例如:数据库备份、发送邮件

## StatefulSet

StatefulSet 作为Controller 为Pod提供唯一的标识。它可以保证部署和scale的顺序。

StatefulSet是为了解决有状态服务的问题(对应Deployments和ReplicaSets是为无状态服务而设计),其应用场景包括:

- 稳定的持久化存储,即Pod重新调度后还是能访问到相同的持久化数据,基于PVC来实现
- 稳定的网络标志,即Pod重新调度后其PodName和HostName不变,基于Headless Service (即没有Cluster IP的Service)来实现
- 有序部署,有序扩展,即Pod是有顺序的,在部署或者扩展的时候要依据定义的顺序依次依次进行(即从0到N-1, 在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态),基于init containers来实现
- 有序收缩,有序删除(即从N-1到0)

## Horizontal Pod Autoscaling

应用的资源使用率通常都有高峰和低谷的时候,如何削峰填谷,提高集群的整体资源利用率,让service中的Pod个数自动调整呢?这就有赖于Horizontal Pod Autoscaling了,顾名思义,使Pod水平自动缩放.

Horizontal Pod Autoscaling 仅适用于Deployment和ReplicaSet ,在V1版本中仅支持根据Pod的CPU利用率扩所容,在vlalpha版本中,支持根据内存和用户自定义的metric扩缩容。