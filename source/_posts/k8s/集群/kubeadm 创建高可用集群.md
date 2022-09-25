---
title: kubeadm 创建高可用集群
date: 2022-03-12 18:11:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - 高可用
---

# 一、环境

| 服务器       | IP         |
| ------------ | ---------- |
| k8s-master01 | 10.0.0.101 |
| k8s-master02 | 10.0.0.102 |
| k8s-master03 | 10.0.0.103 |
| k8s-node01   | 10.0.0.201 |
| k8s-node02   | 10.0.0.202 |
| k8s-node03   | 10.0.0.203 |
| 虚拟IP       | 10.0.0.150 |

# 二、准备操作

```
yum update
# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config 
setenforce 0  

# 关闭swap
# 临时 永久
swapoff -a 
sed -ri 's/.*swap.*/#&/' /etc/fstab    

#这里可能不存在
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

#将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# 强制把系统时间写入CMOS
yum install ntpdate -y
ntpdate time.windows.com
clock -w
```

# 三、安装必要软件

## docker keepalived haproxy安装

官网：自 1.24 版起，Dockershim 已从 Kubernetes 项目中移除。阅读 Dockershim 移除的常见问题了解更多详情。 

所有节点安装

```
# 安装docker支持
yum install docker-ce-18.06.0.ce-3.el7 docker-ce-cli-18.06.0.ce-3.el7 containerd.io docker-compose-plugin -y
# 安装k8s支持 
yum install -y kubeadm-1.24.0 kubectl-1.24.0 kubelet-1.24.0
```

master软件支持

```
# LVS支持
yum install -y keepalived haproxy
```

## cri-dockerd 安装启动(所有节点)

https://github.com/Mirantis/cri-dockerd

下载cri-dockerd源码

```
git clone https://github.com/Mirantis/cri-dockerd.git
```

配置cri-dockerd服务

```
wget https://storage.googleapis.com/golang/getgo/installer_linux
chmod +x ./installer_linux
./installer_linux
source ~/.bash_profile

cd cri-dockerd
mkdir bin
go get && go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
cp -a packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
```

# 四、配置

官网keepalived+haproxy配置方案：[kubeadm/ha-considerations.md at main · kubernetes/kubeadm · GitHub](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing) 

## keepalived 配置

vrrp_strict需要注释，不然就必须开启firewalld

```
! Configuration File for keepalived

global_defs {
   smtp_server 10.0.0.150
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
#   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}
vrrp_instance VI_1 {
    state MASTER
    interface eno16777736
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.0.0.150/24
    }
    track_script {
        chk_haproxy
    }
}

```

```
systemctl start keepalived
```

## haproxy配置

```
frontend  main *:16443
    mode tcp
    default_backend             app
backend app
    mode        tcp
    balance     roundrobin
    server  app1 10.0.0.101:6443 check
    server  app2 10.0.0.102:6443 check
    server  app3 10.0.0.103:6443 check
```

```
systemctl start haproxy
```

## k8s init初始构建文件kubeadm-config.yaml

```
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.24.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
controlPlaneEndpoint: "10.0.0.150:16443"
scheduler: {}
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

# 五 安装k8s

## 1 在keepalive master安装k8s

### 初始化安装k8s

在master01上安装，注意这里master01需保证10.0.0.150绑定到上面

命令行参数方式：

```
kubeadm init \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.24.0 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16 \
--control-plane-endpoint "10.0.0.150:16443" \
--upload-certs
```

配置文件方式：

```
kubeadm init --config kubeadm-config.yaml --upload-certs
```

结果：

```
...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.0.0.150:16443 --token ahps3w.1pdntr699ijvouxu \
        --discovery-token-ca-cert-hash sha256:e94fee115e9bf7d8f91df2268025fde08bd899e010c27fe97d2ee9e53326d028 \
        --control-plane --certificate-key a24d302107d362c3227c171acbeb6eb613688480e2019c25807d6cc53f8a0dc1

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.150:16443 --token ahps3w.1pdntr699ijvouxu \
        --discovery-token-ca-cert-hash sha256:e94fee115e9bf7d8f91df2268025fde08bd899e010c27fe97d2ee9e53326d028
```

### 为root用户配置

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 网络插件

上面提到的kubectl apply -f [podnetwork].yaml，这里采用flannel

```
 kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 加入control-plane时间过期重新生成认证加入

集群节点加入认证生成

```
kubeadm token create --certificate-key a24d302107d362c3227c171acbeb6eb613688480e2019c25807d6cc53f8a0dc1 --print-join-command
```

普通节点加入认证生成

```
kubeadm token create  --print-join-command
```

## 2 在keepalived backup上执行加入控制面板

```
kubeadm join 10.0.0.150:16443 --token ahps3w.1pdntr699ijvouxu \
        --discovery-token-ca-cert-hash sha256:e94fee115e9bf7d8f91df2268025fde08bd899e010c27fe97d2ee9e53326d028 \
        --control-plane --certificate-key a24d302107d362c3227c171acbeb6eb613688480e2019c25807d6cc53f8a0dc1
```

成功结果：

```
This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

为当前用户配置权限证书之类:

```
mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 3 普通node节点加入

```
kubeadm join 10.0.0.150:16443 --token ahps3w.1pdntr699ijvouxu \
        --discovery-token-ca-cert-hash sha256:e94fee115e9bf7d8f91df2268025fde08bd899e010c27fe97d2ee9e53326d028
```

如果提示token认证过期就去master重新生成加入认证

```
kubeadm token create  --print-join-command
```

# 六 安装结果

```
[root@k8s-master01 k8s-install]# kubectl get nodes
NAME           STATUS   ROLES           AGE     VERSION
k8s-master01   Ready    control-plane   58m     v1.24.0
k8s-master02   Ready    control-plane   55m     v1.24.0
k8s-master03   Ready    control-plane   41m     v1.24.0
k8s-node01     Ready    <none>          33m     v1.24.0
k8s-node02     Ready    <none>          4m26s   v1.24.0
k8s-node03     Ready    <none>          26m     v1.24.0
```

# 七 验证结果

创建nginx并且暴露服务

```
[root@k8s-master01 deployment]# kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
[root@k8s-master01 deployment]# kubectl create service nodeport nginx --tcp=80:80         
service/nginx created
[root@k8s-master01 deployment]# kubectl get svc,pod -o wide
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE    SELECTOR
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        169m   <none>
service/nginx        NodePort    10.109.142.181   <none>        80:31454/TCP   34s    app=nginx

NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE         NOMINATED NODE   READINESS GATES
pod/nginx-8f458dc5b-m65bl   1/1     Running   0          3m14s   172.17.0.2   k8s-node02   <none>           <none>
```

地址访问

```
[root@k8s-master01 deployment]# curl 10.0.0.202:31454
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

