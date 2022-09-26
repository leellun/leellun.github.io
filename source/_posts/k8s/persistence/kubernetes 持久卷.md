---
title: kubernetes 持久卷
date: 2021-08-25 21:14:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
---

# 1 概念

**PersistentVolume（PV）**

是集群中由管理员配置的一段网络存储。它是集群中的资源，就像节点是集群资源一样。PV 是容量插件，如Volumes，但其生命周期独立于使用PV 的任何单个pod。此API 对象捕获存储实现的详细信息，包括NFS，iSCSI 或特定于云提供程序的存储系统。

**PersistentVolumeClaim（PVC）**

是由用户进行存储的请求。它类似于pod。Pod 消耗节点资源，PVC 消耗PV 资源。Pod 可以请求特定级别的资源（CPU 和内存）。声明可以请求特定的大小和访问模式（ ReadWriteOnce、ReadOnlyMany 或 ReadWriteMany.例如，可以一次读/写或多次只读）。

**StorageClass** 

尽管 PersistentVolumeClaim 允许用户消耗抽象的存储资源，常见的情况是针对不同的 问题用户需要的是具有不同属性（如，性能）的 PersistentVolume 卷。 集群管理员需要能够提供不同性质的 PersistentVolume，并且这些 PV 卷之间的差别不仅限于卷大小和访问模式，同时又不能将卷是如何实现的这些细节暴露给用户。 为了满足这类需求，就有了 *存储类（StorageClass）* 资源。

**持久化卷声明的保护**

PVC保护的目的是确保由pod正在使用的PVC不会从系统中移除,因为如果被移除的话可能会导致数据丢失当启用PVC保护alpha功能时,如果用户删除了一个pod正在使用的PVC,则该PVC不会被立即删除。PVC的删除将被推迟,直到PVC不再被任何pod使用

# 2 PV和PVC生命周期

PV 卷是集群中的资源。PVC 申领是对这些资源的请求，也被用来执行对资源的申领检查。 PV 卷和 PVC 申领之间的互动遵循如下生命周期：

## 2.1 制备/供应

PV 卷的制备有两种方式：静态制备或动态制备。

### 静态制备

集群管理员创建若干 PV 卷。这些卷对象带有真实存储的细节信息， 并且对集群用户可用（可见）。PV 卷对象存在于 Kubernetes API 中，可供用户消费（使用）。

### 动态制备

如果管理员所创建的所有静态 PV 卷都无法与用户的 PersistentVolumeClaim 匹配， 集群可以尝试为该 PVC 申领动态制备一个存储卷。 这一制备操作是基于 StorageClass 来实现的：PVC 申领必须请求某个 存储类， 同时集群管理员必须已经创建并配置了该类，这样动态制备卷的动作才会发生。 如果 PVC 申领指定存储类为 `""`，则相当于为自身禁止使用动态制备的卷。

为了基于存储类完成动态的存储制备，集群管理员需要在 API 服务器上启用 `DefaultStorageClass` 准入控制器。 举例而言，可以通过保证 `DefaultStorageClass` 出现在 API 服务器组件的 `--enable-admission-plugins` 标志值中实现这点；该标志的值可以是逗号分隔的有序列表。 

## 2.2 绑定

用户创建一个带有特定存储容量和特定访问模式需求的 PersistentVolumeClaim 对象； 在动态制备场景下，这个 PVC 对象可能已经创建完毕。 主控节点中的控制回路监测新的 PVC 对象，寻找与之匹配的 PV 卷（如果可能的话）， 并将二者绑定到一起。 如果为了新的 PVC 申领动态制备了 PV 卷，则控制回路总是将该 PV 卷绑定到这一 PVC 申领。 否则，用户总是能够获得他们所请求的资源，只是所获得的 PV 卷可能会超出所请求的配置。 一旦绑定关系建立，则 PersistentVolumeClaim 绑定就是排他性的， 无论该 PVC 申领是如何与 PV 卷建立的绑定关系。 PVC 申领与 PV 卷之间的绑定是一种一对一的映射，实现上使用 ClaimRef 来记述 PV 卷与 PVC 申领间的双向绑定关系。

如果找不到匹配的 PV 卷，PVC 申领会无限期地处于未绑定状态。 当与之匹配的 PV 卷可用时，PVC 申领会被绑定。 例如，即使某集群上制备了很多 50 Gi 大小的 PV 卷，也无法与请求 100 Gi 大小的存储的 PVC 匹配。当新的 100 Gi PV 卷被加入到集群时， 该 PVC 才有可能被绑定。

## 2.3 使用Using

Pod 将 PVC 申领当做存储卷来使用。集群会检视 PVC 申领，找到所绑定的卷，并为Pod 挂载该卷。对于支持多种访问模式的卷，用户要在 Pod 中以卷的形式使用申领时指定期望的访问模式。用户通过在 Pod 的 `volumes` 块中包含 `persistentVolumeClaim` 节区来调度 Pod，访问所申领的 PV 卷。

