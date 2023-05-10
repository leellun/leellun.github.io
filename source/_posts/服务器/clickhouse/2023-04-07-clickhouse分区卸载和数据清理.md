---
title: clickhouse分区卸载和数据清理
date: 2023-04-07 12:38:02
categories:
  - 服务器
  - clickhouse
tags:
  - clickhouse
author: leellun
---



```
clickhouse-client -m
```



```
select * from system.storage_policies;
```

获取分区信息

```
 SELECT
     partition AS `分区`,
     sum(rows) AS `总行数`,
     formatReadableSize(sum(data_uncompressed_bytes)) AS `原始大小`,
     formatReadableSize(sum(data_compressed_bytes)) AS `压缩大小`,
     round((sum(data_compressed_bytes) / sum(data_uncompressed_bytes)) * 100, 0) AS `压缩率`
 FROM system.parts
 WHERE (database IN ('log')) AND (table IN ('cw_dtlog')) 
 GROUP BY partition
 ORDER BY partition ASC;
```

表存储策略

```
SELECT
name,
data_paths,
metadata_path,
storage_policy
FROM system.tables where storage_policy='moving_from_ssd_to_hdd';
```

```
ALTER TABLE…DROP PARTITION命令用于删除分区和存储在这个分区上的数据。
```

、卸载与装载分区

卸载：通过DETACH语句卸载分区，物理数据并没有删除，被转移到当前数据表目录的detached子目录下。卸载后代表它已经脱离了CK的管理，CK不会主动清理这些文件，除非用户主动删除或者重新装载

装载：ATTACH，卸载的反向操作。

```
alter table tb_name DETACH | ATTCH partition partition_expr 
```



卸载分区

```
alter table log.cw_dtlog DETACH  partition_expr ='20230507'
```





查看分区在热盘还是冷盘

```
SELECT partition, name, path, is_frozen FROM system.parts WHERE database = 'log' AND table = 'cw_dtlog';
```

这里的 partition 是指 ClickHouse 中表的分区，用来将表中的数据进行逻辑上的划分，可以将数据按照时间、地域等规则划分到不同的分区中。在 SQL 查询中，partition 通常表示分区的名称或者标识符，用来在表中定位和操作特定的分区。在 SELECT partition, name, path, is_frozen FROM system.parts WHERE database = 'log' AND table = 'cw_dtlog' 中，partition 用来表示 cw_dtlog 表的所有分区的名称。







在 ClickHouse 中卸载分区的操作可以使用 `DETACH` 语句，语法如下：

```
DETACH [PART|PARTITION] partition_name [FROM] [db.]table_name

```

其中，`partition_name` 指定要卸载的分区名，可以是单个分区，也可以是多个分区的列表。如果省略 `PART` 或 `PARTITION` 关键字，则 ClickHouse 会自动判断传入的名称是分区名还是表名。

卸载分区后，该分区的数据文件并不会立即删除，而是会被移动到 ClickHouse 配置文件中指定的数据目录下的 `shadow` 目录中。此时可以通过 `DROP PARTITION` 命令来彻底删除该分区的数据文件。

删除分区文件的操作可以使用以下语句：

```
DROP PARTITION partition_name FROM [db.]table_name

```

其中，`partition_name` 指定要删除的分区名，`[db.]table_name` 指定要删除分区的表名。在删除分区文件之前，需要先使用 `DETACH` 命令将分区从表中卸载。