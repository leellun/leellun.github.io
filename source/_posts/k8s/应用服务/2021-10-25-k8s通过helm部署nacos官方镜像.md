---
title: k8s通过helm部署nacos官方镜像
categories:
  - 服务器
  - 后端
tags:
  - nacos
  - k8s
date: 2021-10-25 20:14:02
---

# 一、官方部署介绍

github仓库地址

```
https://github.com/nacos-group/nacos-k8s.git
```

官方部署方案：

```
cd nacos-k8s
chmod +x quick-startup.sh
./quick-startup.sh
```

说明：

quick-startup.sh 脚本部署了mysql和nacos

```
#!/usr/bin/env bash

echo "mysql mysql startup"
kubectl create -f ./deploy/mysql/mysql-local.yaml


echo "nacos quick startup"
kubectl create -f ./deploy/nacos/nacos-quick-start.yaml
```

# 二、通过helm部署nacos

nacos-k8s提供了helm部署方案

## 2.1 配置nacos

value.yaml文件配置

```
global:
# 配置分布式模式
  mode: cluster

############################nacos###########################
# namespace配置
namespace: newland
nacos:
# 配置镜像，采用官方推荐的2.x版本
  image:
    repository: nacos/nacos-server
    tag: v2.1.1
    pullPolicy: IfNotPresent
  plugin:
    enable: true
    #nacos-peer-finder-plugin镜像initContainer容器作用，主要方便扩容
    image:
      repository: nacos/nacos-peer-finder-plugin
      tag: 1.1
  # 配置副本数
  replicaCount: 3
  # 域名名称
  domainName: cluster.local
  preferhostmode: hostname
  serverPort: 8848
  health:
    enabled: false
  storage:
# 这里采用外部数据库 mysql
    type: mysql
    db:
    #k8s中通过域名+dns的组合方式
      host: mysql-6x5v.newland
      name: nacos
      port: 3306
      username: nacos
      password: nacos
      param: characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false
```

## 2.2 部署nacos

```
helm install nacos ../helm -n newland 
```

## 2.3 查看部署结果

```
[root@k8s-master01 helm]# kubectl get svc,pod -n newland
NAME                                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                                                         AGE
service/mysql                           NodePort    10.0.0.79    <none>        3306:31023/TCP                                                  33h
service/mysql-6x5v                      ClusterIP   None         <none>        3306/TCP,33060/TCP                                              33h
service/nacos-cs                        NodePort    10.0.0.158   <none>        8848:31384/TCP,9848:32243/TCP,9849:31074/TCP,7848:30000/TCP     64m
service/nacos-hs                        ClusterIP   None         <none>        8848/TCP,9848/TCP,9849/TCP,7848/TCP                             64m

NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-0                  1/1     Running   0          33h
pod/nacos-0                  1/1     Running   0          42m
pod/nacos-1                  1/1     Running   0          38m
pod/nacos-2                  1/1     Running   0          35m
```

通过查看可以看出nacos部署3个副本已经部署成功，并且nacos提供了内部访问none和nodeport的外部访问的两个服务。