---
title: kubernates 之 configMap
date: 2021-08-23 21:14:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - k8s资源
---

# 一、介绍

ConfigMap功能在Kubernetes1.2 版本中引入，许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。ConfigMap API给我们提供了向容器中注入配置信息的机制， ConfigMap可以被用来保存单个属性，也可以用来保存整个配置文件或者JSON二进制大对象

# 二、ConfigMap 的创建

## 1、使用目录创建

```
[root@k8s-master01 k8s-yaml]# mkdir -p configmap/folder
[root@k8s-master01 k8s-yaml]# cd configmap/folder
[root@k8s-master01 folder]# vi game.properties
enemies=aliens
1ives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
[root@k8s-master01 folder]# vi ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
[root@k8s-master01 folder]# kubectl create configmap game-config --from-file=../folder
configmap/game-config created
```

--from-file指定在目录下的所有文件都会被用在ConfigMap里面创建一个键值对，键的名字就是文件名，值就是文件的内容

配置查看：

```
[root@k8s-master01 folder]# kubectl get cm game-config  -o yaml
apiVersion: v1
data:
  game.properties: |
    enemies=aliens
    1ives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret. code. passphrase=UUDDLRLRBABAS
    secret， code. allowed=true
    secret， code. lives=30
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  creationTimestamp: "2022-07-18T23:12:14Z"
  name: game-config
  namespace: default
  resourceVersion: "876568"
  uid: f941e011-cb4d-47bd-9bb1-eecbddfed632
```

## 2、使用文件创建
只要指定为一个文件就可以从单个文件中创建ConfigMap

```
[root@k8s-master01 folder]# kubectl create configmap game-config2 --from-file=../folder/game.properties 
configmap/game-config2 created
[root@k8s-master01 folder]# kubectl get configmaps game-config2 -o yaml 
apiVersion: v1
data:
  game.properties: |
    enemies=aliens
    1ives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
kind: ConfigMap
metadata:
  creationTimestamp: "2022-07-18T23:19:12Z"
  name: game-config2
  namespace: default
  resourceVersion: "877263"
  uid: b94f19cd-436d-4253-8320-3af94396847e
```

--from-file 这个参数可以使用多次，你可以使用两次分别指定上个实例中的那两个配置文件，效果就跟指定整个目录是一样的

## 3、使用字面值创建

使用文字值创建，利用--from-literal参数传递配置信息，该参数可以使用多次，格式如下：

```
[root@k8s-master01 folder]# kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
configmap/special-config created
[root@k8s-master01 folder]# kubectl get configmaps special-config -o yaml
apiVersion: v1
data:
  special.how: very
  special.type: charm
kind: ConfigMap
metadata:
  creationTimestamp: "2022-07-18T23:20:24Z"
  name: special-config
  namespace: default
  resourceVersion: "877388"
  uid: d8577f2c-155c-417b-ad42-cf5e8ef24546
```

# 三、Pod 中使用ConfigMap

1、使用ConfigMap来替代环境变量

special-config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

env-config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

dapi-test-pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
  - name: test-container
    image: harborcloud.com/library/nginx:1.9.1
    command: ["/bin/sh"，"-c"，"env"]
    env:
    - name: SPECIAL_LEVEL_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.how
    - name: SPECIAL_TYPE_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.type
    envFrom:
    - configMapRef:
        name: env-config
  restartPolicy: Never
```

2.用ConfigMap设置命令行参数

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
  - name: test-container
    image: harborcloud.com/library/nginx:1.9.1
    command: ["/bin/sh"，"-c"，"echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)"]
    env:
    - name: SPECIAL_LEVEL_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.how
    - name: SPECIAL_TYPE_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.type
  restartPolicy: Never
```

3、通过数据卷插件使用ConfigMap

在数据卷里面使用这个ConfigMap，有不同的选项。最基本的就是将文件填入数据卷，在这个文件中，键就是文件名，键值就是文件内容

```yaml
apiversion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
  - name: test-container
    image: harborcloud.com/library/myapp:v1.23
    command: ["/bin/sh"，"-c"，"sleep 500s"]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: special-config
  restartPolicy: Never

```

# 四、ConfigMap 的热更新

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-config
  namespace: default
data:
  log_level: INFO
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: harborcloud.com/library/myapp:v1.23
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: log-config
```

```
$ kubectl exec `kubectl get pods -l run=my-nginx -o=name|cut -d "/" -f2` cat /etc/config/log_level
INFO
```

修改ConfigMap

```
$ kubectl edit configmap log-config
```

修改log_level的值为DEBUG等待大概10秒钟时间，再次查看环境变量的值

```
[root@k8s-master01 env]# kubectl exec `kubectl get pods -l run=my-nginx -o=name|cut -d "/" -f2` cat /etc/config/log_level
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
DEBUG
```

# 五、ConfigMap 更新后滚动更新 Pod

更新ConfigMap目前并不会触发相关Pod的滚动更新，可以通过修改 pod annotations的方式强制触发滚动更新

```
$ kubectl patch deployment my-nginx --patch '{"spec":{"template":{"metadata":{"annotations":{"version/config":"20190411"}}}}}'
```

这个例子里我们在.spec.template.metadata.annotations中添加 version/config，每次通过修改version/config来触发动更新

!!! 更新ConfigMap 后：

- 使用该ConfigMap挂载的Env不会同步更新
- 使用该ConfigMap挂载的Volume中的数据需要一段时间(实测大概10秒)才能同步更新