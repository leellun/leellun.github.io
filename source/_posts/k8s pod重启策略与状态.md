---
title: k8s pod重启策略与状态
date: 2021-08-10 21:32:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - 重启策略
---

# 重启策略

PodSpec中有一个 restartPolicy字段，可能的值为Always、OnFailure和Never。默认为Always. 

restartPolicy 适用于Pod中的所有容器。restartPolicy仅指通过同一节点上的kubelet 重新启动容器。失败的容器由kubelet以五分钟为上限的指数退避延迟(10秒,20秒, 40秒.)重新启动,并在成功执行十分钟后重置。如Pod文档中所述,一旦绑定到一个节点, Pod将永远不会重新绑定到另一个节点。

# 状态

Pod的status字段是一个PodStatus对象, PodStatus中有一个 phase字段。

Pod的相位(phase)是Pod在其生命周期中的简单宏观概述。该阶段并不是对容器或Pod的综合汇总,也不是为了做为综合状态机

挂起(Pending)：Pod已被Kubernetes系统接受,但有一个或者多个容器镜像尚未创建。等待时间包括调度Pod的时间和通过网络下载镜像的时间，这可能需要花点时间

运行中(Running)：该Pod已经绑定到了一个节点上, Pod中所有的容器都已被创建。至少有一个容器正在运行,或者正处于启动或重启状态

成功(Succeeded)： Pod中的所有容器都被成功终止,并且不会再重启

失败(Failed)：Pod中的所有容器都已终止了,并且至少有一个容器是因为失败终止。也就是说,容器以非0状态退出或者被系统终止

未知(Unknown)：因为某些原因无法取得Pod的状态,通常是因为与Pod所在主机通信失败

# 状态示例

Pod中只有一个容器并且正在运行，容器成功退出

- 记录事件完成
- 如果restartPolicy为:
  	Always：重启容器；Pod phase 仍为Running
  	OnFailure： Pod phase变成 Succeeded
  	Never：Pod phase成 Succeeded

Pod中只有一个容器并且正在运行。容器退出失败

- 记录失败事件
- 如果restartPolicy为:
  	Always：重启容器: Pod phase 仍为Running
  	OnFailure：重启容器; Pod phase仍为Running
  	Never：Pod phase 变成 Failed

Pod中有两个容器并且正在运行。容器1退出失败

- 记录失败事件
- 如果restartPolicy为:
  	Always：重启容器; Pod phase仍为Running
  	OnFailure：重启容器: Pod phase仍为Running
  	Never： 重启容器: Pod phase为Running
- 如果有容器1没有处于运行状态,并且容器2退出:
          记录失败事件
          如果restartPolicy为:
                  Always: 重启容器: Pod phase为 Running
                 OnFailure: 重启容器; Pod phase仍为Running
                 Never: Pod phase 变成 Failed

Pod 中只有一个容器并处于运行状态。容器运行时内存超出限制

- 容器以失败状态终止
- 记录OOM事件
- 如果restartPolicy为:
  Always: 重启容器: Pod phase为Running
  OnFailure:重启容器; Pod phase 为Running
  Never: 记录失败事件: Pod phase仍为Failed

Pod正在运行,磁盘故障.

- 杀掉所有容器。记录适当事件
- Pod phase变成 Failed
- 如果使用控制器来运行, Pod将在别处重建

Pod正在运行,其节点被分段

- 节点控制器等待直到超时
- 节点控制器将Pod phase设置为Failed
- 如果是用控制器来运行, Pod将在别处重建

# 启动退出

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-containen
    image: harborcloud.com/library/myapp:v1.23   #本地搭建仓库  
    lifecycle:
      postStart: #pod创建后执行
        exec:
          command: ["/bin/sh","-c","touch /tmp/live"]
      preStop: #pod退出前执行
        exec:
          command: ["/bin/sh","-c","rm -rf /tmp/live"]
```

查看是否成功：

```
[root@k8s-master01 probe]# kubectl exec lifecycle-demo -it -- /bin/sh
# ls /tmp
live
```