一旦用户有了申领对象并且该申领已经被绑定，则所绑定的 PV 卷在用户仍然需要它期间一直属于该用户。

## 2.4 保护使用中的存储对象

目的是确保仍被 Pod 使用的 PersistentVolumeClaim（PVC）对象及其所绑定的 PersistentVolume（PV）对象在系统中不会被删除，因为这样做可能会引起数据丢失。用户删除pvc 来回收存储资源，pv 将变成“released”状态。由于还保留着之前的数据，这些数据需要根据不同的策略来处理，否则这些存储资源无法被其他pvc 使用。

如果用户删除被某 Pod 使用的 PVC 对象，该 PVC 申领不会被立即移除。 PVC 对象的移除会被推迟，直至其不再被任何 Pod 使用。 此外，如果管理员删除已绑定到某 PVC 申领的 PV 卷，该 PV 卷也不会被立即移除。 PV 对象的移除也要推迟到该 PV 不再绑定到 PVC。 

当 PVC 的状态为 `Terminating` 且其 `Finalizers` 列表中包含 `kubernetes.io/pvc-protection` 时，PVC 对象是处于被保护状态的。 

```yaml
kubectl describe pvc hostpath
```

当 PV 对象的状态为 `Terminating` 且其 `Finalizers` 列表中包含 `kubernetes.io/pv-protection` 时，PV 对象是处于被保护状态的。 

```
kubectl describe pv task-pv-volume
```

## 2.5 回收（Reclaiming）

当用户不再使用其存储卷时，他们可以从 API 中将 PVC 对象删除， 从而允许该资源被回收再利用。PersistentVolume 对象的回收策略告诉集群， 当其被从申领中释放时如何处理该数据卷。 目前，数据卷可以被 Retained（保留）、Recycled（回收）或 Deleted（删除）。

### 2.5.1 保留（Retain）

persistentVolumeReclaimPolicy: Retain

回收策略 `Retain` 使得用户可以手动回收资源。当 PersistentVolumeClaim 对象被删除时，PersistentVolume 卷仍然存在，对应的数据卷被视为"已释放（released）"。 由于卷上仍然存在这前一申领人的数据，该卷还不能用于其他申领。 管理员可以通过下面的步骤来手动回收该卷：

1. 删除 PersistentVolume 对象。与之相关的、位于外部基础设施中的存储资产 （例如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）在 PV 删除之后仍然存在。
2. 根据情况，手动清除所关联的存储资产上的数据。
3. 手动删除所关联的存储资产。

如果你希望重用该存储资产，可以基于存储资产的定义创建新的 PersistentVolume 卷对象。

### 2.5.2 删除（Delete）

persistentVolumeReclaimPolicy: Delete

对于支持 `Delete` 回收策略的卷插件，删除动作会将 PersistentVolume 对象从 Kubernetes 中移除，同时也会从外部基础设施（如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）中移除所关联的存储资产。 动态制备的卷会继承其 StorageClass 中设置的回收策略， 该策略默认为 `Delete`。管理员需要根据用户的期望来配置 StorageClass； 否则 PV 卷被创建之后必须要被编辑或者修补。

### 2.5.3 回收（Recycle）

persistentVolumeReclaimPolicy: Recycle

目前的回收策略有：

- Retain -- 手动回收
- Recycle -- 基本擦除 (`rm -rf /thevolume/*`)
- Delete -- 诸如 AWS EBS、GCE PD、Azure Disk 或 OpenStack Cinder 卷这类关联存储资产也被删除

目前，仅 NFS 和 HostPath 支持回收（Recycle）。 AWS EBS、GCE PD、Azure Disk 和 Cinder 卷都支持删除（Delete）。

如果下层的卷插件支持，回收策略 `Recycle` 会在卷上执行一些基本的擦除 （`rm -rf /thevolume/*`）操作，之后允许该卷用于新的 PVC 申领。

不过，使用 Kubernetes 控制器管理器命令行参数来配置一个定制的回收器（Recycler） Pod 模板。此定制的回收器 Pod 模板必须包含一个 `volumes` 规约，如下例所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-recycler
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    hostPath:
      path: /data
  containers:
  - name: pv-recycler
    image: "k8s.gcr.io/busybox"
    command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
    volumeMounts:
    - name: vol
      mountPath: /scrub
