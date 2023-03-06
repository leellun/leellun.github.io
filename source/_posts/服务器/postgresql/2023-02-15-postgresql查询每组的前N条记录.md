---
title: postgresql查询每组的前N条记录
date: 2023-02-15 11:14:02
categories:
  - 服务器
  - postgresql
tags:
  - postgresql
---

方式

```
ROW_NUMBER() OVER (PARTITION BY "字段1","字段2..."  ORDER BY 排序字段1 desc ,排序字段2 DESC) AS row_id
```

例如：

```
select * from (select *,ROW_NUMBER() OVER (PARTITION BY t.src_ip, t.event_name, t.threats ) AS row_id FROM t_log t) t  where  t.row_id <=5
```

解析：

这里查询的是t_log表的整张数据，通过  t.src_ip, t.event_name, t.threats  分组；

同时会生成一个分组的row_id，通过t.row_id <=5可以限制每个分组5条数据；

查询结果