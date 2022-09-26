---
title: kubernetes调度器scheduler
date: 2021-08-02 19:18:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
---

# 一、简介
Scheduler 是kubernetes 的调度器,主要的任务是把定义的pod分配到集群的节点上。听起来非常简单,但有很多要考虑的问题:

- 公平：如何保证每个节点都能被分配资源
- 资源高效利用：集群所有资源最大化被使用
- 效率：调度的性能要好,能够尽快地对大批量的pod完成调度工作
- 灵活：允许用户根据自己的需求控制调度的逻辑

Sheduler 是作为单独的程序运行的，启动之后会一直坚挺API Server，获取PodSpec.NodeName为空的 pod,对每个pod都会创建一个binding,表明该pod应该放到哪个节点上

# 二、调度过程

调度分为几个部分：首先是过滤掉不满足条件的节点，这个过程称为predicate；然后对通过的节点按照优先级排序,这个是priority;最后从中选择优先级最高的节点。如果中间任何一步骤有错误,就直接返回错误。

Predicate 有一系列的算法可以使用:

Predicate 有一系列的算法可以使用:

- PodFitsResources：节点上剩余的资源是否大于 pod请求的资源
- PodFitsHost：如果pod指定了NodeName,检查节点名称是否和NodeName匹配
- PodFitsHostPorts：节点上已经使用的port 是否和 pod申请的port冲突
- PodSelectorMatches：过滤掉和pod指定的label 不匹配的节点
- NoDiskConflict：已经mount 的volume 和 pod指定的volume 不冲突,除非它们都是只读

如果在predicate过程中没有合适的节点，pod会一直在pending状态，不断重试调度，直到有节点满足条件。
经过这个步骤，如果有多个节点满足条件，就继续priorities过程:按照优先级大小对节点排序

优先级由一系列键值对组成,键是该优先级项的名称,值是它的权重(该项的重要性)。这些优先级选项包括:

- LeastRequestedPriority :通过计算 CPU 和Memory 的使用率来决定权重,使用率越低权重越高。换句话说,这个优先级指标倾向于资源使用比例更低的节点
- BalancedResourceAllocation :节点上CPU 和Memory 使用率越接近,权重越高。这个应该和上面的一起使用,不应该单独使用
- ImageLocalityPriority :倾向于已经有要使用镜像的节点,镜像总大小值越大,权重越高

通过算法对所有的优先级项目和权重进行计算,得出最终的结果。

# 三、自定义调度器

除了kubernetes 自带的调度器,也可以编写自己的调度器。通过spec:schedulername参数指定调度器的名字,可以为pod选择某个调度器进行调度。比如下面的pod选择my-scheduler进行调度,而不是默认的default-scheduler。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotation-second-scheduler
  labels:
    name: multischeduler-example
spec:
  schedulerName: my-schedulen
  containers:
  - name: pod-with-second-annotation-container
    image: gcr.io/google_containers/pause:2.0
```

# 四、节点亲和性

pod.spec.nodeAffinity

- preferredDuringSchedulinglgnoredDuringExecution：软策略
- requiredDuringSchedulinglgnoredDuringExecution：硬策略

requiredDuringSchedulinglgnoredDuringExecution

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity
  labels:
    app: node-affinity-pod
spec:
  containers:
  - name: with-node-affinity
    image: harborcloud.com/library/nginx:1.9.1
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: NotIn
            values:
            - k8s-node02
```

requiredDuringSchedulinglgnoredDuringExecution

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity
  labels:
    app: node-affinity-pod
spec:
  containers:
  - name: with-node-affinity
    image: harborcloud.com/library/nginx:1.9.1
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - k8s-node02222
```

综合

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity
  labels:
    app: node-affinity-pod
spec:
  containers:
  - name: with-node-affinity
    image: harborcloud.com/library/nginx:1.9.1
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: NotIn
            values:
            - k8s-node02
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: source
            operator: In
            values:
            - qikqiak
```

## 4.1 键值运算关系

- In: label的值在某个列表中
- Notln: label的值不在某个列表中
- Gt: label的值大于某个值
- Lt: label的值小于某个值
- Exists:某个label 存在
- DoesNotExist:某 label 不存在

# 五、Pod亲和性

pod.spec.affinity.podAffinity/podAntiAffinity

- preferredDuringSchedulinglgnoredDuringExecution: 软策略
- requiredDuringSchedulinglgnoredDuringExecution:硬策略

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  affinity:
    podAffinity: #亲和力
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            service.cpaas.io/name: deployment-nginx
        topologyKey: kubernetes.io/hostname # pod同node
    podAntiAffinity: #反亲和力
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: a
              operator: In
              values:
              - b
          topologyKey: kubernetes.io/hostname
  containers:
 - name: test-pod
    image: nginx:1.18
