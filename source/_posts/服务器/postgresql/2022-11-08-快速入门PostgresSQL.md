---
title: 快速入门PostgresSQL
date: 2022-11-08 17:38:02
categories:
  - 服务器
  - postgresql
tags:
  - postgresssql
---

角色查看

```
postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

当前用户

```
postgres=# select current_user;
 current_user 
--------------
 postgres
(1 row)
postgres=# select user;
   user   
----------
 postgres
(1 row)
postgres=# SELECT session_user, current_user;
 session_user | current_user 
--------------+--------------
 postgres     | postgres
```

连接信息

```
postgres=# \conninfo
You are connected to database "postgres" as user "postgres" via socket in "/tmp" at port "5432".
```



查询当前用户信息

```

```

# 一、入门教程

## 1.1 数据库连接

```
CREATE DATABASE dbname;
```

## 1.2  选择数据库

```
# 数据库列表
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)
# 选择数据库
postgres=# \c postgres
You are now connected to database "postgres" as user "postgres".
```

## 1.3 删除数据库

```
DROP DATABASE [ IF EXISTS ] name
```

## 1.4 创建表

```
CREATE TABLE table_name(
   column1 datatype,
   column2 datatype,
   column3 datatype,
   .....
   columnN datatype,
   PRIMARY KEY( 一个或多个列 )
);
```

```
CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL
);
```

删除表

```
DROP TABLE table_name;
```

## 1.5 PostgreSQL 模式（SCHEMA）

PostgreSQL 模式（SCHEMA）可以看着是一个表的集合。一个模式可以包含视图、索引、数据类型、函数和操作符等。

连接到 runoobdb 来创建模式 myschema：

```
runoobdb=# create schema myschema;
CREATE SCHEMA
```

接下来我们再创建一个表格：

```
runoobdb=# create table myschema.company(
   ID   INT              NOT NULL,
   NAME VARCHAR (20)     NOT NULL,
   AGE  INT              NOT NULL,
   ADDRESS  CHAR (25),
   SALARY   DECIMAL (18, 2),
   PRIMARY KEY (ID)
);
```

以上命令创建了一个空的表格，我们使用以下 SQL 来查看表格是否创建：

```
runoobdb=# select * from myschema.company;
 id | name | age | address | salary 
----+------+-----+---------+--------
(0 rows)
```

删除模式

删除一个为空的模式（其中的所有对象已经被删除）：

```
DROP SCHEMA myschema;
```

删除一个模式以及其中包含的所有对象：

```
DROP SCHEMA myschema CASCADE;
```

## 1.6 添加数据

```
runoobdb=# INSERT INTO COMPANY (ID,NAME,AGE,ADDRESS,SALARY,JOIN_DATE) VALUES (1, 'Paul', 32, 'California', 20000.00,'2001-07-13');
```

## 1.7 查询数据

```
SELECT column1, column2,...columnN FROM table_name;
```

```
SELECT * FROM table_name;
```





推荐教程：

[PostgreSQL 索引 | 菜鸟教程 (runoob.com)](https://www.runoob.com/postgresql/postgresql-index.html)

