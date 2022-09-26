---
title: k8s资源清单及常用字段
date: 2021-07-27 19:38:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
---

K8s中所有的内容都抽象为资源,资源实例化之后,叫做对象。

# 名称空间级别

工作负载型资源(workload ): Pod、 ReplicaSet, Deployment、 StatefulSet、DaemonSet、Job、CronJob (ReplicationController 在vl.11 版本被废弃)

服务发现及负载均衡型资源( ServiceDiscovery LoadBalance )： Service, Ingress. ...

配置与存储型资源： Volume(存储卷)、CSI(容器存储接口,可以扩展各种各样的第三方存储卷)

特殊类型的存储卷：ConfigMap(当配置中心来使用的资源类型)、Secret(保存敏感数据)、DownwardAPI (把外部环境中的信息输出给容器)

集群级资源: Namespace、 Node、 Role、 ClusterRole, RoleBinding、 ClusterRoleBinding

元数据型资源: HPA、 PodTemplate、 LimitRange

# 资源清单常用字段

k8s 集群中对资源管理和资源对象编排部署都可以通过声明样式（YAML）文件来解决，通过kubectl 命令直接使用资源清单文件就可以实现对大量的资源对象进行编排部署了。这样的yaml文件我们一般称为资源清单。

## 必须存在的属性

| 参数名                 | 字段类型 | 说明                                           |
| ---------------------- | -------- | ---------------------------------------------- |
| version                | string   | K8S API的版本，可以通过kubectl api-version查询 |
| kind                   | string   | 资源类型和角色                                 |
| metadata               | object   | 元数据对象                                     |
| metadata.name          | string   | 元数据对象的名称，比如pod的名字                |
| metadata.namespace     | string   | 元数据对象的命名控件                           |
| spec                   | object   | 详细定义对象                                   |
| spec.container[]       | list     | 容器列表                                       |
| spec.container[].name  | string   | 定义容器名称                                   |
| spec.container[].image | string   | 定义用到镜像名称                               |

## spec 主要对象

