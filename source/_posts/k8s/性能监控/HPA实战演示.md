---
title: HPA实战演示
date: 2021-09-03 21:14:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
  - hpa
---

# 镜像拉取

将镜像放到本地私有云上

```
docker pull mirrorgooglecontainers/hpa-example
docker tag mirrorgooglecontainers/hpa-example harborcloud.com/library/hpa-example
docker push reg.westos.org/library/hpa-example
```

# 配置文件

hap.yaml 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: harborcloud.com/library/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```

```
[root@k8s-master01 prometheus]# kubectl get svc
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
php-apache                      ClusterIP   10.101.42.132    <none>        80/TCP          10m
```

# 横向扩展

创建 Horizontal Pod Autoscaler 

```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl top pod
```

查看

```
[root@k8s-master01 prometheus]# kubectl top pod
NAME                                            CPU(cores)   MEMORY(bytes)   
grafana-6b44f56546-2s6dn                        1m           46Mi            
kubernetes-dashboard-68c6f6698c-4rcs5           1m           12Mi            
metrics-server-6744b4c64f-jt6vx                 6m           14Mi            
php-apache-f8bcb65f4-ndtjd                      1m           11Mi            
prometheus-alertmanager-cbfbd87c8-4z72h         1m           14Mi            
prometheus-kube-state-metrics-77ddf69b4-hhmxp   2m           13Mi            
prometheus-node-exporter-fd4zl                  1m           13Mi            
prometheus-pushgateway-5dd5bfffb9-bm85t         1m           8Mi             
prometheus-server-558fc5f6fc-756m7              7m           207Mi  
```

# 增加负载 

```
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://10.101.42.132; done"
```

# 查看hpa

```
[root@k8s-master01 ~]# kubectl get hpa
NAME               REFERENCE                     TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   <unknown>/80%   1         15        3          7d11h
php-apache         Deployment/php-apache         221%/50%        1         10        4          13m
[root@k8s-master01 ~]# kubectl top pod
NAME                                            CPU(cores)   MEMORY(bytes)   
grafana-6b44f56546-2s6dn                        10m          52Mi            
kubernetes-dashboard-68c6f6698c-4rcs5           2m           8Mi             
load-generator                                  9m           0Mi             
load-generator2                                 9m           0Mi             
metrics-server-6744b4c64f-jt6vx                 9m           19Mi            
php-apache-f8bcb65f4-6vgm9                      149m         9Mi             
php-apache-f8bcb65f4-8hjd6                      154m         9Mi             
php-apache-f8bcb65f4-fttc2                      141m         9Mi             
php-apache-f8bcb65f4-hmlhf                      151m         9Mi             
php-apache-f8bcb65f4-l57gg                      147m         8Mi             
php-apache-f8bcb65f4-ndtjd                      151m         9Mi             
php-apache-f8bcb65f4-r7h4b                      149m         11Mi            
php-apache-f8bcb65f4-vq2z5                      145m         8Mi             
```

