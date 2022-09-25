---
title: k8s pvc作为存储的简单示例
date: 2021-08-23 21:14:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - pv
  - pvc
---

# 一、pod直接使用pvc

Pod 使用 PersistentVolumeClaim 作为存储:

1. 创建由物理存储支持的 PersistentVolume，未与任何 Pod 关联。
2. 创建一个 PersistentVolumeClaim， 它将自动绑定到合适的 PersistentVolume。
3. 创建一个使用 PersistentVolumeClaim 作为存储的 Pod。

## 1.1 创建 PersistentVolume

创建一个 *hostPath* 类型的 PersistentVolume。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data"
```

执行 ：

```
# 执行上面的清单配置pv
$ kubectl apply -f pv-volume.yaml
$ kubectl get pv task-pv-volume
NAME             CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
task-pv-volume   10Gi       RWO           Retain          Available             manual                   4s
```

输出结果显示该 PersistentVolume 的`状态（STATUS）` 为 `Available`。 这意味着它还没有被绑定给 PersistentVolumeClaim。

## 1.2 创建 PersistentVolumeClaim

创建一个 PersistentVolumeClaim，它请求至少 3 GB 容量的卷， 该卷至少可以为一个节点提供读写访问。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

执行：

```shell
$ kubectl create -f pv-claim.yaml
# 现在输出的 STATUS 为 Bound。
$ kubectl get pv task-pv-volume
NAME             CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                   STORAGECLASS   REASON    AGE
task-pv-volume   10Gi       RWO           Retain          Bound     default/task-pv-claim   manual                   2m
$ kubectl get pvc task-pv-claim
NAME            STATUS    VOLUME           CAPACITY   ACCESSMODES   STORAGECLASS   AGE
task-pv-claim   Bound     task-pv-volume   10Gi       RWO           manual         30s
```

输出结果表明该 PersistentVolumeClaim 绑定了你的 PersistentVolume `task-pv-volume`。

创建 PersistentVolumeClaim 之后，Kubernetes 控制平面将查找满足申领要求的 PersistentVolume。 如果控制平面找到具有相同 StorageClass 的适当的 PersistentVolume， 则将 PersistentVolumeClaim 绑定到该 PersistentVolume 上。

匹配原则：

- 比较storageClassName是否相同
- 比较accessModes是否相同
- 比较storage是否满足

## 1.3 创建 Pod

创建一个 Pod， 该 Pod 使用你的 PersistentVolumeClaim 作为存储卷。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: harborcloud.com/library/nginx:1.9.1
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

执行：

```
kubectl apply -f pv-pod.yaml
```

打开一个 Shell 访问 Pod 中的容器：

```shell
kubectl exec -it task-pv-pod -- /bin/bash
```

在 Shell 中，验证 nginx 是否正在从 hostPath 卷提供 `index.html` 文件：

```
# 一定要在上一步 "kubectl exec" 所返回的 Shell 中执行下面三个命令
root@task-pv-pod:/# apt-get update
root@task-pv-pod:/# apt-get install curl
root@task-pv-pod:/# curl localhost
```

输出结果是你之前写到 hostPath 卷中的 `index.html` 文件中的内容：

```
Hello from Kubernetes storage
```

如果你看到此消息，则证明你已经成功地配置了 Pod 使用 PersistentVolumeClaim 的存储。

## 1.4 清理

删除 Pod、PersistentVolumeClaim 和 PersistentVolume 对象：

```shell
$ kubectl delete pod task-pv-pod
$ kubectl delete pvc task-pv-claim
$ kubectl delete pv task-pv-volume
```

清理顺序和创建顺序相反 pod->pvc->pv

在节点的 Shell 上，删除你所创建的目录和文件：

```shell
$ rm /data/index.html
$ rm /data
```

## 1.5 在两个地方挂载相同的 persistentVolume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
    - name: test
      image: nginx
      volumeMounts:
        # 网站数据挂载
        - name: config
          mountPath: /usr/share/nginx/html
          subPath: html
        # Nginx 配置挂载
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
  volumes:
    - name: config
      persistentVolumeClaim:
        claimName: test-nfs-claim
```

你可以在 nginx 容器上执行两个卷挂载:

`/usr/share/nginx/html` 用于静态网站 `/etc/nginx/nginx.conf` 作为默认配置

## 1.6 访问控制

使用组 ID（GID）配置的存储仅允许 Pod 使用相同的 GID 进行写入。 GID 不匹配或缺失将会导致无权访问错误。 为了减少与用户的协调，管理员可以对 PersistentVolume 添加 GID 注解。 这样 GID 就能自动添加到使用 PersistentVolume 的任何 Pod 中。

使用 `pv.beta.kubernetes.io/gid` 注解的方法如下所示：

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv1
  annotations:
    pv.beta.kubernetes.io/gid: "1234"
```

当 Pod 使用带有 GID 注解的 PersistentVolume 时，注解的 GID 会被应用于 Pod 中的所有容器， 应用的方法与 Pod 的安全上下文中指定的 GID 相同。 每个 GID，无论是来自 PersistentVolume 注解还是来自 Pod 规约，都会被应用于每个容器中 运行的第一个进程。

# 二、有状态服务创建

## 2.1 部署PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /nfsdata
    server: 10.0.0.10
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv2
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /nfsdata2
    server: 10.0.0.10
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv3
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /nfsdata3
    server: 10.0.0.10
```

## 2.2 statefulset创建并使用PVC

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: harborcloud.com/library/nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "nfs"
      resources:
        requests:
          storage: 1Gi  
```

## 2.3 创建服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