```

定制回收器 Pod 模板中在 `volumes` 部分所指定的特定路径要替换为正被回收的卷的路径。 

操作：

```
[root@k8s-master01 storage]# vi pv-recycler.yaml
[root@k8s-master01 storage]# kubectl create -f pv-recycler.yaml 
[root@k8s-master01 storage]# kubectl get pod
NAME          READY   STATUS      RESTARTS   AGE
pv-recycler   0/1     Completed   0          2m43s
```

等pv-recycler状态为Completed时，/data文件夹下的文件就全部删完了。

## 2.6 PersistentVolume 删除保护 finalizer

可以在 PersistentVolume 上添加终结器（Finalizer）， 以确保只有在删除对应的存储后才删除具有 `Delete` 回收策略的 PersistentVolume。

新引入的 `kubernetes.io/pv-controller` 和 `external-provisioner.volume.kubernetes.io/finalizer` 终结器仅会被添加到动态制备的卷上。

- 终结器 `kubernetes.io/pv-controller` 会被添加到树内插件卷上。
- 终结器 `external-provisioner.volume.kubernetes.io/finalizer` 会被添加到 CSI 卷上。

下面是一个例子： 

- 为特定的树内卷插件启用 `CSIMigration` 特性将删除 `kubernetes.io/pv-controller` 终结器， 同时添加 `external-provisioner.volume.kubernetes.io/finalizer` 终结器。 
- 同样，禁用 `CSIMigration` 将删除 `external-provisioner.volume.kubernetes.io/finalizer` 终结器， 同时添加 `kubernetes.io/pv-controller` 终结器。 

## 2.7 预留 PersistentVolume

通过在 PersistentVolumeClaim 中指定 PersistentVolume，你可以声明该特定 PV 与 PVC 之间的绑定关系。如果该 PersistentVolume 存在且未被通过其 `claimRef` 字段预留给 PersistentVolumeClaim，则该 PersistentVolume 会和该 PersistentVolumeClaim 绑定到一起。 

绑定操作不会考虑某些卷匹配条件是否满足，包括节点亲和性等等。 控制面仍然会检查存储类、 访问模式和所请求的存储尺寸都是合法的。 

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
  namespace: foo
spec:
  storageClassName: "" # 此处须显式设置空字符串，否则会被设置为默认的 StorageClass
  volumeName: foo-pv
  ...
```

此方法无法对 PersistentVolume 的绑定特权做出任何形式的保证。 如果有其他 PersistentVolumeClaim 可以使用你所指定的 PV， 则你应该首先预留该存储卷。你可以将 PV 的 `claimRef` 字段设置为相关的 PersistentVolumeClaim 以确保其他 PVC 不会绑定到该 PV 卷。 

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: foo-pv
spec:
  storageClassName: ""
  claimRef:
    name: foo-pvc
    namespace: foo
  ...
```

如果你想要使用 `claimPolicy` 属性设置为 `Retain` 的 PersistentVolume 卷时， 包括你希望复用现有的 PV 卷时，这点是很有用的 

## 2.8 扩充 PVC 申领

现在，对扩充 PVC 申领的支持默认处于被启用状态。你可以扩充以下类型的卷：

- azureDisk
- azureFile
- awsElasticBlockStore
- cinder (deprecated)
- csi
- flexVolume (deprecated)
- gcePersistentDisk
- glusterfs
- rbd
- portworxVolume

只有当 PVC 的存储类中将 `allowVolumeExpansion` 设置为 true 时，你才可以扩充该 PVC 申领。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gluster-vol-default
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://192.168.10.100:8080"
  restuser: ""
  secretNamespace: ""
  secretName: ""
allowVolumeExpansion: true
```

如果要为某 PVC 请求较大的存储卷，可以编辑 PVC 对象，设置一个更大的尺寸值。 这一编辑操作会触发为下层 PersistentVolume 提供存储的卷的扩充。 Kubernetes 不会创建新的 PV 卷来满足此申领的请求。 与之相反，现有的卷会被调整大小。 

**重设包含文件系统的卷的大小**

只有卷中包含的文件系统是 XFS、Ext3 或者 Ext4 时，你才可以重设卷的大小.

当卷中包含文件系统时，只有在 Pod 使用 `ReadWrite` 模式来使用 PVC 申领的情况下才能重设其文件系统的大小。 文件系统扩充的操作或者是在 Pod 启动期间完成，或者在下层文件系统支持在线 扩充的前提下在 Pod 运行期间完成。

如果 FlexVolumes 的驱动将 `RequiresFSResize` 能力设置为 `true`，则该 FlexVolume 卷可以在 Pod 重启期间调整大小。

**重设使用中 PVC 申领的大小**

必须在执行扩展操作之前创建一个使用该 PVC 的 Pod。

```
FlexVolume 卷的重设大小只能在下层驱动支持重设大小的时候才可进行。
 扩充 EBS 卷的操作非常耗时。同时还存在另一个配额限制： 每 6 小时只能执行一次（尺寸）修改操作。
```

**处理扩充卷过程中的失败**

