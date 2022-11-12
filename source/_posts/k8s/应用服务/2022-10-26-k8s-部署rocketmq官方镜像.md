---
title: k8s 部署rocketmq官方镜像
categories:
  - 服务器
  - 后端
tags:
  - rocketmq
  - k8s
date: 2022-10-26 20:14:02
---

## 1、安装CRDS

### 1.1  下载安装crds

```
git clone https://github.com/apache/rocketmq-operator
cd rocketmq-operator && make deploy
```

### 1.2 检查是否安装成功

```
[root@k8s-master01 rocketmq-operator]# kubectl get crd | grep rocketmq.apache.org
brokers.rocketmq.apache.org                                  2022-11-06T14:13:48Z
consoles.rocketmq.apache.org                                 2022-11-06T14:13:48Z
controllers.rocketmq.apache.org                              2022-11-06T14:13:47Z
nameservices.rocketmq.apache.org                             2022-11-06T14:13:48Z
topictransfers.rocketmq.apache.org                           2022-11-06T14:13:48Z
[root@k8s-master01 rocketmq-operator]# kubectl get pods | grep rocketmq-operator
rocketmq-operator-5b59b6f6f8-wlblz   1/1     Running   0          2m34s
```

## 2 创建实例

```
[root@k8s-master01 rocketmq-operator]# cd example && kubectl create -f rocketmq_v1alpha1_rocketmq_cluster.yaml
configmap/broker-config created
broker.rocketmq.apache.org/broker created
nameservice.rocketmq.apache.org/name-service created
console.rocketmq.apache.org/console created
[root@k8s-master01 example]# kubectl get sts -n newland
NAME                 READY   AGE
broker-0-master      1/1     97s
broker-0-replica-1   1/1     97s
broker-1-master      1/1     97s
broker-1-replica-1   1/1     97s
name-service         1/1     2m2s
```

 