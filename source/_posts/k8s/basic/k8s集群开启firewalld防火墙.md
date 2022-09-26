---
title: k8s集群开启firewalld防火墙
date: 2021-08-02 19:18:02
categories:
  - 服务器
  - k8s
tags:
  - kubernetes 
  - k8s
  - firewalld
  - 防火墙
---

# 一、基础设置

```
# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

#将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效
```

# 二、假设k8s部署情况如下：

| 主机名       | 主机IP    |
| ------------ | --------- |
| k8s-master01 | 10.0.0.10 |
| k8s-node01   | 10.0.0.11 |
| k8s-node02   | 10.0.0.21 |
| k8s-node03   | 10.0.0.22 |

# 三、所有机器上执行如下命令：

```
# 确保开启防火墙服务
systemctl restart firewalld

# 将集群内所有的节点IP配置到防火墙可信区中
firewall-cmd --permanent --zone=trusted --add-source=10.0.0.10
firewall-cmd --permanent --zone=trusted --add-source=10.0.0.11
firewall-cmd --permanent --zone=trusted --add-source=10.0.0.21
firewall-cmd --permanent --zone=trusted --add-source=10.0.0.22

# 增加防火墙规则
firewall-cmd --permanent --direct --add-rule ipv4 filter INPUT 1 -j ACCEPT -m comment --comment "kube-proxy redirects"
firewall-cmd --permanent --direct --add-rule ipv4 filter FORWARD 1  -j ACCEPT -m comment --comment "docker subnet"

# 设置防火墙伪装ip, 打开NAT，默认是关闭状态
firewall-cmd --add-masquerade --permanent

# 所有k8s的NodePort端口添加到例外
firewall-cmd --permanent --zone=public --add-port=30000-32767/tcp

# 重新加载配置
firewall-cmd --reload
```

