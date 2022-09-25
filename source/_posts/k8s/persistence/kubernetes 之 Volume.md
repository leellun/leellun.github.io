---
title: kubernetes 之 Volume
date: 2021-08-26 21:14:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - k8s持久卷
---

# 1 Volume 介绍

目的：容器磁盘上的文件的生命周期是短暂的,这就使得在容器中运行重要应用时会出现一些问题。首先,当容器崩溃时, kubelet会重启它,但是容器中的文件将丢失--容器以干净的状态(镜像最初的状态)重新启动。其次,在Pod中同时运行多个容器时,这些容器之间通常需要共享文件。Kubernetes中的Volume抽象就很好的解决了
这些问题。

Volume 是Pod 中能够被多个容器访问的共享目录。Kubernetes 的Volume 定义在Pod 上，它被一个Pod 中的多个容器挂载到具体的文件目录下。Volume 与Pod 的生命周期相同，但与容器的生命周期不相关，当容器终止或重启时，Volume 中的数据也不会丢失。要使用volume，pod 需要指定volume 的类型和内容（ 字段），和映射到容器的位置（ 字段）。

Kubernetes 中的卷有明确的寿命--与封装它的Pod 相同。所f以,卷的生命比Pod中的所有容器都长,当这个容器重启时数据仍然得以保存。当然,当Pod不再存在时,卷也将不复存在。也许更重要的是, Kubernetes支持多种类型的卷, Pod可以同时使用任意数量的卷

Kubernetes 支持多种类型的Volume,包括：
emptyDir、hostPath、gcePersistentDisk、awsElasticBlockStore、nfs、iscsi、flocker、glusterfs、rbd、cephfs、gitRepo、secret、persistentVolumeClaim、downwardAPI、azureFileVolume、azureDisk、vsphereVolume、Quobyte、PortworxVolume、ScaleIO。

# 2 Volume挂载

## emptyDir

emptyDirEmptyDir 类型的volume创建于pod 被调度到某个宿主机上的时候，而同一个pod 内的容器都能读写EmptyDir 中的同一个文件。一旦这个pod 离开了这个宿主机，EmptyDir 中的数据就会被永久删除。所以目前EmptyDir 类型的volume 主要用作临时空间，比如Web 服务器写日志或者tmp 文件需要的临时目录。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: harborcloud.com/library/nginx:1.9.1
    name: nginx-web
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  - image: harborcloud.com/library/busybox:v1.35
    name: busybox-1
    command: ["/bin/sh","-c","sleep 6000s"]
    volumeMounts:
    - mountPath: /cache2
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

## hostPath

HostPath 属性的volume 使得对应的容器能够访问当前宿主机上的指定目录。一旦这个pod 离开了这个宿主机，HostDir 中的数据虽然不会被永久删除，但数据也不会随pod 迁移到其他宿主机上

hostPath卷将主机节点的文件系统中的文件或目录挂载到集群中

hostPath的一些用法有：

- 运行一个需要访问 Docker 内部机制的容器；可使用 `hostPath` 挂载 `/var/lib/docker` 路径。
- 在容器中运行 cAdvisor 时，以 `hostPath` 方式挂载 `/sys`。
- 允许 Pod 指定给定的 `hostPath` 在运行 Pod 之前是否应该存在，是否应该创建以及应该以什么方式存在。

| 取值                | 行为                                                         |
| :------------------ | :----------------------------------------------------------- |
|                     | 空字符串（默认）用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查。 |
| `DirectoryOrCreate` | 如果在给定路径上什么都不存在，那么将根据需要创建空目录，权限设置为 0755，具有与 kubelet 相同的组和属主信息。 |
| `Directory`         | 在给定路径上必须存在的目录。                                 |
| `FileOrCreate`      | 如果在给定路径上什么都不存在，那么将在那里根据需要创建空文件，权限设置为 0644，具有与 kubelet 相同的组和所有权。 |
| `File`              | 在给定路径上必须存在的文件。                                 |
| `Socket`            | 在给定路径上必须存在的 UNIX 套接字。                         |
| `CharDevice`        | 在给定路径上必须存在的字符设备。                             |
| `BlockDevice`       | 在给定路径上必须存在的块设备。                               |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: harborcloud.com/library/nginx:1.9.1
    name: test-container
# 指定在容器中挂接路径
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
# 指定所提供的存储卷
  volumes:
  - name: test-volume # 宿主机上的目录hostPath:
    hostPath:
      # host上目录
      path: /data
      # 类型(可选)
      type: Directory
```

## nfs

允许一块现有的网络硬盘在同一个pod 内的容器间共享

```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: redis
spec:
  selector:
    matchLabels:
      app: redis
  revisionHistoryLimit: 2
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      # 应用的镜像
      - image: redis
        name: redis
        imagePullPolicy: IfNotPresent # 应用的内部端口
        ports:
        - containerPort: 6379
          name: redis6379
        env:
        - name: ALLOW_EMPTY_PASSWORD
          value: "yes"
        - name: REDIS_PASSWORD
          value: "redis"
        # 持久化挂接位置，在docker 中
        volumeMounts:
        - name: redis-persistent-storage
          mountPath: /data
      volumes:
        # 宿主机上的目录
      - name: redis-persistent-storage
        nfs:
          path: /k8s-nfs/redis/data
          server: 192.168.126.112
```

## configMap

ConfigMap 对象中存储的数据可以被 `configMap` 类型的卷引用，然后被 Pod 中运行的容器化应用使用。

引用 configMap 对象时，你可以在 volume 中通过它的名称来引用。 你可以自定义 ConfigMap 中特定条目所要使用的路径。 下面的配置显示了如何将名为 `log-config` 的 ConfigMap 挂载到名为 `configmap-pod` 的 Pod 中：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox:1.28
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
```

`log-config` ConfigMap 以卷的形式挂载，并且存储在 `log_level` 条目中的所有内容都被挂载到 Pod 的 `/etc/config/log_level` 路径下。 请注意，这个路径来源于卷的 `mountPath` 和 `log_level` 键对应的 `path`。

- 容器以 subPath 卷挂载方式使用 ConfigMap 时，将无法接收 ConfigMap 的更新。
- 文本数据挂载成文件时采用 UTF-8 字符编码。如果使用其他字符编码形式，可使用 `binaryData` 字段。

