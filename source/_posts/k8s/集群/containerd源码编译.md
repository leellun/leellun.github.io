---
title: containerd源码编译
date: 2022-03-05 20:14:02
categories:
  - 服务器
tags:
  - containerd 
---

# 一、go安装

yum安装：

```
yum install -y epel-release
yum install -y golang
```

源码安装：

```
wget https://storage.googleapis.com/golang/getgo/installer_linux
chmod +x ./installer_linux
./installer_linux
source ~/.bash_profile
```

# 二、runc安装

编译

```
git clone https://github.com/opencontainers/runc
cd runc
make&& make install
```

出现错误No package 'libseccomp' found：

```
yum install -y libseccomp-devel
```

验证结果

```
[root@localhost runc]# runc --version   
runc version 1.1.0+dev
commit: v1.1.0-254-gf835196
spec: 1.0.2-dev
go: go1.17.10
libseccomp: 2.3.1
```

# 三、containerd

```
git clone https://github.com/containerd/containerd
wget -c https://github.com/protocolbuffers/protobuf/releases/download/v3.11.4/protoc-3.11.4-linux-x86_64.zip
sudo unzip protoc-3.11.4-linux-x86_64.zip -d /usr/local
```

centos环境

```
yum install btrfs-progs-devel
```

编译

```
cd containerd
make
make install
```

# 四、安装问题

1 出现错误No package 'libseccomp' found：

```
yum install libseccomp-devel
```

2 failed to create CRI service: failed to find snap...r \"overlayfs\""

```
#查看文件系统，如果centos7 则文件系统可能xfs，通过一下命令可以查看
$ df -T
文件系统                              类型        1K-块    已用     可用 已用% 挂载点
/dev/mapper/centos-root               xfs      14379008 2803720 11575288   20% /
```

解决方案：

方法一：添加新分区执行ftype=1,格式化xfs分区并且设置ftype=1

```
$ mkfs.xfs -f -n ftype=1 /dev/sda3
```

方法二：如果不能 添加新分区 也不能格式化现有分区，将cri的snapshotter设置成native

```
$ vi /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      ignore_rdt_not_enabled_errors = false
      no_pivot = false
      snapshotter = "native"
```





