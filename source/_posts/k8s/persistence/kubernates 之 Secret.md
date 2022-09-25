---
title: kubernates 之 Secret
date: 2021-08-24 20:14:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - k8s资源
---

# 一、Secret 存在意义
Secret 解决了密码、token、密钥等敏感数据的配置问题,而不需要把这些敏感数据暴露到镜像或者Pod Spec中。Secret 可以以Volume或者环境变量的方式使用

# 二、Secret有三种类型

Service Account

用来访 Kubernetes API, 由Kubernetes 自动创建,并且会自动挂载到Pod的/run/secrets/kubernetes.io/serviceaccount 目录中

Opaque 

base64编码格式的Secret,用来存储密码、密钥等

kubernetes.io/dockerconfigjson

用来存储私有 docker registry 的认证信息

## 2.1 Service Account

Service Account 用来访 Kubernetes API, 由Kubernetes 自动创建,并且会自动挂载到Pod的/run/secrets/kubernetes.io/serviceaccount 目录中

```
[root@k8s-master01 secret]# kubectl run nginx --image harborcloud.com/library/nginx:1.9.1
pod/nginx created
[root@k8s-master01 secret]# kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          12s
[root@k8s-master01 secret]# kubectl exec nginx ls /run/secrets/kubernetes.io/serviceaccount  
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
ca.crt
namespace
token
```

## 2.2 Opaque Secret

### 1、创建说明

Opaque 类型的数据是一个map类型,要求value是base64编码格式:

```
[root@k8s-master01 secret]# echo -n "admin"|base64 #账号
YWRtaW4=
[root@k8s-master01 secret]# echo -n "admin"|base64 #密码
YWRtaW4=
```

secrets.yml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: YWRtaW4=
  username: YWRtaW4=
```

### 2、使用方式
2.1、将Secret挂载到Volume中

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: seret-test
  name: seret-test
spec:
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
  containers:
  - image: harborcloud.com/library/myapp:v1.23
    name: db
    volumeMounts:
    - name: secrets
      mountPath: '/etc/secret'
      readOnly: true
```

2.2、将Secret导出到环境变量中

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pod-deployment
  template:
    metadata:
      labels:
        app: pod-deployment
    spec:
      containers:
      - name: pod-1
        image: harborcloud.com/library/myapp:v1.23
        ports:
        - containerPort: 80
        env:
        - name: TEST_USER
          valueFrom:
            secretKeyRef:
              name: mysecret 
              key: username
        - name: TEST_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
```

## 2.3 kubernetes.io/dockerconfigjson

使用Kuberctl 创建 docker registry 认证的secret

```
$ kubectl create secret docker-registry myregistrykey --docker-server=DOCKER_REGISTRY_SERVER -docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

例子：

```
[root@k8s-master01 secret]# kubectl create secret docker-registry myregistrykey --docker-server=harborcloud.com --docker-username=admin --docker-password=Harbor12345 --docker-email=leelun@sina.cn
secret/myregistrykey created
```

### 1 公共镜像的pod

未设置认证secret创建pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
  - name: foo
    image: harborcloud.com/newland/myapp:1.9.1
```

未设置认证secret的结果

```
[root@k8s-master01 secret]# kubectl get pods
NAME                             READY   STATUS         RESTARTS   AGE
foo                              0/1     ErrImagePull   0          4s
[root@k8s-master01 secret]# kubectl describe pod foo
Events:
  Type     Reason     Age                            From               Message
  ----     ------     ----                           ----               -------
  Normal   Scheduled  26s                            default-scheduler  Successfully assigned default/foo to k8s-node02
  Normal   BackOff    <invalid>                      kubelet            Back-off pulling image "harborcloud.com/newland/myapp:1.9.1"
  Warning  Failed     <invalid>                      kubelet            Error: ImagePullBackOff
  Normal   Pulling    <invalid> (x2 over <invalid>)  kubelet            Pulling image "harborcloud.com/newland/myapp:1.9.1"
  Warning  Failed     <invalid> (x2 over <invalid>)  kubelet            Failed to pull image "harborcloud.com/newland/myapp:1.9.1": rpc error: code = Unknown desc = Error response from daemon: unauthorized: unauthorized to access repository: newland/myapp, action: pull: unauthorized to access repository: newland/myapp, action: pull
  Warning  Failed     <invalid> (x2 over <invalid>)  kubelet            Error: ErrImagePull
```

### 2 私有镜像 

在创建Pod 的时候,通过imagePullSecrets来引用刚创建的'myregistrykey)

```
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
  - name: foo
    image: harborcloud.com/newland/myapp:1.9.1
  imagePullSecrets:
  - name: myregistrykey
```

创建并查询结果

```
[root@k8s-master01 secret]# kubectl create -f pod-secret-image.yaml 
pod/foo created
[root@k8s-master01 secret]# kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
foo                              1/1     Running   0          4s
[root@k8s-master01 secret]# kubectl describe pod foo
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  43s        default-scheduler  Successfully assigned default/foo to k8s-node02
  Normal  Pulling    <invalid>  kubelet            Pulling image "harborcloud.com/newland/myapp:1.9.1"
  Normal  Pulled     <invalid>  kubelet            Successfully pulled image "harborcloud.com/newland/myapp:1.9.1" in 1.289816846s
  Normal  Created    <invalid>  kubelet            Created container foo
  Normal  Started    <invalid>  kubelet            Started container foo
```

