---
title: etcd集群
date: 2022-03-10 20:14:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - etcd
---

# 一、准备环境

| 角色   | IP            |
| ------ | ------------- |
| etcd-1 | 192.168.66.31 |
| etcd-2 | 192.168.66.41 |
| etcd-3 | 192.168.66.42 |

# 二、软件下载

单独为etcd放置一个目录，方便后面直接通过scp命令迁移

```
mkdir /opt/etcd
mkdir /opt/etcd/{bin,cfg,ssl,data,wal} –p
```

下载

```
wget -c https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
tar -zxf etcd-v3.5.0-linux-amd64.tar.gz -C etcd-v3.5.0
```

移动执行文件并配置环境

```
cd etcd-v3.5.0
mv ./{etcd,etcdctl,etcdutl} /opt/etcd/bin/
# 将/opt/etcd/bin 加入PATH
vi /etc/profile
```

# 三、证书生成

工具下载：

```
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl_1.5.0_linux_amd64 -o cfssl
chmod +x cfssl
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssljson_1.5.0_linux_amd64 -o cfssljson
chmod +x cfssljson
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl-certinfo_1.5.0_linux_amd64 -o cfssl-certinfo
chmod +x cfssl-certinfo
mv {cfssl,cfssljson,cfssl-certinfo} /usr/local/bin
```

（1）自签证书颁发机构（CA）

```
[root@localhost etcd]# cat > ca-config.json<< EOF 
{
    "signing":{
        "default":{
            "expiry":"87600h"
        },
        "profiles":{
            "kubernetes":{
                "expiry":"87600h",
                "usages":[
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
[root@localhost etcd]# cat > ca-csr.json<< EOF 
{
    "CN":"etcd CA",
    "key":{
        "algo":"rsa",
        "size":2048
    },
    "names":[
        {
            "C":"CN",
            "L":"Beijing",
            "ST":"Beijing"
        }
    ]
}
EOF
```

生成 CA 秘钥文件（`ca-key.pem`）和证书文件（`ca.pem`） ：

```
[root@localhost etcd]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2022/07/31 17:15:14 [INFO] generating a new CA key and certificate from CSR
2022/07/31 17:15:14 [INFO] generate received request
2022/07/31 17:15:14 [INFO] received CSR
2022/07/31 17:15:14 [INFO] generating key: rsa-2048
2022/07/31 17:15:15 [INFO] encoded CSR
2022/07/31 17:15:15 [INFO] signed certificate with serial number 504349668567155459345189436720647214038928670128
```

（2）使用自签 CA 签发 Etcd HTTPS 证书

创建证书申请文件：  

```
cat > etcd-csr.json<< EOF
{
    "CN":"etcd",
    "hosts":[
        "192.168.66.31",
        "192.168.66.41",
        "192.168.66.42"
    ],
    "key":{
        "algo":"rsa",
        "size":2048
    },
    "names":[
        {
            "C":"CN",
            "L":"BeiJing",
            "ST":"BeiJing"
        }
    ]
}
EOF
```

生成证书： 为 API 服务器生成秘钥和证书，默认会分别存储为`etcd-key.pem` 和 `etcd.pem` 两个文件。 

```
[root@localhost etcd]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
2022/07/31 17:34:11 [INFO] generate received request
2022/07/31 17:34:11 [INFO] received CSR
2022/07/31 17:34:11 [INFO] generating key: rsa-2048
2022/07/31 17:34:11 [INFO] encoded CSR
2022/07/31 17:34:11 [INFO] signed certificate with serial number 550588339086205748107774212753833209082394411557
```

为etcd放置证书

```
mv {ca.pem , etcd-key.pem , etcd.pem} /opt/etcd/ssl
```

# 四、etcd配置文件

