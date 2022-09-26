---
title: kubernetes  kubectl常用命令
date: 2021-07-25 22:38:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
  - kubectl 
---

# 1 kubectl 概述

kubectl 是Kubernetes 集群的命令行工具，通过kubectl 能够对集群本身进行管理，并能够在集群上进行容器化应用的安装部署。

[kubectl 概述 | Kubernetes](https://kubernetes.io/zh/docs/reference/kubectl/overview/) 

# 2 kubectl 命令的语法

```
kubectl [command] [TYPE] [NAME] [flags]
```

- `command`：指定要对一个或多个资源执行的操作，例如 `create`、`get`、`describe`、`delete`。
- `TYPE`：指定[资源类型](https://kubernetes.io/zh/docs/reference/kubectl/overview/#%E8%B5%84%E6%BA%90%E7%B1%BB%E5%9E%8B)。资源类型不区分大小写， 可以指定单数、复数或缩写形式。 
- `NAME`：指定资源的名称。名称区分大小写。 如果省略名称，则显示所有资源的详细信息 `kubectl get pods`。 
- `flags`: 指定可选的参数。例如，可以使用 `-s` 或 `-server` 参数指定 Kubernetes API 服务器的地址和端口。

# 3 常用命令

[kubectl | Kubernetes](https://kubernetes.io/zh/docs/reference/kubectl/kubectl/) 

[常用命令](http://docs.kubernetes.org.cn/475.html)

## 3.1 基础命令

| create  | 从文件或 stdin 创建一个或多个资源。                          |
| ------- | ------------------------------------------------------------ |
| expose  | 将副本控制器、服务或 pod 作为新的 Kubernetes 服务暴露。      |
| run     | 在集群上运行指定的镜像。                                     |
| get     | 列出一个或多个资源。                                         |
| delete  | 从文件、标准输入或指定标签选择器、名称、资源选择器或资源中删除资源。 |
| set     | 为对象设置功能特性                                           |
| explain | 获取多种资源的文档。例如 pod, node, service 等。             |
| edit    | 使用默认编辑器编辑和更新服务器上一个或多个资源的定义。       |

## 3.2 部署命令

| rollout   | 管理资源的部署。                     |
| --------- | ------------------------------------ |
| scale     | 更新指定副本控制器的大小。           |
| autoscale | 自动伸缩由副本控制器管理的一组 pod。 |

## 3.3 集群管理命令

| certificate  | 修改证书资源                          |
| ------------ | ------------------------------------- |
| cluster-info | 显示集群信息                          |
| top          | 显示资源（CPU/内存/存储）的使用情况。 |
| cordon       | 将节点标记为不可调度。                |
| uncordon     | 将节点标记为可调度。                  |
| drain        | 腾空节点以准备维护。                  |
| taint        | 更新一个或多个节点上的污点。          |

## 3.4 故障诊断命令

| describe     | 显示一个或多个资源的详细状态                          |
| ------------ | ----------------------------------------------------- |
| logs         | 在 pod 中打印容器的日志。                             |
| attach       | 附加到正在运行的容器，查看输出流或与容器（stdin）交互 |
| exec         | 执行命令到容器                                        |
| port-forward | 转发一个或多个本地端口到一个pod                       |
| proxy        | 运行 Kubernetes API 服务器的代理                      |
| cp           | 拷贝文件或目录到容器中                                |
| auth         | 检查授权                                              |

## 3.5 其它命令

| convert      | 在不同的 API 版本之间转换配置文件。配置文件可以是 YAML 或 JSON 格式。 |
| ------------ | ------------------------------------------------------------ |
| replace      | 从文件或标准输入中替换资源                                   |
| apply        | 从文件或 stdin 对资源应用配置更改。                          |
| patch        | 使用策略合并 patch 程序更新资源的一个或多个字段。            |
| label        | 添加或更新一个或多个资源的标签。                             |
| annotate     | 添加或更新一个或多个资源的注解。                             |
| completion   | 为指定的 shell （bash 或 zsh）输出 shell 补齐代码。          |
| api-versions | 列出可用的 API 版本。                                        |
| config       | 修改 kubeconfig 文件。有关详细信息，请参阅各个子命令。       |
| plugin       | 提供用于与插件交互的实用程序。                               |
| version      | 显示运行在客户端和服务器上的 Kubernetes 版本。               |

## 3.6 kubectl create 与kubectl apply区别

| **kubectl create**                                           | **kubectl apply**                                      |
| ------------------------------------------------------------ | ------------------------------------------------------ |
| 它首先删除资源，然后从提供的文件中创建资源                   | 只更新文件中给出的属性                                 |
| 在create中使用的文件应该是完整的                             | apply中使用的文件可能是一个不完整的规范                |
| Create工作于资源的每个属性                                   | Apply仅对资源的某些属性有效                            |
| 如果您将使用同一个文件的替换命令，该命令将失败，因为缺少信息 | 您可以应用只更改注释的文件，而不指定资源的任何其他属性 |

# 4 常用命令使用案例

```
//获取所有可用的api 版本
kubectl api-versions
//给Node加上标签
kubectl label nodes <your-node-name> disktype=ssd
```

快速创建pod及查看运行状态

```shell
// 创建deployment 
kubectl create deployment nginx --image=nginx
// 将资源暴露为新的Kubernetes Service
kubectl expose deployment nginx --port=80 --type=NodePort
//查看pod
kubectl get pod/po <Pod_name>
kubectl get pod/po <Pod_name> -o wide
//显示所有pod
kubectl get pods
//删除pod
kubectl delete -f pod pod_name.yaml
kubectl delete pod --all/[pod_name]
```

通过yaml创建deployment 

```
kubectl apply -f nginx-create.yaml
```

获取所有的node

```
kubectl get nodes
```

获取指定node信息

```
kubectl get nodes node_name
```

帮助命令

```
kubectl --help
//查看某个操作
kubectl get -help
```

快速获取yaml

```
kubectl create deployment nginx --image=nginx -o yaml --dry-run > nginx-create.yaml
# 或者
kubectl get deploy nginx -o yaml ---export > nginx-test.yaml
```

# 5 练习操作

## 5.1 删除操作

**删除deplyment**

```
//删除myapp
kubectl delete deployment myapp
//删除所有
kubectl delete deployment --all
```

**删除pod**

```
//删除myapp
kubectl delete pod myapp
//删除所有
kubectl delete pod --all
```

**删除svc**

```
//删除nginx svc
kubectl delete svc nginx
```

## 5.2 通过yaml创建资源

create和apply的区别在**3.2 常用命令**有说明

```
kubectl create -f  init.yaml
或
kubectl apply -f  init.yaml
```

## 5.3 查看k8s可用的apiVersion

```
kubectl api-versions
```

## 5.4 查看所有api资源

通过配置清单创建资源时，会出现error: unable to recognize "init.yaml": no matches for kind "Pod" in version "V1"，说明对应apiVersion 版本没有指定资源的定义，可以通过如下命令来查看对应资源的版本。

```
kubectl api-resources
```

## 5.5 查看资源

查看pod资源

```yaml
kubectl get pod
```

## 5.6 分析资源

查看pod的描述

```
kubectl describe pod myapp-pod
```

## 5.7 资源日志查看

查看pod的指定容器的日志

```
kubectl logs myapp-pod -c  init-myservice
```

## 5.8 编辑资源清单

```
kubectl edit pod myapp-pod
```

## 5.9 获取资源的api版本

```
kubectl explain pod
```

## 5.10 获取已创建pod的yaml

```
kubectl get pod lifecycle-demo -o yaml
```