如果扩充下层存储的操作失败，集群管理员可以手动地恢复 PVC 申领的状态并 取消重设大小的请求。否则，在没有管理员干预的情况下，控制器会反复重试 重设大小的操作。

1. 将绑定到 PVC 申领的 PV 卷标记为 `Retain` 回收策略；
2. 删除 PVC 对象。由于 PV 的回收策略为 `Retain`，我们不会在重建 PVC 时丢失数据。
3. 删除 PV 规约中的 `claimRef` 项，这样新的 PVC 可以绑定到该卷。 这一操作会使得 PV 卷变为 "可用（Available）"。
4. 使用小于 PV 卷大小的尺寸重建 PVC，设置 PVC 的 `volumeName` 字段为 PV 卷的名称。 这一操作将把新的 PVC 对象绑定到现有的 PV 卷。
5. 不要忘记恢复 PV 卷上设置的回收策略。

# 3 持久卷类型

```
awsElasticBlockStore - AWS 弹性块存储（EBS）
azureDisk - Azure Disk
azureFile - Azure File
cephfs - CephFS volume
cinder - Cinder （OpenStack 块存储） (弃用)
csi - 容器存储接口 (CSI)
fc - Fibre Channel (FC) 存储
flexVolume - FlexVolume
flocker - Flocker 存储
gcePersistentDisk - GCE 持久化盘
glusterfs - Glusterfs 卷
hostPath - HostPath 卷 （仅供单节点测试使用；不适用于多节点集群； 请尝试使用 local 卷作为替代）
iscsi - iSCSI (SCSI over IP) 存储
local - 节点上挂载的本地存储设备
nfs - 网络文件系统 (NFS) 存储
photonPersistentDisk - Photon 控制器持久化盘。 （这个卷类型已经因对应的云提供商被移除而被弃用）。
portworxVolume - Portworx 卷
quobyte - Quobyte 卷
rbd - Rados 块设备 (RBD) 卷
scaleIO - ScaleIO 卷 (弃用)
storageos - StorageOS 卷
vsphereVolume - vSphere VMDK 卷

以下的持久卷已被弃用。这意味着当前仍是支持的，但是 Kubernetes 将来的发行版会将其移除。

cinder - Cinder（OpenStack 块存储）（于 v1.18 弃用）
flexVolume - FlexVolume （于 v1.23 弃用）
flocker - Flocker 存储（于 v1.22 弃用）
quobyte - Quobyte 卷 （于 v1.22 弃用）
storageos - StorageOS 卷（于 v1.22 弃用）
旧版本的 Kubernetes 仍支持这些“树内（In-Tree）”持久卷类型：

photonPersistentDisk - Photon 控制器持久化盘。（v1.15 之后 不可用）
scaleIO - ScaleIO 卷（v1.21 之后 不可用）
```

# 4 持久卷

每个 PV 对象都包含 spec 部分和 status 部分，分别对应卷的规约和状态。 PersistentVolume 对象的名称必须是合法的 DNS 子域名.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

## 4.1 容量

每个 PV 卷都有确定的存储容量。 容量属性是使用 PV 对象的 `capacity` 属性来设置的。目前，存储大小是可以设置和请求的唯一资源。目前，存储大小是可以设置和请求的唯一资源。 未来可能会包含 IOPS、吞吐量等属性。 

## 4.2 卷模式

针对 PV 持久卷，Kubernetes 支持两种卷模式（`volumeModes`）：`Filesystem（文件系统）` 和 `Block（块）`。 `volumeMode` 是一个可选的 API 参数。 如果该参数被省略，默认的卷模式是 `Filesystem`。

`volumeMode` 属性设置为 `Filesystem` 的卷会被 Pod *挂载（Mount）* 到某个目录。 如果卷的存储来自某块设备而该设备目前为空，Kuberneretes 会在第一次挂载卷之前 在设备上创建文件系统。

volumeMode` 设置为 `Block。这类卷以块设备的方式交给 Pod 使用，其上没有任何文件系统。Pod 和 卷之间不存在文件系统层。

## 4.3 访问模式

| 访问模式         | 描述                                                         | 命令行接口 |
| ---------------- | ------------------------------------------------------------ | ---------- |
| ReadWriteOnce    | 卷可以被一个节点以读写方式挂载。 ReadWriteOnce 访问模式也允许运行在同一节点上的多个 Pod 访问卷。 | RWO        |
| ReadOnlyMany     | 该volume可以被多个节点以只读方式映射                         | ROX        |
| ReadWriteMany    | 该volume只能被多个节点以读写的方式映射                       | RWX        |
| ReadWriteOncePod | 卷可以被单个 Pod 以读写方式挂载。 如果你想确保整个集群中只有一个 Pod 可以读取或写入该 PVC， 请使用 ReadWriteOncePod 访问模式。这只支持 CSI 卷以及需要 Kubernetes 1.22 以上版本。 | RWOP       |

