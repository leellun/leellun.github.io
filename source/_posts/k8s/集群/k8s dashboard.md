---
title: kubernetes 入门学习 dashboard
date: 2021-08-02 19:18:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
  - dashboard
---



# 1 dashboard安装

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

结果：

```shell
namespace/kubernetes-dashboard created  # 命名空间kubernetes-dashboard
serviceaccount/kubernetes-dashboard created # 服务账号kubernetes-dashboard
service/kubernetes-dashboard created # 服务kubernetes-dashboard
secret/kubernetes-dashboard-certs created #  secret创建
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created 
configmap/kubernetes-dashboard-settings created # 配置
role.rbac.authorization.k8s.io/kubernetes-dashboard created # 角色和角色绑定
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard configured
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard unchanged
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

查看校验资源的安装情况

```shell
$ kubectl get deployments -n kubernetes-dashboard
$ kubectl get services -n kubernetes-dashboard
$ kubectl get pods -n kubernetes-dashboard
$ kubectl get secrets -n kubernetes-dashboard
$ kubectl get configMap -n kubernetes-dashboard
$ kubectl  get services -n kubernetes-dashboard
```

# 2 开放外部访问端口

kubernetes-dashbaord安装完毕后，kubernetes-dashboard默认service的类型为ClusterIP，为了从外部访问控制面板，开放为NodePort类型

过程：编辑之前——编辑——编辑之后

```shell
[root@k8smaster influxdb]# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.104.64.108   <none>        8000/TCP   2m33s
kubernetes-dashboard        ClusterIP   10.102.42.206   <none>        443/TCP    2m33s
[root@k8smaster influxdb]# kubectl edit svc/kubernetes-dashboard -n kubernetes-dashboard
service/kubernetes-dashboard edited
[root@k8smaster influxdb]# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.104.64.108   <none>        8000/TCP        10m
kubernetes-dashboard        NodePort    10.102.42.206   <none>        443:30367/TCP   10m
```

# 3 授权用户访问集群

dashboard-rbac.yaml定义

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: happycloudlab 
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: happycloudlab
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: happycloudlab
  namespace: kubernetes-dashboard
```

操作：

```shell
[root@k8smaster influxdb]# vi dashboard-rbac.yaml
[root@k8smaster influxdb]# kubectl create -f dashboard-rbac.yaml 
serviceaccount/happycloudlab created
clusterrolebinding.rbac.authorization.k8s.io/happycloudlab created
```

# 4 获取token

通过token字段来登陆，token通过base64加密,这里的happycloudlab-token-*须通过命令查看，具体是哪一个。

## 4.1 获取具体secret并且提取yaml信息

```shell
[root@k8smaster influxdb]# kubectl get secret -n kubernetes-dashboard
NAME                               TYPE                                  DATA   AGE
default-token-vzxqv                kubernetes.io/service-account-token   3      14m
happycloudlab-token-5dxhd          kubernetes.io/service-account-token   3      44s
kubernetes-dashboard-certs         Opaque                                0      14m
kubernetes-dashboard-csrf          Opaque                                1      14m
kubernetes-dashboard-key-holder    Opaque                                2      14m
kubernetes-dashboard-token-fqtcx   kubernetes.io/service-account-token   3      14m
[root@k8smaster influxdb]# kubectl get secrets -n kubernetes-dashboard happycloudlab-token-5dxhd -o yaml
```

![1630834989795](.\k8s dashboard\1630834989795.png)

## 4.2 通过echo获取base64编码token

```shell
[root@k8smaster influxdb]# echo 'ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklrWmtSVE5UYVhOT2RrUlFWRU5vV2xOeE5FSkJjMGQxTURkM2JETkJOa2Q2TFZaNVdWbzFhV3hOV0ZFaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUpyZFdKbGNtNWxkR1Z6TFdSaGMyaGliMkZ5WkNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZqY21WMExtNWhiV1VpT2lKb1lYQndlV05zYjNWa2JHRmlMWFJ2YTJWdUxUVmtlR2hrSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXpaWEoyYVdObExXRmpZMjkxYm5RdWJtRnRaU0k2SW1oaGNIQjVZMnh2ZFdSc1lXSWlMQ0pyZFdKbGNtNWxkR1Z6TG1sdkwzTmxjblpwWTJWaFkyTnZkVzUwTDNObGNuWnBZMlV0WVdOamIzVnVkQzUxYVdRaU9pSm1NemM1TkdOak5TMHhNMk5pTFRRMVpEQXRPRFJpWWkxaU5UVmtNRFF3TkdVMU9Ea2lMQ0p6ZFdJaU9pSnplWE4wWlcwNmMyVnlkbWxqWldGalkyOTFiblE2YTNWaVpYSnVaWFJsY3kxa1lYTm9ZbTloY21RNmFHRndjSGxqYkc5MVpHeGhZaUo5LkpBZVFBRS1CT1RGZDhXOGp3cEFpa0lTQlF4N2N6bmhySmZneDFxLXA5OTRLSUI5aXRfeFZ3bUlEaEI4bVlZQTJQLXRBNWp4d2d6NDZHamdSZTlfc1dYdFRrTHYwT3dpYmtNeDU3Q0RFVzkxV09CNzkyeTZiOWNWX1BhV3hPVkRyZFFvSVo3S3A0OVFITFNkN1lhREl1eE15UDlzX3pQaTI3dmc0YUZwLUFLS1ZWV0NqcDFvaURFM213Y3FJd2xha3JySW5ZUmg0THNKTFc5TTBVc3BCRklZUFhGNEh5QWxvX2NGX29qNVlHMUFJWUJGTGtlQUZyaEYwalFzQmhseHVYVVRubC01TmN2WjlDRXJJbGJ4VEpfMWtLcVc2UjBwOXFsZ1dpZzFPamhFNzg4NWs0dnVzeDM5S004d2U1ZHBJbjV6WDdIMzBFSjgwOGRpOGVyTFFQUQ==' | base64 -d
```

得到结果：

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IkZkRTNTaXNOdkRQVENoWlNxNEJBc0d1MDd3bDNBNkd6LVZ5WVo1aWxNWFEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJoYXBweWNsb3VkbGFiLXRva2VuLTVkeGhkIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImhhcHB5Y2xvdWRsYWIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJmMzc5NGNjNS0xM2NiLTQ1ZDAtODRiYi1iNTVkMDQwNGU1ODkiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6aGFwcHljbG91ZGxhYiJ9.JAeQAE-BOTFd8W8jwpAikISBQx7cznhrJfgx1q-p994KIB9it_xVwmIDhB8mYYA2P-tA5jxwgz46GjgRe9_sWXtTkLv0OwibkMx57CDEW91WOB792y6b9cV_PaWxOVDrdQoIZ7Kp49QHLSd7YaDIuxMyP9s_zPi27vg4aFp-AKKVVWCjp1oiDE3mwcqIwlakrrInYRh4LsJLW9M0UspBFIYPXF4HyAlo_cF_oj5YG1AIYBFLkeAFrhF0jQsBhlxuXUTnl-5NcvZ9CErIlbxTJ_1kKqW6R0p9qlgWig1OjhE7885k4vusx39KM8we5dpIn5zX7H30EJ808di8erLQPQ
```

# 5 登录dashboard

获取外网访问端口

```
[root@k8smaster influxdb]# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.104.64.108   <none>        8000/TCP        10m
kubernetes-dashboard        NodePort    10.102.42.206   <none>        443:30367/TCP   10m
```

填写token并登录

![1630835361904](.\k8s dashboard\1630835361904.png)

![1630835709026](.\1630835709026.png)