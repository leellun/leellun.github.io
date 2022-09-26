---
title: k8s集群安全实战—为指定用户设置操作权限
date: 2021-09-03 21:14:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
---

# 1 下载、解压并准备如下所示的命令行工具。 

```
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl_1.5.0_linux_amd64 -o cfssl
chmod +x cfssl
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssljson_1.5.0_linux_amd64 -o cfssljson
chmod +x cfssljson
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl-certinfo_1.5.0_linux_amd64 -o cfssl-certinfo
chmod +x cfssl-certinfo
```

将证书工具放到/usr/local/bin下面

```
mv cfssl /usr/local/bin/cfssl
mv cfssljson /usr/local/bin/cfssljson
mv cfssl-certinfo /usr/local/bin/cfssl-certinfo
```

# 2 准备配置文件 

创建一个 JSON 配置文件，用于 CA 证书签名请求（CSR） ，这里存放在/usr/local/k8s-install

```
vi /usr/local/k8s-install/devuser-csr.json
{
  "CN": "devuser",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "CN",
    "ST": "Beijing",
    "L": "Beijing",
    "O": "k8s",
    "OU": "System"
  }]
}
```

# 3 针对用户devuser生成证书文件

```
$ cd /etc/kubernetes/pki/
$ cfssl gencert -ca=ca.crt -ca-key=ca.key -profile=kubernetes /usr/local/k8s-install/devuser-csr.json|cfssljson -bare devuser
$ ls|grep devuser
devuser.csr
devuser-key.pem
devuser.pem
```

# 4 设置集群参数

```
$ export KUBE_APISERVER="https://10.0.0.10:6443"
$ cd /usr/local/k8s-install
$ kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/ca.crt 
--embed-certs=true --server=${KUBE_APISERVER} 
--kubeconfig=devuser.kubeconfig
```

得到结果：

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBD...
    server: https://10.0.0.10:6443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```

这里把证书导进来了

# 5 设置客户端认证参数

```
kubectl config set-credentials devuser \ 
--client-certificate=/etc/kubernetes/pki/devuser.pem \
--client-key=/etc/kubernetes/pki/devuser-key.pem \
--embed-certs=true \
--kubeconfig=devuser.kubeconfig          
```

得到结果：

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBD...
    server: https://10.0.0.10:6443
  name: kubernetes
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: devuser
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJU...
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSB...
```

得到用户的信息，用户名，证书以及私钥的信息。

# 6 提前创建dev空间

root用户下

````
$ kubectl create namespace dev
````

# 7 设置上下文参数

```
$ kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=devuser \
--namespace=dev \
--kubeconfig=devuser.kubeconfig
```

得到结果

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBD...
    server: https://10.0.0.10:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: dev
    user: devuser
  name: kubernetes
current-context: ""
kind: Config
preferences: {}
users:
- name: devuser
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJU...
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSB...
```

通过命令在devuser.kubeconfig文件中多出一个上下文：

```
- context:
    cluster: kubernetes
    namespace: dev
    user: devuser
  name: kubernetes
```

# 8 设置默认上下文

```
kubectl config use-context kubernetes --kubeconfig=devuser.kubeconfig
```

得到结果：

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBD...
    server: https://10.0.0.10:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: dev
    user: devuser
  name: kubernetes
current-context: kubernetes
kind: Config
preferences: {}
users:
- name: devuser
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJU...
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSB...
```

在文件中出现

```
current-context: kubernetes
```

# 9 创建角色绑定

```
$ kubectl create rolebinding devuser-admin-binding \
--clusterrole=admin \
--user=devuser \
--namespace=dev
```

# 10 将devuser.kubeconfig放到devuser home目录下

devuser用户，先创建/home/devuser/.kube文件夹

```
[devuser@k8s-master01 ~]$ mkdir .kube
```

root用户操作

```
[root@k8s-master01 devuser]# cp ./devuser.kubeconfig /home/devuser/.kube/config
[root@k8s-master01 devuser]# chown -R devuser:devuser /home/devuser/.kube/config
```

# 11 验证结果

```
[devuser@k8s-master01 ~]$ kubectl run nginx --image=nginx
[devuser@k8s-master01 ~]$ kubectl get pod 
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          2m33s
```

root用户查看

```
[root@k8s-master01 devuser]# kubectl get pod --all-namespaces
NAMESPACE       NAME                                        READY   STATUS      RESTARTS        AGE
dev             nginx                                       1/1     Running     0               2m1s
```