**卷插件访问模式**

| 卷插件               | ReadWriteOnce | ReadOnlyMany | ReadWriteMany                    | ReadWriteOncePod |
| -------------------- | ------------- | ------------ | -------------------------------- | ---------------- |
| AWSElasticBlockStore | ✓             | -            | -                                | -                |
| AzureFile            | ✓             | ✓            | ✓                                | -                |
| AzureDisk            | ✓             | -            | -                                | -                |
| CephFS               | ✓             | ✓            | ✓                                | -                |
| Cinder               | ✓             | -            | -                                | -                |
| CSI                  | 取决于驱动    | 取决于驱动   | 取决于驱动                       | 取决于驱动       |
| FC                   | ✓             | ✓            | -                                | -                |
| FlexVolume           | ✓             | ✓            | 取决于驱动                       | -                |
| Flocker              | ✓             | -            | -                                | -                |
| GCEPersistentDisk    | ✓             | ✓            | -                                | -                |
| Glusterfs            | ✓             | ✓            | ✓                                | -                |
| HostPath             | ✓             | -            | -                                | -                |
| iSCSI                | ✓             | ✓            | -                                | -                |
| Quobyte              | ✓             | ✓            | ✓                                | -                |
| NFS                  | ✓             | ✓            | ✓                                | -                |
| RBD                  | ✓             | ✓            | -                                | -                |
| VsphereVolume        | ✓             | -            | - （Pod 运行于同一节点上时可行） | -                |
| PortworxVolume       | ✓             | -            | ✓                                | -                |
| StorageOS            | ✓             | -            | -                                | -                |

在某些场合下，卷访问模式也会限制 PersistentVolume 可以挂载的位置。 卷访问模式并**不会**在存储已经被挂载的情况下为其实施写保护。 即使访问模式设置为 ReadWriteOnce、ReadOnlyMany 或 ReadWriteMany，它们也不会对卷形成限制。 

## 4.4 类storageClassName

每个 PV 可以属于某个类（Class），通过将其 `storageClassName` 属性设置为某个 [StorageClass](https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/) 的名称来指定。 特定类的 PV 卷只能绑定到请求该类存储卷的 PVC 申领。 未设置 `storageClassName` 的 PV 卷没有类设定，只能绑定到那些没有指定特定存储类的 PVC 申领。 

## 4.5 挂载选项

Kubernetes 管理员可以指定持久卷被挂载到节点上时使用的附加挂载选项。 

以下卷类型支持挂载选项：

- `awsElasticBlockStore`
- `azureDisk`
- `azureFile`
- `cephfs`
- `cinder` (**已弃用**于 v1.18)
- `gcePersistentDisk`
- `glusterfs`
- `iscsi`
- `nfs`
- `quobyte` (**已弃用**于 v1.22)
- `rbd`
- `storageos` (**已弃用**于 v1.22)
- `vsphereVolume`

Kubernetes 不对挂载选项执行合法性检查。如果挂载选项是非法的，挂载就会失败。 早前，Kubernetes 使用注解 `volume.beta.kubernetes.io/mount-options` 而不是 `mountOptions` 属性。 

## 4.6 节点亲和性

每个 PV 卷可以通过设置 节点亲和性来定义一些约束，进而限制从哪些节点上可以访问此卷。 使用这些卷的 Pod 只会被调度到节点亲和性规则所选择的节点上执行。 volumenodeaffinity

## 4.7 PV 卷阶段状态 

- Available – 资源尚未被claim 使用

- Bound – 卷已经被绑定到claim 了
- Released – claim 被删除，卷处于释放状态，但未被集群回收。
- Failed – 卷自动回收失败

# 5 PersistentVolumeClaims

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes: #访问模式
    - ReadWriteOnce
  volumeMode: Filesystem #卷模式
  resources: #资源
    requests:
      storage: 8Gi
  storageClassName: slow
  selector: #选择
    matchLabels: #标签匹配
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

**访问模式**

申领在请求具有特定访问模式的存储时，使用与卷相同的访问模式

**卷模式**

申领使用与卷相同的约定来表明是将卷作为文件系统还是块设备来使用。

**资源**

申领和 Pod 一样，也可以请求特定数量的资源。在这个上下文中，请求的资源是存储。 卷和申领都使用相同的资源模型。

**选择算符**

申领可以设置标签选择算符来进一步过滤卷集合。只有标签与选择算符相匹配的卷能够绑定到申领上。 选择算符包含两个字段：

- `matchLabels` - 卷必须包含带有此值的标签
- `matchExpressions` - 通过设定键（key）、值列表和操作符（operator） 来构造的需求。合法的操作符有 In、NotIn、Exists 和 DoesNotExist。