```

# 六、亲和性/反亲和性调度策略比较如下:

| 调度策略        | 匹配标签 | 操作符                                        | 拓扑域支持 | 调度目标                                |
| --------------- | -------- | --------------------------------------------- | ---------- | --------------------------------------- |
| nodeAffinity    | 主机     | In, Notln, Exists,DoesNotExist, Gt, Lt <br /> | 否         | 指定主机                                |
| podAffinity     | POD      | In, Notln, Exists,DoesNotExist                | 是         | POD与指定POD同一拓扑域 （同一个node上） |
| podAnitAffinity | POD      | In, Notln, Exists,DoesNotExist                | 是         | POD与指定POD不在同一拓扑域              |

# 七、污点(Taint) 和容忍(Toleration)

节点亲和性，是pod的一种属性(偏好或硬性要求),它使pod被吸引到一类特定的节点。Taint则相反,它使节点能够排斥一类特定的pod。Taint 和toleration 相互配合,可以用来避免 pod 被分配到不合适的节点上。每个节点上都可以应用一个或多个taint,这表示对于那些不能容忍这些taint的pod,是不会被该节点接受的。如果将toleration应用于pod上,则表示这些pod可以(但不要求)被调度到具有匹配taint的节点上

## 7.1 污点(Taint)

1、污点(Taint)的组成
使用kubectl taint命令可以给某个Node节点设置污点, Node被设置上污点之后就和Pod之间存在了一种相斥的关系,可以让Node拒绝Pod的调度执行,甚至将Node已经存在的Pod驱逐出去。

每个污点的组成如下:

```
key=value:effect
```

每个污点有一个key和value作为污点的标签,其中value 可以为空, effect描述污点的作用。

当前taint effect 支持如下三个选项:

- NoSchedule ：表示k8s将不会将Pod调度到具有该污点的Node上
- PreferNoSchedule：表示k8s将尽量避免将Pod调度到具有该污点的Node上
- NoExecute ：表示k8s将不会将Pod调度到具有该污点的Node上,同时会将Node 上已经存在的Pod驱逐出去

1、污点的设置、查看和去除

```
#设置污点
kubectl taint nodes node1 key1=value1: NoSchedule
#节点说明中,查找Taints字段
kubectl describe pod pod - name
#去除污点
kubectl taint nodes node1 key1: NoSchedule-
# master 节点的添加Taint
kubectl taint nodes k8s-master01 node-role.kubernetes.io/master=:NoSchedule
# master去除污点
kubectl taint nodes k8s-master01 node-role.kubernetes.io/master=:NoSchedule-
```

2 例如：

```
[root@k8s-master01 scheduler]# kubectl get pod -o wide
NAME    READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
pod-1   1/1     Running   0          8m28s   10.244.2.205   k8s-node02   <none>           <none>
pod-3   1/1     Running   0          7m27s   10.244.2.207   k8s-node02   <none>           <none>
[root@k8s-master01 scheduler]# kubectl taint node  k8s-node02 checkstatus=k8s:NoExecute
node/k8s-node02 tainted
[root@k8s-master01 scheduler]# kubectl get pod -o wide
No resources found in default namespace.
```

将k8s-node02设置污点NoExecute ，pod将从k8s-node02移除。

## 7.2 容忍(Tolerations)

设置了污点的Node将根据taint 的 effect: NoSchedule, PreferNoSchedule, NoExecute 和Pod 之间产生互斥的关系, Pod将在一定程度上不会被调度到Node上。但我们可以在Pod上设置容忍(Toleration),意思是设置了容忍的Pod将可以容忍污点的存在,可以被调度到存在污点的Node上。

pod.spec.tolerations

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  key: "key2"
  operator: "Exists"
  effect: "NoSchedule"
```

完整yaml：

将在60s过后才会被node移除pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
  labels:
    app: pod-1
spec:
  containers:
  - name: with-node-affinity
    image: harborcloud.com/library/nginx:1.9.1
  tolerations:
  - key: "checkstatus"
    operator: "Equal"
    value: "k8s"
    effect: "NoExecute"
    tolerationSeconds: 60
```

其中key, vaule, effect 要与Node上设置的taint保持一致
operator 的值为Exists 将会忽略value值
tolerationSeconds 用于描述当Pod需要被驱逐时可以在Pod上继续保留运行的时间.。

1 当不指定key值时,表示容忍所有的污点key

```yaml
tolerations:
- operator: "Exists"
```

2 当不指定effect值时,表示容忍所有的污点作用

```yaml
tolerations:
- key: "key"
  operator: "Exists"
```

3 有多个Master存在时,防止资源浪费,可以如下设置

```
kubectl taint nodes Node-Name node-role.kubernetes.io/master=:PreferNoSchedule
```

# 八、指定调度节点

1 Pod.spec.nodeName 将 Pod 直接调度到指定的Node 节点上,会跳过Scheduler的调度策略,该匹配规则是强制匹配

```yaml
apiVersion: extensions/vlbeta1
kind: Deployment
metadata:
  name: myweb
spec:
  replicas: 7
  template:
    metadata:
      labels:
        app: myweb
    spec:
      nodeName: k8s-node01 #将pod调度到k8s-node01
      containers:
      - name: myweb
        image: hub.atguigu.com/library/myapp:v1
        ports:
        - containerPort: 80
```

2 Pod.spec.nodeSelector:通过 kubernetes 的label-selector 机制选择节点,由调度器调度策略匹配label,而后调度Pod到目标节点,该匹配规则属于强制约束

```yaml
apiVersion: extensions/vlbeta1
kind: Deployment
metadata:
  name: myweb
spec:
  replicas: 2
  template:
  metadata:
    labels:
      app: myweb
  spec:
    nodeSelector: # 未来Kubernetes会将nodeSelector废除
      type: backEndNode1 #自定义节点调度器
    containers:
    - name: myweb
      image: harbor/tomcat:8.5-jre8
      ports:
      - containerPort: 80
```

