---
title: k8s私有环境中通过OpenELB 暴露服务端口并结合ingress配置路由规则访问
date: 2022-09-25 10:14:02
categories:
  - 服务器
tags:
  - kubernetes
  - k8s
  - openeelb
  - ingress
---

# 一、节点信息

| 节点         | ip            |
| ------------ | ------------- |
| k8s-master01 | 192.168.66.10 |

说明：当前搭建的微服务项目服务太多，资源有限就只搭建了一个节点

# 二、在 kube-proxy 设置strictARP 

```
kubectl edit configmap kube-proxy -n kube-system
```

修改：

`data.config.conf.ipvs.strictARP` to `true`. 

```
ipvs:
  strictARP: true
```

重启 kube-proxy:

```
kubectl rollout restart daemonset kube-proxy -n kube-system
```

# 三、对存在多个网卡的节点添加一个 annotation 来指定网卡

如果只有一个网卡可以跳过此步骤。

```
kubectl annotate nodes k8s-master01 layer2.openelb.kubesphere.io/v1alpha1="192.168.66.10"
```

# 四、创建Eip 对象

```
# vi layer2-eip.yaml
apiVersion: network.kubesphere.io/v1alpha2
kind: Eip
metadata:
  name: layer2-eip
spec:
  address: 192.168.66.91-192.168.66.100
  interface: eno16777736
  protocol: layer2
```

注意：

（1）address指定的ip域为未使用的同一网段下的ip范围

（2）interface为本地网卡名

```
kubectl apply -f layer2-eip.yaml
```

# 五、搭建测试服务

layer2-openelb.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: layer2-openelb
spec:
  replicas: 2
  selector:
    matchLabels:
      app: layer2-openelb
  template:
    metadata:
      labels:
        app: layer2-openelb
    spec:
      containers:
        - image: luksa/kubia
          name: kubia
          ports:
            - containerPort: 8080
```

layer2-svc.yaml

```
kind: Service
apiVersion: v1
metadata:
  name: layer2-svc
  annotations:
    lb.kubesphere.io/v1alpha1: openelb
    protocol.openelb.kubesphere.io/v1alpha1: layer2
    eip.openelb.kubesphere.io/v1alpha2: layer2-eip
spec:
  selector:
    app: layer2-openelb
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 8080
  externalTrafficPolicy: Cluster
```

重要提示：

```
lb.kubesphere.io/v1alpha1: openelb 用来指定该 Service 使用 OpenELB。
protocol.openelb.kubesphere.io/v1alpha1: layer2 表示指定 OpenELB 用于 Layer2 模式。
eip.openelb.kubesphere.io/v1alpha2: layer2-eip 用来指定了 OpenELB 使用的 Eip 对象，如果未配置此注解，
```

创建开始：

```
kubectl apply -f layer2-openelb.yaml layer2-svc.yaml
```

# 六、结果查看

1. 服务查看

   ```
   # kubectl get svc
   NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
   kubernetes   ClusterIP      10.233.0.1      <none>         443/TCP        20h
   layer2-svc   LoadBalancer   10.233.13.139   192.168.66.91   80:32658/TCP   14s
   ```

2. curl 192.168.66.91可以得到结果，在局域网下，访问192.168.66.91也可以得到结果

# 七、ingress整合

找到ingress 的kind: Service项

```
metadata:
  annotations:
    lb.kubesphere.io/v1alpha1: openelb
    protocol.openelb.kubesphere.io/v1alpha1: layer2
    eip.openelb.kubesphere.io/v1alpha2: layer2-eip
```