来自 `matchLabels` 和 `matchExpressions` 的所有需求都按逻辑与的方式组合在一起。 这些需求都必须被满足才被视为匹配。

**类**

申领可以通过为 `storageClassName` 属性设置 StorageClass的名称来请求特定的存储类。 只有所请求的类的 PV 卷，即 `storageClassName` 值与 PVC 设置相同的 PV 卷， 才能绑定到 PVC 申领。

PVC 申领不必一定要请求某个类。如果 PVC 的 `storageClassName` 属性值设置为 `""`， 则被视为要请求的是没有设置存储类的 PV 卷，因此这一 PVC 申领只能绑定到未设置存储类的 PV 卷（未设置注解或者注解值为 `""` 的 PersistentVolume（PV）对象在系统中不会被删除， 因为这样做可能会引起数据丢失。未设置 `storageClassName` 的 PVC 与此大不相同， 也会被集群作不同处理。具体筛查方式取决于 [`DefaultStorageClass` 准入控制器插件](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass) 是否被启用。

- 如果准入控制器插件被启用，则管理员可以设置一个默认的 StorageClass。 所有未设置 `storageClassName` 的 PVC 都只能绑定到隶属于默认存储类的 PV 卷。 设置默认 StorageClass 的工作是通过将对应 StorageClass 对象的注解 `storageclass.kubernetes.io/is-default-class` 赋值为 `true` 来完成的。 如果管理员未设置默认存储类，集群对 PVC 创建的处理方式与未启用准入控制器插件时相同。 如果设定的默认存储类不止一个，准入控制插件会禁止所有创建 PVC 操作。
- 如果准入控制器插件被关闭，则不存在默认 StorageClass 的说法。 所有未设置 `storageClassName` 的 PVC 都只能绑定到未设置存储类的 PV 卷。 在这种情况下，未设置 `storageClassName` 的 PVC 与 `storageClassName` 设置为 `""` 的 PVC 的处理方式相同。

取决于安装方法，默认的 StorageClass 可能在集群安装期间由插件管理器（Addon Manager）部署到集群中。

当某 PVC 除了请求 StorageClass 之外还设置了 `selector`，则这两种需求会按逻辑与关系处理： 只有隶属于所请求类且带有所请求标签的 PV 才能绑定到 PVC。

**说明：** 目前，设置了非空 `selector` 的 PVC 对象无法让集群为其动态制备 PV 卷。

早前，Kubernetes 使用注解 `volume.beta.kubernetes.io/storage-class` 而不是 `storageClassName` 属性。这一注解目前仍然起作用，不过在将来的 Kubernetes 发布版本中该注解会被彻底废弃。

# 6 使用申领作为卷

Pod 将申领作为卷来使用，并藉此访问存储资源。 申领必须位于使用它的 Pod 所在的同一名字空间内。 集群在 Pod 的名字空间中查找申领，并使用它来获得申领所使用的 PV 卷。 之后，卷会被挂载到宿主上并挂载到 Pod 中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

### 关于名字空间的说明

PersistentVolume 卷的绑定是排他性的。 由于 PersistentVolumeClaim 是名字空间作用域的对象，使用 "Many" 模式（`ROX`、`RWX`）来挂载申领的操作只能在同一名字空间内进行。

### 类型为 `hostpath` 的 PersistentVolume

`hostPath` PersistentVolume 使用节点上的文件或目录来模拟网络附加（network-attached）存储。 相关细节可参阅[`hostPath` 卷示例](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume)。

## 原始块卷支持

**特性状态：** `Kubernetes v1.18 [stable]`

以下卷插件支持原始块卷，包括其动态制备（如果支持的话）的卷：

- AWSElasticBlockStore
- AzureDisk
- CSI
- FC （光纤通道）
- GCEPersistentDisk
- iSCSI
- Local 卷
- OpenStack Cinder
- RBD （Ceph 块设备）
- VsphereVolume

# 7使用原始块卷的持久卷

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
```

### 申请原始块卷的 PVC 申领

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
```