| 参数名                                                       | 字段类型 | 说明                                                         |
| ------------------------------------------------------------ | -------- | ------------------------------------------------------------ |
| version                                                      | string   | K8S API的版本，可以通过kubectl api-version查询               |
| kind                                                         | string   | 资源类型和角色                                               |
| metadata                                                     | object   | 元数据对象                                                   |
| metadata.name                                                | string   | 元数据对象的名称，比如pod的名字                              |
| metadata.namespace                                           | string   | 元数据对象的命名控件                                         |
| metadata.labels                                              | list     | 自定义标签列表                                               |
| metadata.annotations                                         | list     | 自定义注解列表                                               |
| spec                                                         | object   | 详细定义对象                                                 |
| spec.container[]                                             | list     | 容器列表                                                     |
| spec.container[].name                                        | string   | 定义容器名称                                                 |
| spec.container[].image                                       | string   | 定义用到镜像名称                                             |
| spec.container[].imagePullPolicy                             | string   | 镜像拉取策略,    Always：不管镜像是否存在都会进行一次拉取。  Never：不管镜像是否存在都不会进行拉取  IfNotPresent：只有镜像不存在时，才会进行镜像拉取。 |
| spec.container[].command[]                                   | list     | 指定容器启动命令                                             |
| spec.container[].args[]                                      | list     | 命令参数                                                     |
| spec.container[].workingDir                                  | string   | 指定容器的工作目录                                           |
| spec.container[].volumeMounts[]                              | list     | 指定容器内部的存储卷配置                                     |
| spec.container[].volumeMounts[].name                         | string   | 指定挂载存储卷的名称                                         |
| spec.container[].volumeMounts[].mountPath                    | string   | 指定挂载存储卷的路径                                         |
| spec.container[].volumeMounts[].readyOnly                    | string   | true或false，读写模式                                        |
| spec.container[].ports[]                                     | list     | 指定容器需要用到的端口列表                                   |
| spec.container[].ports[].name                                | string   | 端口名称                                                     |
| spec.container[].ports[].containerPort                       | string   | 指定容器要监听的端口号                                       |
| spec.container[].ports[].hostPort                            | string   | 指定容器所在主机需要监听的端口号，默认跟containerPort相同，如果设置了hostPort同一台主机无法启动该容器的相同副本(因为主机的端口号不能相同，这样会冲突) |
| spec.container[].ports[].protocol                            | string   | 指定端口协议，支持TCP和UDP，默认值TCP                        |
| spec.container[].env[]                                       | list     | 指定容器运行需要的环境                                       |
| spec.container[].env[].name                                  | string   | 环境变量名称                                                 |
| spec.container[].env[].value                                 | string   | 环境变量值                                                   |
| spec.container[].resources                                   | object   | 指定资源限制和资源请求的值                                   |
| spec.container[].resources.limits                            | object   | 指定设置容器运行时资源的运行上限                             |
| spec.container[].resources.limits.cpu                        | string   | 指定cpu的限制，单位core数，将用于docker run --cpu-shares 参数 |
| spec.container[].resources.limits.memory                     | string   | 指定MEM内存的限制，单位MIB ,GIB                              |
| spec.container[].resources.requests                          | object   | 指定容器启动和调度的限制设置                                 |
| spec.container[].resources.requests.cpu                      | string   | cpu请求，单位core数，容器启动时初始化可用数量                |
| spec.container[].resources.requests.memory                   | string   | 内存请求，单位MIB,GIB 容器启动的初始化可用数量               |
| spec.container[].livenessProbe                               | object   | 对Pod内各容器健康检查的设置，当探测无响应几次之后，系统将自动重启该容器。可以设置的方法包括：exec、httpGet和tcpSocket。对一个容器仅需设置一种健康检查方法 |
| spec.container[].livenessProbe.exec                          | object   | 对Pod内各容器健康检查的设置                                  |
| spec.container[].livenessProbe.exec.command[]                | list     | exec方式需要指定的命令或者脚本                               |
| spec.container[].livenessProbe.httpGet                       | object   | 对Pod内各容器健康检查的设置，httpget方式。需指定path、port   |
| spec.container[].livenessProbe.httpGet.path                  | string   |                                                              |
| spec.container[].livenessProbe.httpGet.port                  | string   |                                                              |
| spec.container[].livenessProbe.httpGet.host                  | string   |                                                              |
| spec.container[].livenessProbe.httpGet.scheme                | string   |                                                              |
| spec.container[].livenessProbe.httpGet .httpHeaders[]        | list     |                                                              |
| spec.container[].livenessProbe.httpGet .httpHeaders[] .name  | string   |                                                              |
| spec.container[].livenessProbe.httpGet .httpHeaders[] .value | string   |                                                              |
| spec.container[].livenessProbe.tcpSocket                     | object   |                                                              |
| spec.container[].livenessProbe.tcpSocket.port                | string   |                                                              |
| spec.container[].livenessProbe.initialDelaySeconds           | string   | 容器启动完成后首次探测的时间，单位为s                        |
| spec.container[].livenessProbe.timeoutSeconds                | string   | 探测等待响应的超时时间，单位为s,默认1s                       |
| spec.container[].livenessProbe.periodSeconds                 | string   | 定期探测时间设置，单位s,默认10s探测一次                      |
| spec.container[].livenessProbe.successThreshold              | string   | 失败后检查成功的最小连续成功次数。默认为1.活跃度必须为1。最小值为1。 |
| spec.container[].livenessProbe.failureThreshold              | string   | 当Pod成功启动且检查失败时，Kubernetes将在放弃之前尝试failureThreshold次。放弃生存检查意味着重新启动Pod。而放弃就绪检查，Pod将被标记为未就绪。默认为3.最小值为1。 |
| spec.container[].livenessProbe.securityContext               | object   |                                                              |
| spec.container[].livenessProbe.securityContext .privileged   | string   |                                                              |
| spec.restartPolicy                                           | string   | 定义pod重启策略，默认值Always                   Always：Pod一旦终止运行，则无论容器是如何终止的kubelet服务都将重启它                    OnFailure：只有Pod以非零退出码终止时，kubelet才会重启该容器，如果容器正常结束（退出码为0），则kubelet将不会重启它      Never：Pod终止后，kubelet将退出码报告给Master，不会重启该Pod |
| spec.nodeSelector                                            | object   | 定义node的label过滤标签，以key：value格式指定                |
| spec.imagePullSecrets                                        | Object   | 定义pull镜像secret名称，以name:secretKey格式指定             |
| spec.hostNetwork                                             | Boolean  | 定义是否使用主机网络模式，默认值false，设置true表示使用宿主网路，不适用docker网桥，同时设置true将无法在同一台宿主机上启动第二个副本。 |
| spec.volumes[]                                               | list     | 在该pod上定义的共享存储卷列表                                |
| spec.volumes[].name                                          | string   | 共享存储卷的名称，在一个pod中每个存储卷定义一个名称，容器定义部分的containers[].volumeMounts[].name将引用该共享存储卷的名称。可以定义多个volume，每个volume的name保持唯一。 |
| spec.volumes[].emptyDir                                      | string   | 类型为emptyDir的存储卷，表示与Pod同生命周期的一个临时目录，其值为一个空对象：emptyDir:{} |
| spec.volumes[].hostPath                                      | object   | 类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录，通过volume[].hostPath.path指定 |
| spec.volumes[].hostPath.path                                 | string   | Pod所在主机的目录，将被用于容器中mount的目录                 |
| spec.volumes[].secret                                        | object   | 类型为secret的存储卷，表示挂载集群预定义的secret对象到容器内部 |
| spec.volumes[].secret.secretName                             | string   |                                                              |
| spec.volumes[].secret.items[]                                | list     |                                                              |
| spec.volumes[].secret.items[].key                            | string   |                                                              |
| spec.volumes[].secret.items[].path                           | string   |                                                              |
| spec.volumes[].configMap                                     | object   | 类型为configMap的存储卷，表示挂载集群预定义的configMap对象到容器内部 |
| spec.volumes[].configMap.name                                | string   |                                                              |
| spec.volumes[].configMap.items[]                             | list     |                                                              |
| spec.volumes[].configMap.items[].key                         | string   |                                                              |
| spec.volumes[].configMap.items[].path                        | string   |                                                              |

# 资源清单格式

```
apiVersion: group/apiversion #如果没有给定group名称,那么默认为core,可以使用kubectl api-versions #获取当前k8s版本上所有的 apiVersion版本信息(每个版本可能不同)
kind:#资源类别
metadata:#资源元数据
  name
  namespace
  1ables
  annotations #主要目的是方便用户阅读查找
spec: #期望的状态(disired state)
status: #当前状态,本字段有Kubernetes 自身维护,用户不能去定义

```

