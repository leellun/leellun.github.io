---
title: kubesphere快速部署mysql
date: 2022-07-26 18:38:02
categories:
  - 服务器
tags:
  - kubernetes 
  - k8s
  - kubesphere
  - mysql
---

# 一、docker部署方式

```
docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:8.0 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```



# 二、k8s有状态部署方式

## 2.1 准备配置文件

创建my-cnf的ConfigMap，键为my.cnf

```
[client]
default-character-set=utf8mb4
[mysql]
default-character-set=utf8mb4
[mysqld]
max_connections = 2000
secure_file_priv=/var/lib/mysql
basedir=/var/lib/mysql
datadir=/var/lib/mysql/data
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
skip-name-resolve
open_files_limit = 65535
table_open_cache = 128
log_error = /var/lib/mysql/mysql-error.log
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /var/lib/mysql/mysql-slow.log
default-storage-engine = InnoDB
innodb_file_per_table = 1
innodb_open_files = 500
innodb_buffer_pool_size = 64M
innodb_write_io_threads = 4
innodb_read_io_threads = 4
innodb_thread_concurrency = 0
innodb_purge_threads = 1
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 2M
innodb_log_file_size = 32M
innodb_log_files_in_group = 3
innodb_max_dirty_pages_pct = 90
innodb_lock_wait_timeout = 120
bulk_insert_buffer_size = 8M
myisam_sort_buffer_size = 8M
myisam_max_sort_file_size = 10G
myisam_repair_threads = 1
interactive_timeout = 28800
wait_timeout = 28800
[mysqldump]
quick
max_allowed_packet = 16M
[myisamchk]
key_buffer_size = 8M
sort_buffer_size = 8M
read_buffer = 4M
write_buffer = 4M
```

![1663167746173](kubesphere快速部署mysql/1663167746173.png)

![1663170735856](kubesphere快速部署mysql/1663170735856.png)





## 2.2 mysql有状态持久卷创建

![1663167951290](kubesphere快速部署mysql/1663167951290.png)

## 2.3 创建mysql服务

![1663168060307](kubesphere快速部署mysql/1663168060307.png)

配置文件挂在：持久卷可以创建也可以指定持久卷模板，地址 /var/lib/mysql

![1663169165653](kubesphere快速部署mysql/1663169165653.png)

存储卷挂载：挂载配置文件my.cnf，挂载/etc/mysql/my.cnf

![1663169211109](kubesphere快速部署mysql/1663169211109.png)

# 