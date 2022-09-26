---
title: k8s pod生命周期—探针
date: 2021-08-08 22:32:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
---

# 三种类型处理程序

探针是由kubelet对容器执行的定期诊断。要执行诊断, kubelet调用由容器实现的Handler。有三种类型的处理程序:

ExecAction：在容器内执行指定命令。如果命令退出时返回码为0则认为诊断成功。

TCPSocketAction：对指定端口上的容器的IP地址进行TCP检查。如果端口打开,则诊断被认为是成功的。

HTTPGetAction：对指定的端口和路径上的容器的IP地址执行HTTP Get请求。如果响应的状态码大于等于200且小于400,则诊断被认为是成功的

每次探测都将获得以下三种结果之一：

成功：容器通过了诊断。

失败：容器未通过诊断。

未知：诊断失败,因此不会采取任何行动

# 探测方式

**livenessProbe**

指示容器是否正在运行。如果存活探测失败,则kubelet会杀死容器,并且容器将受到其重启策略的影响。如果容器不提供存活探针,则默认状态为Success

**readinessProbe**

指示容器是否准备好服务请求。如果就绪探测失败,端点控制器将从与Pod匹配的所有Service的端点中删除该Pod的IP地址。初始延迟之前的就绪状态默认为Failure。如果容器不提供就绪探针,则默认状态为Success

# Pod hook 

Pod hook (子)是由Kubernetes 管理的kubelet发起的,当容器中的进程启动前或者容器中的进程终止之前运行,这是包含在容器的生命周期之中。可以同时为Pod中的所有容器都配置hook

Hook 的类型包括两种:

- exec:执行一段命令
- HTTP:发送HTTP请求

# 探针示例

注意：harborcloud.com是我本地搭建的云仓库

```
harborcloud.com/library/myapp:v1.23 =>nginx
harborcloud.com/library/busybox:v1.35 =>busybox
```

## 就绪探针

### 资源清单

readinessProbe-httpget

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-httpget-pod
  namespace: default
spec:
  containers:
  - name: readiness-httpget-container
    image: harborcloud.com/library/myapp:v1.23
    imagePullPolicy: IfNotPresent
    readinessProbe:
      httpGet:
        port: 80
        path: /index1.html
      initialDelaySeconds: 1
      periodSeconds: 3
```

### 清单应用

```
[root@k8s-master01 probe]# vi readinessProbe-httpget.ymal
[root@k8s-master01 probe]# kubectl create -f readinessProbe-httpget.ymal
# kubectl get pod 可能需要等待一会才会 status：running
[root@k8s-master01 probe]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
readiness-httpget-pod   0/1     Running   0          2m3s
```

结果分析：为什么会出现READY 0/1

```
[root@k8s-master01 probe]# kubectl describe pod readiness-httpget-pod
Name:         readiness-httpget-pod
Namespace:    default
Priority:     0
Node:         k8s-node02/fd56:a9ae:cb0f::853
Start Time:   Sun, 17 Jul 2022 00:10:36 +0800
Labels:       <none>
Annotations:  <none>
Status:       Running
IP:           10.244.2.5
IPs:
  IP:  10.244.2.5
Containers:
  readiness-httpget-container:
    Container ID:   docker://692f676aa6a3b9a16eac5373d78398df1780d6e7b87e129a3035a871e3617d61
    Image:          wangyanglinux/myapp:v1
    Image ID:       docker-pullable://wangyanglinux/myapp@sha256:9c3dc30b5219788b2b8a4b065f548b922a34479577befb54b03330999d30d513
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 17 Jul 2022 00:12:13 +0800
    Ready:          False
    Restart Count:  0
    Readiness:      http-get http://:80/index1.html delay=1s timeout=1s period=3s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2n7st (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  kube-api-access-2n7st:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                             From               Message
  ----     ------     ----                            ----               -------
  Normal   Scheduled  2m17s                           default-scheduler  Successfully assigned default/readiness-httpget-pod to k8s-node02
  Warning  Failed     <invalid>                       kubelet            Failed to pull image "wangyanglinux/myapp:v1": rpc error: code = Unknown desc = Get https://registry-1.docker.io/v2/: net/http: TLS handshake timeout
  Warning  Failed     <invalid>                       kubelet            Error: ErrImagePull
  Normal   BackOff    <invalid>                       kubelet            Back-off pulling image "wangyanglinux/myapp:v1"
  Warning  Failed     <invalid>                       kubelet            Error: ImagePullBackOff
  Normal   Pulling    <invalid> (x2 over 45s)         kubelet            Pulling image "wangyanglinux/myapp:v1"
  Normal   Pulled     <invalid>                       kubelet            Successfully pulled image "wangyanglinux/myapp:v1" in 16.208867618s
  Normal   Created    <invalid>                       kubelet            Created container readiness-httpget-container
  Normal   Started    <invalid>                       kubelet            Started container readiness-httpget-container
  Warning  Unhealthy  <invalid> (x15 over <invalid>)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 404
```

通过上面分析可以看出 

```
Readiness:      http-get http://:80/index1.html delay=1s timeout=1s period=3s #success=1 #failure=3
error: code = Unknown desc = Get https://registry-1.docker.io/v2/: net/http: TLS handshake timeout
statuscode: 404
```

处理异常

```
[root@k8s-master01 probe]# kubectl exec readiness-httpget-pod -it -- /bin/sh
# cd /usrshare/nginx/html
# echo "234srwerwe">>index1.html
[root@k8s-master01 probe]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
readiness-httpget-pod   1/1     Running   0          6m22s
```

当index1.html添加后pod正常启动

## 存活检测

### livenessProbe-exec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec-pod
  namespace: default
spec:
  containers:
  - name: liveness-exec-containen
    image: harborcloud.com/library/busybox:v1.35
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh","-c","touch /tmp/live;sleep 60;rm -rf /tmp/live;sleep 3600"]
    livenessProbe:
      exec:
        command: ["test","-e","/tmp/live"]
      initialDelaySeconds: 1
      periodSeconds: 3
```

操作：

```
[root@k8s-master01 probe]# kubectl create -f livenessProbe-exec.yaml 
pod/liveness-exec-pod created
[root@k8s-master01 probe]# kubectl get pod
NAME                READY   STATUS    RESTARTS   AGE
liveness-exec-pod   1/1     Running   0          14s
[root@k8s-master01 probe]# kubectl get pod
NAME                READY   STATUS    RESTARTS            AGE
liveness-exec-pod   1/1     Running   1 (<invalid> ago)   2m11s
```

时间轴：

创建pod成功——等待60秒后删除/tmp/live——存活检测/tmp/live被删除了，然后就重启pod

### livenessProbe-httpget

````yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-httpget-pod
  namespace: default
spec:
  containers:
  - name: liveness-httpget-containen
    image: harborcloud.com/library/myapp:v1.23
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80
    livenessProbe:
      httpGet:
        port: http
        path: /index.html
      initialDelaySeconds: 1
      periodSeconds: 3
      timeoutSeconds: 10
````

### livenessProbe-tcp

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-tcp
spec:
  containers:
  - name: nginx
    image: harborcloud.com/library/myapp:v1.23
    livenessProbe:
      initialDelaySeconds: 5
      timeoutSeconds: 1
      tcpSocket:
        port: 80
```

