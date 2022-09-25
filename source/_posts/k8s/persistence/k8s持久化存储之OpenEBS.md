---
title: k8s持久化存储之OpenEBS
date: 2022-07-23 17:38:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - openebs
---

# 一、介绍

OpenEBS 是 CNCF 项目的一部分，采用 Apache v2 许可证。是 Kubernetes 部署使用最广泛且易用的开源存储解决方案。

**目的：**

让持久化工作负载的存储和存储服务完全集成到环境中，这样每个团队和工作负载都可以从控制的粒度和 Kubernetes 原生行为中获益。

**特点：**

- 微服务架构，使用 Kubernetes 自身的能力来编排管理 OpenEBS 组件。

- OpenEBS 支持一系列存储引擎，以便开发人员能够部署适合其应用程序设计目标的存储技术。

  像 Cassandra 这样的分布式应用程序可以使用 LocalPV 引擎实现最低延迟的写操作。

  像 MySQL 和 PostgreSQL 这样的独立应用程序可以使用 ZFS 引擎 (cStor) 进行恢复。

  像 Kafka 这样的流媒体应用程序可以使用 NVMe 引擎 Mayastor 在边缘环境中获得最佳性能。

  在各种引擎类型中，OpenEBS 为高可用性、快照、克隆和易管理性提供了一致的框架。

- 管理员和开发人员可以使用 Kubernetes 提供的所有工具来交互和管理 OpenEBS。

# 二、OpenEBS 存储引擎建议

| 应用需求                                         | 存储类型                        | OpenEBS 卷类型                                               |
| ------------------------------------------------ | ------------------------------- | ------------------------------------------------------------ |
| 低时延、高可用性、同步复制、快照、克隆、精简配置 | SSD/ 云存储卷                   | OpenEBS Mayastor                                             |
| 高可用性、同步复制、快照、克隆、精简配置         | 机械 /SSD/ 云存储卷             | OpenEBS cStor                                                |
| 高可用性、同步复制、精简配置                     | 主机路径或外部挂载存储          | OpenEBS Jiva                                                 |
| 低时延、本地 PV                                  | 主机路径或外部挂载存储          | Dynamic Local PV - Hostpath, Dynamic Local PV - Rawfile      |
| 低时延、本地 PV                                  | 本地机械 /SSD/ 云存储卷等块设备 | Dynamic Local PV - Device                                    |
| 低延迟，本地 PV，快照，克隆                      | 本地机械 /SSD/ 云存储卷等块设备 | OpenEBS Dynamic Local PV - ZFS , OpenEBS Dynamic Local PV - LVM |

- 多机环境，如果有额外的块设备（非系统盘块设备）作为数据盘，选用 `OpenEBS Mayastor`、`OpenEBS cStor`
- 多机环境，如果没有额外的块设备（非系统盘块设备）作为数据盘，仅单块系统盘块设备，选用 `OpenEBS Jiva`
- 单机环境，建议本地路径 `Dynamic Local PV - Hostpath, Dynamic Local PV - Rawfile`，由于单机多用于测试环境，数据可靠性要求较低。

# 三、安装

