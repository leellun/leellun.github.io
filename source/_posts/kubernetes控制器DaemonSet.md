---
title: kubernetes控制器DaemonSet
date: 2021-07-13 13:32:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - daemonset
---

# 什么是DaemonSet？

DaemonSet 确保全部(或者一些) Node上运行一个Pod的副本。当有Node加入集群时,也会为他们新增一个Pod.当有Node从集群移除时,这些Pod也会被回收。删除DaemonSet将会删除它创建的所有Pod。

DaemonSet 的一些典型用法：

- 在每个节点上运行集群守护进程，集群存储daemon,例如在每个Node上运行glusterd,ceph。
- 在每个节点上运行日志收集守护进程，例如fluentd,logstash。
- 在每个节点上运行监控守护进程，例如Prometheus Node Exporter, collectd, Datadog 代理、New Relic 代理,或 Ganglia gmond

一种简单的用法是为每种类型的守护进程在所有的节点上都启动一个 DaemonSet。 一个稍微复杂的用法是为同一种守护进程部署多个 DaemonSet；每个具有不同的标志， 并且对不同硬件类型具有不同的内存、CPU 要求。 

# 编写 DaemonSet Spec

简单DaemonSet资源配置

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: deamonset-example
  labels:
    app: daemonset
spec:
  selector:
    matchLabels:
      name: deamonset-example
  template:
    metadata:
      labels:
        name: deamonset-example
    spec:
      containers:
      - name: daemonset-example
        image: harborcloud.com/library/nginx:1.9.1
```

详细 DaemonSet 资源配置

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # 这些容忍度设置是为了让该守护进程集在控制平面节点上运行
      # 如果你不希望自己的控制平面节点运行 Pod，可以删除它们
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

## 仅在某些节点上运行 Pod

如果指定了 .spec.template.spec.nodeSelector，DaemonSet 控制器将在能够与 Node 选择算符 匹配的节点上创建 Pod。 类似这种情况，可以指定 .spec.template.spec.affinity，之后 DaemonSet 控制器 将在能够与节点亲和性 匹配的节点上创建 Pod。 如果根本就没有指定，则 DaemonSet Controller 将在所有节点上创建 Pod

