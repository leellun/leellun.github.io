---
title: postgresql源码安装
date: 2022-11-10 20:14:02
categories:
  - 服务器
  - postgresql
tags:
  - postgresql
---

## 1、官网下载源码

```
# 浏览器下载较好
wget -c https://ftp.postgresql.org/pub/source/v14.0/postgresql-14.0.tar.gz
```

安装相关环境

```
yum install -y gcc-c++ gcc cmake ncurses-devel perl zlib*
```

## 2 源码编译

```
tar -zxf postgresql-14.0.tar.gz
cd postgresql-14.0

# 开始编译
./configure --prefix=/home/postgresql
make && make install 
```

## 3 初始化数据库

```
# 切换用户progres
su progres
mkdir /home/postgresql/data/

# 初始化数据库
/home/postgresql/bin/initdb -D /home/postgresql/data/
```

## 4 修改配置

```
vi /home/postgresql/data/pg_hba.conf
# replication privilege.
host    all             all             0.0.0.0/0               md5

vi /home/postgresql/data/postgresql.conf
# 修改监听地址
listen_addresses = '*' 
```

## 5 启动数据库

```
/home/postgresql/bin/pg_ctl -D /home/postgresql/data -l logfile start
```

修改密码

```
/home/postgresql/bin/psql
```

## 6 重新加载配置

不用重启，让配置生效

```
/home/postgresql/bin/pg_ctl -D /home/postgresql/data   reload
```

## 7 用户管理

```
-- 创建用户
CREATE USER name [[ WITH ] option [...]]

-- 创建角色 在PG中，用户与角色是没有区别的，角色默认没有login权限，无法登陆，如果授予login之后，也可以像用户一样登陆。
CREATE ROLE name [[ WITH ] option [...]]
```

option常用选项如下：

- SUPERUSER | NOSUPERUSER：创建出来的用户是否为超级用户
- CREATEDB | NOCREATEDB：创建出来的用户是否有create database的权限
- CREATEROLE | NOCREATEROLE：创建出来的用户是否有创建其它角色的权限
- CREATEUSER | NOCREATEUSER：创建出来的用户是否有创建其它用户的权限
- INHERIT | NOINHERIT：确定角色是否继承其它角色的权限
- LOGIN | NOLOGIN：创建出来的角色是否有登录权限
- CONNECTION LIMIT n：创建出来的角色并发连接数限制数量，默认值是“-1”,表示没有限制
- VALID UNTIL 'timestamp'：密码失效时间

修改密码：

```
ALTER USER 用户名 WITH PASSWORD 'xxx';
```

## 8 权限的管理

| 权限        | 说明           |
| ----------- | -------------- |
| TABLE       | 表操作权限     |
| column_name | 列操作权限     |
| SEQUENCE    | 序列操作权限   |
| DATABASE    | 数据库操作权限 |
| DOMAIN      | *域* 操作权限  |

### 8.1 授权

在postgresql数据库中，任何逻辑对象(包括数据库)都是有所有者的，也就是说数据库对象都是属于某个用户的，所以，无需把对象的权限赋予所有者，因为所有者默认就拥有所有的权限，在PG数据库中，删除及其修改对象的权限都不能赋予别的用户，它是所有者的固有权限，不能赋予或撤销，所有者也隐式地拥有把操作该对象的权限授予其他用户的权利。 

```
-- 将角色授予给另外一个角色
GRANT role_name[,...] TO role_name[,...] WITH ADMIN OPTION;

-- 角色回收
REVOKE role_name[,...] FROM role_name[,...][CASCADE | RESTRICT]
```

```
GRANT { { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER }
    [, ...] | ALL [ PRIVILEGES ] }
    ON { [ TABLE ] table_name [, ...]
         | ALL TABLES IN SCHEMA schema_name [, ...] }
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { { SELECT | INSERT | UPDATE | REFERENCES } ( column_name [, ...] )
    [, ...] | ALL [ PRIVILEGES ] ( column_name [, ...] ) }
    ON [ TABLE ] table_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { { USAGE | SELECT | UPDATE }
    [, ...] | ALL [ PRIVILEGES ] }
    ON { SEQUENCE sequence_name [, ...]
         | ALL SEQUENCES IN SCHEMA schema_name [, ...] }
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { { CREATE | CONNECT | TEMPORARY | TEMP } [, ...] | ALL [ PRIVILEGES ] }
    ON DATABASE database_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { USAGE | ALL [ PRIVILEGES ] }
    ON DOMAIN domain_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { USAGE | ALL [ PRIVILEGES ] }
    ON FOREIGN DATA WRAPPER fdw_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { USAGE | ALL [ PRIVILEGES ] }
    ON FOREIGN SERVER server_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { EXECUTE | ALL [ PRIVILEGES ] }
    ON { { FUNCTION | PROCEDURE | ROUTINE } routine_name [ ( [ [ argmode ] [ arg_name ] arg_type [, ...] ] ) ] [, ...]
         | ALL { FUNCTIONS | PROCEDURES | ROUTINES } IN SCHEMA schema_name [, ...] }
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { USAGE | ALL [ PRIVILEGES ] }
    ON LANGUAGE lang_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { { SELECT | UPDATE } [, ...] | ALL [ PRIVILEGES ] }
    ON LARGE OBJECT loid [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { { CREATE | USAGE } [, ...] | ALL [ PRIVILEGES ] }
    ON SCHEMA schema_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { CREATE | ALL [ PRIVILEGES ] }
    ON TABLESPACE tablespace_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT { USAGE | ALL [ PRIVILEGES ] }
    ON TYPE type_name [, ...]
    TO role_specification [, ...] [ WITH GRANT OPTION ]
    [ GRANTED BY role_specification ]

GRANT role_name [, ...] TO role_specification [, ...]
    [ WITH ADMIN OPTION ]
    [ GRANTED BY role_specification ]

where role_specification can be:

    [ GROUP ] role_name
  | PUBLIC
  | CURRENT_ROLE
  | CURRENT_USER
  | SESSION_USER
```