[openobs官方安装文档](https://openebs.io/docs/2.12.x/user-guides/cstor)

在安装openebs之前先去除污点，然后等安装完成再添加回来污点

```
# 去除污点
kubectl taint nodes k8s-master01 node-role.kubernetes.io/master=:NoSchedule-
# 添加污点
kubectl taint nodes k8s-master01 node-role.kubernetes.io/master=:NoSchedule
```

添加helm repo

```
helm repo add openebs https://openebs.github.io/charts
helm repo update
```

安装openebs（这里只会安装Jiva和Local组件）

```
helm install openebs --namespace openebs openebs/openebs --create-namespace --version 3.0.x
```

添加cstor支持

```
helm install openebs --namespace openebs openebs/openebs --create-namespace --set cstor.enabled=true --version 3.2.0
```

查看安装

```
# 查看安装pod
kubectl get pod -n openebs
# 查看安装blockdevice，这里的blockdevice是磁盘，当添加一块未分配的磁盘，就会有值
kubectl get bd -n openebs
```

OpenEBS依赖与iSCSI做存储管理，因此需要先确保您的集群上已有安装openiscsi。 （这里当报错的时候可以安装试试）

```
yum -y install iscsi-initiator-utils
systemctl enable iscsid --now
systemctl start iscsid
```

查看安装状况：

```
[root@k8s-master01 ~]# kubectl get pod -n openebs
NAME                                            READY   STATUS    RESTARTS         AGE
openebs-cstor-admission-server-b74f5487-lkz84   1/1     Running   1 (6m21s ago)    8h
openebs-cstor-csi-controller-0                  6/6     Running   0                4m44s
openebs-cstor-csi-node-4df4w                    2/2     Running   2 (61m ago)      8h
openebs-cstor-csi-node-x8bmt                    2/2     Running   6 (16m ago)      8h
openebs-cstor-csi-node-zzn4k                    2/2     Running   2 (6m21s ago)    8h
openebs-cstor-cspc-operator-84464fb479-fh949    1/1     Running   3 (16m ago)      8h
openebs-cstor-cvc-operator-646f6f676b-xhd44     1/1     Running   2 (16m ago)      46m
openebs-localpv-provisioner-55b65f8b55-zqj29    1/1     Running   13 (6m23s ago)   8h
openebs-ndm-429hl                               1/1     Running   2 (4m28s ago)    8h
openebs-ndm-9kkzd                               1/1     Running   1 (6m21s ago)    8h
openebs-ndm-operator-6c944d87b6-5ddxz           1/1     Running   2 (16m ago)      46m
openebs-ndm-sqnwx                               1/1     Running   6 (15m ago)      8h
```

# 四、添加磁盘

```
[root@k8s-master01 ~]# kubectl get bd -n openebs
NAME                                           NODENAME       SIZE           CLAIMSTATE   STATUS     AGE
blockdevice-57886fae032a3d3638badeb1282dd67e   k8s-node02     21473771008    Unclaimed    Active     53s
blockdevice-d923fc382d96ff6eea7d9ab8efb66224   k8s-master01   21473771008    Unclaimed    Active     11m
blockdevice-e5009ce419c80719025c4cc9409253ab   k8s-node01     21473771008    Unclaimed    Active     33s
```

# 五、配置

## 5.1 创建cStor存储池

cspc.yaml ：

```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: "k8s-master01"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-d923fc382d96ff6eea7d9ab8efb66224"
     poolConfig:
       dataRaidGroupType: "stripe"

   - nodeSelector:
       kubernetes.io/hostname: "k8s-node01"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-e5009ce419c80719025c4cc9409253ab"
     poolConfig:
       dataRaidGroupType: "stripe"

   - nodeSelector:
       kubernetes.io/hostname: "k8s-node02"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-57886fae032a3d3638badeb1282dd67e"
     poolConfig:
       dataRaidGroupType: "stripe"
```

dataRaidGroupType:可以根据您的需要设置为 `stripe` or `mirror` 。下面以配置为stripe为例。 

```
[root@k8s-master01 openebs]# kubectl get CStorPoolCluster -n openebs
NAME              HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES   AGE
cstor-disk-pool                      3                      3                  42s
```

## 5.2 storageclass创建

### 5.2.1 cstor的创建

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cstor-csi-disk
provisioner: cstor.csi.openebs.io
allowVolumeExpansion: true
parameters:
  cas-type: cstor
  # cstorPoolCluster should have the name of the CSPC
  cstorPoolCluster: cstor-disk-pool
  # replicaCount should be <= no. of CSPI created in the selected CSPC
  replicaCount: "3"
```

添加硬盘后查看磁盘情况

```
磁盘 /dev/sdb：21.5 GB, 21474836480 字节，41943040 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：gpt
Disk identifier: ADA9C10B-8C31-4EE2-A29B-F2701E9554DC


#         Start          End    Size  Type            Name
 1         2048     41943006     20G  Linux filesyste OpenEBS_NDM
```

### 5.2.2 jiva 

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: jiva-storageclass
  annotations:
    openebs.io/cas-type: jiva
    cas.openebs.io/config: |
      - name: StoragePool
        value: default
provisioner: openebs.io/provisioner-iscsi
```

### 5.2.3  hostpath

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: localpv-hostpath-sc
  annotations:
    openebs.io/cas-type: local
    cas.openebs.io/config: |
      - name: BasePath
        value: "/var/openebs/local"
      - name: StorageType
        value: "hostpath"
provisioner: openebs.io/local
```

### 5.2.4 device

下面的类型需要添加硬盘

```
磁盘 /dev/sdc：21.5 GB, 21474836480 字节，41943040 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：gpt
Disk identifier: BAD0A706-0A9D-478A-85C6-319224EC5D1F


#         Start          End    Size  Type            Name
 1         2048     41943006     20G  Linux filesyste OpenEBS_NDM
```

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: localpv-device-sc
  annotations:
    openebs.io/cas-type: local
    cas.openebs.io/config: |
      - name: StorageType
        value: "device"
      - name: FSType
        value: ext4
provisioner: openebs.io/local
```

查看：

```
[root@k8s-master01 openebs]# kubectl get sc -n openebs
NAME               PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
cstor-csi-disk     cstor.csi.openebs.io   Delete          Immediate              true                   43s
openebs-device     openebs.io/local       Delete          WaitForFirstConsumer   false                  9h
openebs-hostpath   openebs.io/local       Delete          WaitForFirstConsumer   false                  9h
```

### 5.2.5 cStor、Jiva、LocalPV 特性比较：

| 特性                 | Jiva  | cStor    | Local PV |
| -------------------- | ----- | -------- | -------- |
| 轻量级运行于用户空间 | Yes   | Yes      | Yes      |
| 同步复制             | Yes   | Yes      | No       |
| 适合低容量工作负载   | Yes   | Yes      | Yes      |
| 支持快照，克隆       | Basic | Advanced | No       |
| 数据一致性           | Yes   | Yes      | NA       |
| 使用 Velero 恢复备份 | Yes   | Yes      | Yes      |
| 适合高容量工作负载   | No    | Yes      | Yes      |
| 自动精简配置         |       | Yes      | No       |
| 磁盘池或聚合支持     |       | Yes      | No       |
| 动态扩容             |       | Yes      | Yes      |
| 数据弹性 (RAID 支持) |       | Yes      | No       |
| 接近原生磁盘性能     | No    | No       | Yes      |

大多数场景推荐 `cStor`，因其提供了强大的功能，包括快照 / 克隆、存储池功能（如精简资源调配、按需扩容等）。

`Jiva` 适用于低容量需求的工作负载场景，例如 `5` 到 `50G`。尽管使用 `Jiva` 没有空间限制，但建议将其用于低容量工作负载。`Jiva` 非常易于使用，并提供企业级容器本地存储，而不需要专用硬盘。有快照和克隆功能的需求的场景，优先考虑使用 `cStor` 而不是 `Jiva`。

## 5.3 默认sc

```
kubectl patch storageclass cstor-csi-disk -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```



推荐文章：

https://blog.csdn.net/easylife206/article/details/125213855

https://zhuanlan.zhihu.com/p/519172233