### 在容器中添加原始块设备路径的 Pod 规约

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```

**说明：** 向 Pod 中添加原始块设备时，你要在容器内设置设备路径而不是挂载路径。

### 绑定块卷

如果用户通过 PersistentVolumeClaim 规约的 `volumeMode` 字段来表明对原始块设备的请求， 绑定规则与之前版本中未在规约中考虑此模式的实现略有不同。 下面列举的表格是用户和管理员可以为请求原始块设备所作设置的组合。 此表格表明在不同的组合下卷是否会被绑定。

静态制备卷的卷绑定矩阵：

| PV volumeMode | PVC volumeMode | Result |
| ------------- | -------------- | ------ |
| 未指定        | 未指定         | 绑定   |
| 未指定        | Block          | 不绑定 |
| 未指定        | Filesystem     | 绑定   |
| Block         | 未指定         | 不绑定 |
| Block         | Block          | 绑定   |
| Block         | Filesystem     | 不绑定 |
| Filesystem    | Filesystem     | 绑定   |
| Filesystem    | Block          | 不绑定 |
| Filesystem    | 未指定         | 绑定   |

**说明：** Alpha 发行版本中仅支持静态制备的卷。 管理员需要在处理原始块设备时小心处理这些值。

# 8 对卷快照及从卷快照中恢复卷的支持

**特性状态：** `Kubernetes v1.20 [stable]`

卷快照（Volume Snapshot）仅支持树外 CSI 卷插件。 有关细节可参阅[卷快照](https://kubernetes.io/zh-cn/docs/concepts/storage/volume-snapshots/)文档。 树内卷插件被弃用。你可以查阅[卷插件 FAQ](https://git.k8s.io/community/sig-storage/volume-plugin-faq.md) 了解已弃用的卷插件。

### 基于卷快照创建 PVC 申领

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
spec:
  storageClassName: csi-hostpath-sc
  dataSource:
    name: new-snapshot-test
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

# 9 卷克隆

卷克隆功能特性仅适用于 CSI 卷插件。

### 基于现有 PVC 创建新的 PVC 申领

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  storageClassName: my-csi-plugin
  dataSource:
    name: existing-src-pvc-name
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

# 10 卷填充器（Populator）与数据源

**特性状态：** `Kubernetes v1.24 [beta]`

Kubernetes 支持自定义的卷填充器。要使用自定义的卷填充器，你必须为 kube-apiserver 和 kube-controller-manager 启用 `AnyVolumeDataSource` 特性门控。

卷填充器利用了 PVC 规约字段 `dataSourceRef`。 不像 `dataSource` 字段只能包含对另一个持久卷申领或卷快照的引用， `dataSourceRef` 字段可以包含对同一命名空间中任何对象的引用（不包含除 PVC 以外的核心资源）。 对于启用了特性门控的集群，使用 `dataSourceRef` 比 `dataSource` 更好。

## 数据源引用

`dataSourceRef` 字段的行为与 `dataSource` 字段几乎相同。 如果其中一个字段被指定而另一个字段没有被指定，API 服务器将给两个字段相同的值。 这两个字段都不能在创建后改变，如果试图为这两个字段指定不同的值，将导致验证错误。 因此，这两个字段将总是有相同的内容。

在 `dataSourceRef` 字段和 `dataSource` 字段之间有两个用户应该注意的区别：

- `dataSource` 字段会忽略无效的值（如同是空值）， 而 `dataSourceRef` 字段永远不会忽略值，并且若填入一个无效的值，会导致错误。 无效值指的是 PVC 之外的核心对象（没有 apiGroup 的对象）。
- `dataSourceRef` 字段可以包含不同类型的对象，而 `dataSource` 字段只允许 PVC 和卷快照。

用户应该始终在启用了特性门控的集群上使用 `dataSourceRef`，而在没有启用特性门控的集群上使用 `dataSource`。 在任何情况下都没有必要查看这两个字段。 这两个字段的值看似相同但是语义稍微不一样，是为了向后兼容。 特别是混用旧版本和新版本的控制器时，它们能够互通。

## 使用卷填充器

卷填充器是能创建非空卷的[控制器](https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/)， 其卷的内容通过一个自定义资源决定。 用户通过使用 `dataSourceRef` 字段引用自定义资源来创建一个被填充的卷：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: populated-pvc
spec:
  dataSourceRef:
    name: example-name
    kind: ExampleDataSource
    apiGroup: example.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

因为卷填充器是外部组件，如果没有安装所有正确的组件，试图创建一个使用卷填充器的 PVC 就会失败。 外部控制器应该在 PVC 上产生事件，以提供创建状态的反馈，包括在由于缺少某些组件而无法创建 PVC 的情况下发出警告。

你可以把 alpha 版本的[卷数据源验证器](https://github.com/kubernetes-csi/volume-data-source-validator) 控制器安装到你的集群中。 如果没有填充器处理该数据源的情况下，该控制器会在 PVC 上产生警告事件。 当一个合适的填充器被安装到 PVC 上时，该控制器的职责是上报与卷创建有关的事件，以及在该过程中发生的问题。

## 编写可移植的配置

如果你要编写配置模板和示例用来在很多集群上运行并且需要持久性存储，建议你使用以下模式：

- 将 PersistentVolumeClaim 对象包含到你的配置包（Bundle）中，和 Deployment 以及 ConfigMap 等放在一起。
- 不要在配置中包含 PersistentVolume 对象，因为对配置进行实例化的用户很可能 没有创建 PersistentVolume 的权限。

- 为用户提供在实例化模板时指定存储类名称的能力。

  仍按用户提供存储类名称，将该名称放到 `persistentVolumeClaim.storageClassName` 字段中。 这样会使得 PVC 在集群被管理员启用了存储类支持时能够匹配到正确的存储类，

  如果用户未指定存储类名称，将 `persistentVolumeClaim.storageClassName` 留空（nil）。 这样，集群会使用默认 `StorageClass` 为用户自动制备一个存储卷。 很多集群环境都配置了默认的 `StorageClass`，或者管理员也可以自行创建默认的 `StorageClass`。

- 在你的工具链中，监测经过一段时间后仍未被绑定的 PVC 对象，要让用户知道这些对象， 因为这可能意味着集群不支持动态存储（因而用户必须先创建一个匹配的 PV），或者 集群没有配置存储系统（因而用户无法配置需要 PVC 的工作负载配置）。

# 11 StorageClasses

每个StorageClass包含字段provisioninger和参数，当属于类的PersistentVolume需要动态配置时使用。 

StorageClass对象的名称很重要，用户可以如何请求特定的类。 管理员在首次创建StorageClass对象时设置类的名称和其他参数，并且在创建对象后无法更新对象。 

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs # 存储类有一个供应商，它确定用于配置PV的卷插件。 必须指定此字段。
parameters: # 存储类具有描述属于存储类的卷的参数。
  type: gp2
```