命令行参数：[Configuration flags | etcd](https://etcd.io/docs/v3.5/op-guide/configuration/) 

yaml配置文件：[etcd/etcd.conf.yml.sample at main · etcd-io/etcd · GitHub](https://github.com/etcd-io/etcd/blob/main/etcd.conf.yml.sample) 

这里先在etcd-1配置好，下面的etcd-2、etcd-3等后面再配置

etcd-1

```yaml
name: "etcd-1"
data-dir: "/opt/etcd/data"
wal-dir: "/opt/etcd/wal"
# 触发磁盘快照的已提交事务数。
snapshot-count: 10000
# 心跳
heartbeat-interval: 100
# 选举超时的时间(毫秒)。
election-timeout: 1000
# 当后端大小超过给定的配额时发出告警。0表示使用默认配额。
quota-backend-bytes: 0
# 用于侦听对等流量的逗号分隔的url列表。
listen-peer-urls: https://192.168.66.31:2380
# 用于侦听客户机通信的逗号分隔的url列表。
listen-client-urls: https://192.168.66.31:2379
# 快照文件保留的最大数量(0为无限制)。
max-snapshots: 5
# 保留wal文件的最大数量(0是无限的)。
max-wals: 5
# 用于CORS(跨源资源共享)的逗号分隔的源白列表。
#cors:

# 这个成员的对等url的列表，以通告给集群的其他成员。url需要是逗号分隔的列表。
initial-advertise-peer-urls: https://192.168.66.31:2380
#这个成员的对等url的列表，以通告给集群的其他成员。url需要是逗号分隔的列表。
advertise-client-urls: https://192.168.66.31:2379

# 后面做keepalived haproxy vip
discovery: ''
# Valid values include 'exit', 'proxy'
discovery-fallback: 'proxy'
# HTTP proxy to use for traffic to discovery service.
discovery-proxy: ''
# DNS domain used to bootstrap initial cluster.
discovery-srv: ''
# Initial cluster configuration for bootstrapping.
initial-cluster: 'etcd-1=https://192.168.66.31:2380,etcd-2=https://192.168.66.41:2380,etcd-3=https://192.168.66.42:2380'
# Initial cluster token for the etcd cluster during bootstrap.
initial-cluster-token: 'etcd-cluster'
# Initial cluster state ('new' or 'existing').
initial-cluster-state: 'new'

# 拒绝可能导致仲裁丢失的重新配置请求。
strict-reconfig-check: false

# 通过HTTP服务器启用运行时分析数据
enable-pprof: true

# Valid values include 'on', 'readonly', 'off'
proxy: 'off'

# Time (in milliseconds) an endpoint will be held in a failed state.
proxy-failure-wait: 5000

# Time (in milliseconds) of the endpoints refresh interval.
proxy-refresh-interval: 30000

# Time (in milliseconds) for a dial to timeout.
proxy-dial-timeout: 1000

# Time (in milliseconds) for a write to timeout.
proxy-write-timeout: 5000

# Time (in milliseconds) for a read to timeout.
proxy-read-timeout: 0

client-transport-security:
  # Path to the client server TLS cert file.
  cert-file: /opt/etcd/ssl/etcd.pem

  # Path to the client server TLS key file.
  key-file: /opt/etcd/ssl/etcd-key.pem

  # Enable client cert authentication.
  client-cert-auth: false

  # Path to the client server TLS trusted CA cert file.
  trusted-ca-file: /opt/etcd/ssl/ca.pem

  # Client TLS using generated certificates
  auto-tls: false

peer-transport-security:
  # Path to the peer server TLS cert file.
  cert-file: /opt/etcd/ssl/etcd.pem

  # Path to the peer server TLS key file.
  key-file: /opt/etcd/ssl/etcd-key.pem

  # Enable peer client cert authentication.
  client-cert-auth: false

  # Path to the peer server TLS trusted CA cert file.
  trusted-ca-file: /opt/etcd/ssl/ca.pem

  # Peer TLS using generated certificates.
  auto-tls: false

# The validity period of the self-signed certificate, the unit is year.
self-signed-cert-validity: 1

# Enable debug-level logging for etcd.
log-level: info

logger: zap

# Specify 'stdout' or 'stderr' to skip journald logging even when running under systemd.
log-outputs: [stderr]

# Force to create a new one member cluster.
force-new-cluster: false

auto-compaction-mode: periodic
auto-compaction-retention: "1"
```

etcd-2，其它配置与上面一样

```yaml
name: "etcd-2"
listen-peer-urls: https://192.168.66.41:2380
# 用于侦听客户机通信的逗号分隔的url列表。
listen-client-urls: https://192.168.66.41:2379
# 这个成员的对等url的列表，以通告给集群的其他成员。url需要是逗号分隔的列表。
initial-advertise-peer-urls: https://192.168.66.41:2380
#这个成员的对等url的列表，以通告给集群的其他成员。url需要是逗号分隔的列表。
advertise-client-urls: https://192.168.66.41:2379
```

etcd-3

```yaml
name: "etcd-2"
listen-peer-urls: https://192.168.66.41:2380
# 用于侦听客户机通信的逗号分隔的url列表。
listen-client-urls: https://192.168.66.41:2379
# 这个成员的对等url的列表，以通告给集群的其他成员。url需要是逗号分隔的列表。
initial-advertise-peer-urls: https://192.168.66.41:2380
#这个成员的对等url的列表，以通告给集群的其他成员。url需要是逗号分隔的列表。
advertise-client-urls: https://192.168.66.41:2379
```

# 五、配置服务

```
cat > etcd.service << EOF 
[Unit] 
Description=Etcd Server 
After=network.target 
After=network-online.target 
Wants=network-online.target 
[Service] 
Type=notify 
ExecStart=/opt/etcd/bin/etcd  --config-file /opt/etcd/cfg/etcd.yml 
Restart=on-failure 
LimitNOFILE=65536 
[Install] 
WantedBy=multi-user.target 
EOF
```

将服务放置该存在的地方：

```
cp etcd.service /etc/systemd/system/
```

# 六、文件生成完毕，查看结果

```
[root@localhost etcd]# pwd
/opt/etcd
[root@localhost etcd]# ls -l
总用量 4
drwxr-xr-x. 2 root root  45 7月  31 18:15 bin
drwxr-xr-x. 2 root root  21 7月  31 19:55 cfg
drwxr-xr-x. 3 root root  19 7月  31 20:35 data
-rw-r--r--. 1 root root 283 7月  31 20:49 etcd.service
drwxr-xr-x. 2 root root  57 7月  31 18:55 ssl
drwx------. 2 root root  50 7月  31 20:35 wal
```

- bin etcd二进制命令，etcd  etcdctl  etcdutl
- cfg 步骤四生成的etcd yaml配置文件 ，etcd.yml
- data etcd数据文件目录
- wal 日志文件目录
- ssl 证书文件，ca.pem  server-key.pem  server.pem

# 七、同步文件到其它的node

```
scp /opt/etcd root@192.168.66.41:/opt/
scp /opt/etcd root@192.168.66.42:/opt/
```

将服务防止该有地方:

```
cp etcd.service /etc/systemd/system/
```

执行步骤四，修改etcd-2 etcd-3的配置

# 八、启动服务

```
#分别在每一个node上执行
systemctl start etcd
```

# 九、查看集群状态

```
$ etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/etcd.pem --key=/opt/etcd/ssl/etcd-key.pem --endpoints="https://192.168.66.31:2379,https://192.168.66.41:2379,https://192.168.66.42:2379" endpoint status --write-out=table
+------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|        ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.66.31:2379 | 1f46bee47a4f04aa |   3.5.0 |   20 kB |     false |      false |         7 |         26 |                 26 |        |
| https://192.168.66.41:2379 |   b3e5838df5f510 |   3.5.0 |   20 kB |     false |      false |         7 |         26 |                 26 |        |
| https://192.168.66.42:2379 | a437554da4f2a14c |   3.5.0 |   20 kB |      true |      false |         7 |         26 |                 26 |        |
+------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

# 十、问题排查

1 member a437554da4f2a14c has already been bootstrapped

三种方式解决。

(1)修改成  --initial-cluster-state=existing 

(2)删除所有etcd节点的 data-dir 文件（不删也行），重启各个节点的etcd服务，这个时候，每个节点的data-dir的数据都会被更新，就不会有以上故障了 

(3)第三种方式是复制其他节点的data-dir中的内容，以此为基础上以 --force-new-cluster 的形式强行拉起一个，然后以添加新成员的方式恢复这个集群。 