```
# 授权增删改查
GRANT update,delete,insert,select ON ALL TABLES IN SCHEMA public TO signalmap_rw;
```

例如：

```
# 给所有用户授予insert权限针对films表
GRANT INSERT ON films TO PUBLIC;

# 授权manuel用户在试图kinds的所有权限
GRANT ALL PRIVILEGES ON kinds TO manuel;

# 注意，如果由超级用户或types的所有者执行，上面的操作确实会授予所有特权，但如果由其他人执行，则只授予其他人拥有授予选项的权限。
GRANT admins TO joe;
```

```
-- 回收public模式的create权限
revoke create on schema public from public;
-- 创建只读用户，密码为readonly
create user readonly password 'readonly';
-- 授权public模式给readonly
grant usage on schema public to readonly;
-- 授权public模式的所有表权限给readonly用户
grant select on all tables in schema public to readonly;
-- 切换到readonly测试
\c - readonly 
-- 切换到postgres用户
\c - postgres
-- 将public模式的默认表查询权限授予readonly用户
alter default privileges in schema public grant select on tables to readonly;
```

```
-- 查看某用户的系统权限
SELECT * FROM  pg_roles WHERE rolname='postgres';

-- 查看某用户的表级别权限
select * from information_schema.table_privileges where grantee='postgres';

-- 查看某用户的usage权限
select * from information_schema.usage_privileges where grantee='postgres';

-- 查看某用户在存储过程函数的执行权限
select * from information_schema.routine_privileges where grantee='postgres'; 

-- 查看某用户在某表的列上的权限
select * from information_schema.column_privileges where grantee='postgres'; 

-- 查看当前用户能够访问的数据类型
select * from information_schema.data_type_privileges ; 

-- 查看用户自定义类型上授予的USAGE权限
select * from information_schema.udt_privileges where grantee='postgres';
```

### 8.2 回收权限

```
REVOKE [ GRANT OPTION FOR ]
    { { SELECT | INSERT | UPDATE | DELETE | TRUNCATE | REFERENCES | TRIGGER }
    [, ...] | ALL [ PRIVILEGES ] }
    ON { [ TABLE ] table_name [, ...]
         | ALL TABLES IN SCHEMA schema_name [, ...] }
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { { SELECT | INSERT | UPDATE | REFERENCES } ( column_name [, ...] )
    [, ...] | ALL [ PRIVILEGES ] ( column_name [, ...] ) }
    ON [ TABLE ] table_name [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { { USAGE | SELECT | UPDATE }
    [, ...] | ALL [ PRIVILEGES ] }
    ON { SEQUENCE sequence_name [, ...]
         | ALL SEQUENCES IN SCHEMA schema_name [, ...] }
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { { CREATE | CONNECT | TEMPORARY | TEMP } [, ...] | ALL [ PRIVILEGES ] }
    ON DATABASE database_name [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { USAGE | ALL [ PRIVILEGES ] }
    ON DOMAIN domain_name [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { USAGE | ALL [ PRIVILEGES ] }
    ON FOREIGN DATA WRAPPER fdw_name [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { USAGE | ALL [ PRIVILEGES ] }
    ON FOREIGN SERVER server_name [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { EXECUTE | ALL [ PRIVILEGES ] }
    ON { { FUNCTION | PROCEDURE | ROUTINE } function_name [ ( [ [ argmode ] [ arg_name ] arg_type [, ...] ] ) ] [, ...]
         | ALL { FUNCTIONS | PROCEDURES | ROUTINES } IN SCHEMA schema_name [, ...] }
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { USAGE | ALL [ PRIVILEGES ] }
    ON LANGUAGE lang_name [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { { SELECT | UPDATE } [, ...] | ALL [ PRIVILEGES ] }
    ON LARGE OBJECT loid [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { { CREATE | USAGE } [, ...] | ALL [ PRIVILEGES ] }
    ON SCHEMA schema_name [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { CREATE | ALL [ PRIVILEGES ] }
    ON TABLESPACE tablespace_name [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ GRANT OPTION FOR ]
    { USAGE | ALL [ PRIVILEGES ] }
    ON TYPE type_name [, ...]
    FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

REVOKE [ ADMIN OPTION FOR ]
    role_name [, ...] FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]

where role_specification can be:

    [ GROUP ] role_name
  | PUBLIC
  | CURRENT_ROLE
  | CURRENT_USER
  | SESSION_USER
```

Revoke insert privilege for the public on table `films`:

```
REVOKE INSERT ON films FROM PUBLIC;
```

Revoke all privileges from user `manuel` on view `kinds`:

```
REVOKE ALL PRIVILEGES ON kinds FROM manuel;
```

Note that this actually means “revoke all privileges that I granted”.

Revoke membership in role `admins` from user `joe`:

```
REVOKE admins FROM joe;
```

### 8.3 授权过程

```
# 所有表
grant all privileges on all tables in schema public  to sonaruser;
grant all privileges on schema public  to sonaruser;
```