| 参数        | 类型   | 说明                                                         |
| ----------- | ------ | ------------------------------------------------------------ |
| Provisioner | string | 存储类有一个供应商，它确定用于配置PV的卷插件。 必须指定此字段 |
| parameters  | object | 存储类具有描述属于存储类的卷的参数。 取决于供应商，可以接受不同的参数。 |

# 附录：

[官方文档持久卷](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes)

| 参数                                                         | 类型   | 说明                                                         |
| ------------------------------------------------------------ | ------ | ------------------------------------------------------------ |
| spec.nfs                                                     | object | nfs服务器                                                    |
| spec.nfs.path                                                | string | 挂载路径                                                     |
| spec.nfs.server                                              | string | nfs服务器地址                                                |
| spec.volumeMode                                              | string | Filesystem（文件系统）` 和 `Block（块）                      |
| spec.volumeName                                              | string | pv                                                           |
| spec.accessModes                                             | list   | 访问模式                                                     |
| spec.capacity                                                | object | 通常，PV将具有特定的存储容量。 这是使用PV的容量属性设置的。  |
| spec.capacity.storage                                        | string | 使用PV的容量属性设置                                         |
| spec.persistentVolumeReclaimPolicy                           | string | 回收策略     保留（Retain），回收（Recycle）和删除（Delete）。 |
| spec.storageClassName                                        | string | 通过将storageClassName属性设置为StorageClass的名称来指定。 特定类的PV只能绑定到请求该类的PVC。 没有storageClassName的PV没有类，只能绑定到不需要特定类的PVC。                 注：使用了(过去)注释volume.beta.kubernetes.io/storage-class 仍然可用，不建议 |
| spec.gcePersistentDisk                                       | object | gcePersistentDisk  持久卷                                    |
| spec.gcePersistentDisk.fsType                                | string | 文件系统类型 ext4 ext3                                       |
| spec.gcePersistentDisk.pdName                                | string | gcePersistentDisk 名称                                       |
| spec.resources                                               | object | 声明（如pod）可以请求特定数量的资源。                        |
| spec.resources.requests                                      | object | 可以请求特定数量的资源                                       |
| spec.resources.requests.storage                              | string | 存储限制 单位MIB ,GIB                                        |
| spec.volumes[]                                               | list   | 在该pod上定义的共享存储卷列表                                |
| spec.volumes[].name                                          | string |                                                              |
| spec.volumes[].persistentVolumeClaim                         | object | 命名空间对象                                                 |
| spec.volumes[].persistentVolumeClaim.claimName               | string | PersistentVolumeClaim name                                   |
| spec.volumeMounts[]                                          | list   | 指定容器内部的存储卷配置                                     |
| spec.volumeMounts[].name                                     | string | 名称                                                         |
| spec.volumeMounts[].mountPath                                | string | 挂载路径                                                     |
| metadata.annotations                                         | object | 持久卷上的注释                                               |
| metadata.annotations.volume.beta.kubernetes.io/ mount-options | string | 使用持久卷上的注释volume.beta.kubernetes.io/mount-options来指定安装选项 |
| spec.selector                                                | object | 标签选择器                                                   |
| spec.selector.matchLabels                                    |        | 卷必须具有带此值的标签 key: value                            |
| spec.selector.matchExpressions[]                             |        | 通过指定关键字和值的关键字，值列表和运算符所做的要求列表。 有效运算符包括In，NotIn，Exists和DoesNotExist |
