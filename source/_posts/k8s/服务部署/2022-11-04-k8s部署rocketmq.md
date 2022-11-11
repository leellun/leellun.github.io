---
title: k8s部署rocketmq
date: 2022-11-04 20:14:02
categories:
  - 服务器
  - k8s
tags:
  - k8s
  - rocketmq
---

# 一、 下载rocketmq

rocketmq已经采用operator，而operator 是一种 kubernetes 的扩展形式，可以帮助用户以 Kubernetes 的声明式 API 风格自定义来管理应用及服务。更多介绍可以参考：[十分钟弄懂 k8s Operator 应用的制作流程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/246550722)

rocketmq-operator克隆：

```
git clone https://github.com/apache/rocketmq-operator
```

在make之前需要配置fq的网络，因为下载的依赖需要，比如我搭建了一个xray，本地配置了一个xray服务。

```
vi /etc/profile
http_proxy=http://127.0.0.1:10809/
https_proxy=http://127.0.0.1:10809/
```

开始make

```
cd rocketmq-operator && make deploy
```

查看

```
[root@k8s-master01 rocketmq-operator]# kubectl get crd | grep rocketmq.apache.org
brokers.rocketmq.apache.org                           2022-11-04T03:04:19Z
consoles.rocketmq.apache.org                          2022-11-04T03:04:20Z
controllers.rocketmq.apache.org                       2022-11-04T03:04:19Z
nameservices.rocketmq.apache.org                      2022-11-04T03:04:19Z
topictransfers.rocketmq.apache.org                    2022-11-04T03:04:20Z
[root@k8s-master01 rocketmq-operator]# kubectl get pods | grep rocketmq-operator
rocketmq-operator-5b59b6f6f8-jv4zf   1/1     Running   0          12m
```

# 二、开始创建rocketmq

搭建完rocketmq的operator的支持服务后，就可以开始创建rocketmq了。

```
cd example && kubectl create -f rocketmq_v1alpha1_rocketmq_cluster.yaml
```

清单文件：rocketmq_v1alpha1_rocketmq_cluster.yaml

```
ConfigMap:配置
Broker：broker服务
NameService：nameservice
Console：rocketmq操作界面
```

查看安装情况

```
error: the server doesn't have a resource type "ss"
[root@k8s-master01 example]# kubectl get sts -n newland
NAME                   READY   AGE
broker-0-master        1/1     4m17s
broker-0-replica-1     1/1     4m17s
broker-1-master        1/1     4m17s
broker-1-replica-1     1/1     4m17s
```

# 三、创建服务

```
kubectl create -f rocketmq_v1alpha1_cluster_service.yaml
```

配置：

```
apiVersion: v1
kind: Service
metadata:
  name: console-service
  namespace: newland
  labels:
    app: rocketmq-console
spec:
  type: NodePort
  selector:
    app: rocketmq-console
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      nodePort: 30010
---
apiVersion: v1
kind: Service
metadata:
  name: name-server-service
  namespace: newland
spec:
  type: NodePort
  selector:
    name_service_cr: name-service
  ports:
    - port: 9876
      targetPort: 9876
#       use this port to access the name server cluster
      nodePort: 30012
---
```

查看

```
[root@k8s-master01 example]# kubectl get svc -n newland 
NAME                            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                                                         AGE
console-service                 NodePort    10.0.0.189   <none>        8080:30010/TCP                                                  7m51s
name-server-service             NodePort    10.0.0.68    <none>        9876:30012/TCP                                                  7m51s                                                  37h
```

