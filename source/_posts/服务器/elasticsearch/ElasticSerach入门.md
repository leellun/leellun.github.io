---
title: elasticsearch基本操作
date: 2019-03-25 18:14:02
categories:
  - 服务器
  - elasticsearch
tags:
  - elasticsearch
---

# 一、HTTP操作

## 1.1 HTTP请求方法

| 请求方法 | 说明                                                         | 开始版本  |
| -------- | ------------------------------------------------------------ | --------- |
| GET      | 请求会`显示`请求指定的资源。一般来说`GET`方法应该只用于数据的读取，而不应当用于会产生副作用的`非幂等`的操作中。 | HTTP/0.9  |
| POST     | 向指定资源提交数据，请求服务器进行处理，请求数据会被包含在请求体中。`POST`方法是`非幂等`的方法，因为这个请求可能会创建新的资源或修改现有资源。 | HTTP/1.0  |
| HEAD     | 与`GET`方法一样，都是向服务器发出指定资源的请求。但是`HEAD`请求只获取服务器的响应头信息 | HTTP/1.0  |
| OPTIONS  | 请求服务器回显其收到的请求信息，该方法主要用于HTTP请求的测试或诊断。 | HTTP/1.1  |
| PUT      | 请求会身向指定资源位置上传其最新内容，`PUT`方法是`幂等`的方法。 | HTTP/1.1  |
| DELETE   | 请求用于请求服务器删除所请求`URI`                            | HTTP/1.1  |
| TRACE    | 与`HEAD`类似，一般也是用于客户端查看服务器的性能。           | HTTP/1.1  |
| CONNECT  | 能够将连接改为管道方式的代理服务器。                         | HTTP/1.1  |
| PATCH    | `PATCH`请求与`PUT`请求类似，1 但`PATCH`一般用于资源的部分更新，而`PUT`一般用于资源的整体更新。 2 当资源不存在时，`PATCH`会创建一个新的资源，而`PUT`只会对已在资源进行更新。 | HTTP/1.1+ |

## 1.2 ElasticSearch请求操作

| 请求方法 | 描述                                       |
| -------- | ------------------------------------------ |
| PUT      | 创建索引、创建映射                         |
| GET      | 查看索引、查看文档、查看映射               |
| DELETE   | 删除索引、删除文档                         |
| POST     | 创建文档、修改文档、修改字段、条件删除文档 |

# 二、索引

## 2.1 创建索引 

```
PUT http://localhost:9200/my_index?pretty
响应结果
{
    "acknowledged": true,//响应结果 true操作成功
    "shards_acknowledged": true,//分片结果,分片操作成功
    "index": "my_index"//索引名称
}
注意：创建索引库的分片数默认 1 片，在 7.0.0 之前的 Elasticsearch 版本中，默认 5 片
```

重复PUT添加索引会报错already exists

#### Text文本类型

- analyzer

  通过analyzer属性指定分词器。

- index

  index属性指定是否索引。

#### keyword关键字

目前已经取代了"index": false。上边介绍的text文本字段在映射时要设置分词器，keyword字段为关键字字段，通常搜索keyword是按照整体搜索，所以创建keyword字段的索引时是不进行分词的，比如：邮政编码、手机号码、身份证等。keyword字段通常用于过虑、排序、聚合等。

#### date日期类型

日期类型不用设置分词器。

通常日期类型的字段用于排序。

format

通过format设置日期格式

下边的设置允许date字段存储年月日时分秒、年月日及毫秒三种格式。 

```
{
    "properties": {
        "timestamp": {
            "type":   "date",
            "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd"
        }
    }
}
```

#### 数值类型

ES支持以下几种数值类型

long、integer、short、byte、double、float、half_float、scaled_float

## 2.2 查看索引

```
GET http://127.0.0.1:9200/_cat/indices?v
响应结果
health status index            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .geoip_databases 94WlVXjcTfKD4vm-U1NgVw   1   0         40            0     38.3mb         38.3mb
yellow open   my_index         hy9Y15AZRbKRFqNh8Lz9Ag   1   1          0            0       226b           226b
```

字段含义：

| 表头           | 含义                                                         |
| -------------- | ------------------------------------------------------------ |
| health         | 当前服务器健康状态：green(集群完整 ) yellow 单点正常、集群不完整 ) red 单点不正常 |
| status         | 索引打开、关闭状态                                           |
| index          | 索引名                                                       |
| uuid           | 索引统一编号                                                 |
| pri            | 主分片数量                                                   |
| rep            | 副本数量                                                     |
| docs.count     | 可用文档数量                                                 |
| docs.deleted   | 文档删除状态（逻辑删除）                                     |
| store.size     | 主分片和副分片整体占空间大小                                 |
| pri.store.size | 主分片占空间大小                                             |

## 2.3 查看指定索引

```
GET http://localhost:9200/my_index
响应结果
{
    "my_index"【索引名】: {
        "aliases"【别名】: {},
        "mappings"【映射】: {},
        "settings"【设置】: {
            "index"【设置-索引】: {
                "routing"【路由】: {
                    "allocation": {
                        "include": {
                            "_tier_preference": "data_content"
                        }
                    }
                },
                "number_of_shards"【设置-索引-主分片数量】: "1",
                "provided_name"【设置-索引-名称】: "my_index",
                "creation_date"【设置-索引-创建时间】: "1625634858360",
                "number_of_replicas"【设置-索引-副分片数量】: "1",
                "uuid"【设置-索引-唯一标志】: "hEFnoYJITayhJkQpcdg6LQ",
                "version"【版本】: {
                    "created": "7130299"
                }
            }
        }
    }
}
```

## 2.4 删除索引

```
DELETE http://localhost:9200/my_index
响应结果
{
    "acknowledged": true
}
```

# 三、文档操作

## 3.1 创建文档

```
POST http://localhost:9200/my_index/_doc
请求体
{
   "title":"小米电视",
   "category":"小米",
   "images":"http://www.xxx.com/img.jpg",
   "price":3999.00
}

响应结果
{
    "_index": "my_index",
    "_type"[类型]: "_doc",
    "_id": "zM58f3oBqPFmLfO0eC3l",
    "_version": 1,
    "result"【结果 created表示创建成功】: "created",
    "_shards": {
        "total"【分片总数】: 2,
        "successful"【成功数量】: 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}

通过
POST http://127.0.0.1:9200/my_index/_doc/1 指定唯一标识
可以返回 "_id":"1"
```

## 3.2 查看文档

```
GET http://127.0.0.1:9200/my_index/_doc/zM58f3oBqPFmLfO0eC3l

响应结果
{
    "_index"【索引】: "my_index",
    "_type"【文档类型】: "_doc",
    "_id": "zM58f3oBqPFmLfO0eC3l",
    "_version"【版本号，表示该数据被修改了多少次】: 1,
    "_seq_no"【递增的序列号，保证每次操作都比前一次大】: 0,
    "_primary_term"【当分片重新选举时，+1】: 1,
    "found": true,
    "_source"【文档源信息】: {
        "title": "小米电视",
        "category": "小米",
        "images": "http://www.xxx.com/img.jpg",
        "price": 3999.00
    }
}
```

- _version：文档版本，每次更新文档时递增。
- _seq_no：配置给文档的序列号用来进行索引操作，用于确保文档的旧版本不会覆盖新版本。
- _primary_term：配置给文档的主term用来进行索引操作。

## 3.3 修改文档

POST  /index/type/id/_update

```
POST http://127.0.0.1:9200/my_index/_doc/zM58f3oBqPFmLfO0eC3l
请求体【这里可以修改一条数据的局部信息】
{
   "title":"小米电视23",
   "category":"小米",
   "images":"http://www.xxx.com/img.jpg",
   "price":3999.00
}
响应结果
{
    "_index": "my_index",
    "_type": "_doc",
    "_id": "zM58f3oBqPFmLfO0eC3l",
    "_version": 2,
    "result": "updated"【修改状态】,
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 1,
    "_primary_term": 1
}
```

## 3.4 删除文档

```
DELETE http://127.0.0.1:9200/my_index/_doc/zM58f3oBqPFmLfO0eC3l
响应结果
{
    "_index": "my_index",
    "_type": "_doc",
    "_id": "zM58f3oBqPFmLfO0eC3l",
    "_version": 3,
    "result": "deleted",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 2,
    "_primary_term": 1
}
```

## 3.5 条件删除文档

```
POST http://127.0.0.1:9200/my_index/_delete_by_query
请求体：
{
    "query":{
        "match":{
            "_id":1
        }
    }
}
响应结果
{
    "took"【耗时】: 243,
    "timed_out"【是否超时】: false,
    "total"【总数】: 1,
    "deleted"【删除数量】: 1,
    "batches": 1,
    "version_conflicts": 0,
    "noops": 0,
    "retries": {
        "bulk": 0,
        "search": 0
    },
    "throttled_millis": 0,
    "requests_per_second": -1.0,
    "throttled_until_millis": 0,
    "failures": []
}
```

# 四、映射操作

## 4.1 创建映射

index：true索引映射，false普通字段

```
PUT http://127.0.0.1:9200/my_index/_mapping
请求体
{
    "properties":{
        "title":{
            "type":"text",
            "index":true
        },
        "category":{
            "type":"text",
            "index":true
        },
        "images":{
            "type":"text",
            "index":true
        },
        "price":{
            "type":"float",
            "index":true
        }
    }
}
响应结果
{
    "acknowledged": true
}
```

## 4.2 查看映射

```
GET http://127.0.0.1:9200/my_index/_mapping

结果
{
    "my_index": {
        "mappings": {
            "properties": {
                "category": {
                    "type": "text",
                    "fields": {
                        "keyword": {
                            "type": "keyword",
                            "ignore_above": 256
                        }
                    }
                },
                "images": {
                    "type": "text",
                    "fields": {
                        "keyword": {
                            "type": "keyword",
                            "ignore_above": 256
                        }
                    }
                },
                "price": {
                    "type": "float"
                },
                "title": {
                    "type": "text",
                    "fields": {
                        "keyword": {
                            "type": "keyword",
                            "ignore_above": 256
                        }
                    }
                }
            }
        }
    }
}
```

## 4.3 索引创建并同时关联

```
PUT http://192.168.10.103:9200/my_index2
请求体
{
    "settings": {},
    "mappings": {
        "properties": {
            "title": {
                "type": "text",
                "index": true
            },
            "category": {
                "type": "text",
                "index": true
            },
            "images": {
                "type": "text",
                "index": false
            },
            "price": {
                "type": "double",
                "index": false
            }
        }
    }
}

结果
{
    "acknowledged": true,
    "shards_acknowledged": true,
    "index": "my_index2"
}
```

# 五、高级查询

## 5.1 查询所有

```
GET http://127.0.0.1:9200/my_index/_search

结果
{
    "took": 1,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 1.0,
                "_source": {
                    "title": "小米电视",
                    "category": "小米",
                    "images": "http://www.xxx.com/img.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

took：执行毫秒

timed_out：是否超时

_shards：到几个分片搜索，成功几个，跳过几个，失败几个。 

total：查询总数

max_score：相关度。越相关分数越高

hits：查询命中的document

## 5.2 字段匹配查询

```
GET http://127.0.0.1:9200/my_index/_search
{
    "query":{
        "match":{
            "title":"小米电视"
        }
    }
}
结果：
{
    "took": 13,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.1507283,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 1.1507283,
                "_source": {
                    "title": "小米电视",
                    "category": "小米",
                    "images": "http://www.xxx.com/img.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

## 5.3 多字段匹配查询

```
GET http://127.0.0.1:9200/my_index/_search
{
    "query": {
        "multi_match": {
            "query": "小米",
            "fields": [//指定从字段title和category中查找
                "title",
                "category"
            ]
        }
    }
}
结果
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.5753642,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 0.5753642,
                "_source": {
                    "title": "小米电视",
                    "category": "小米",
                    "images": "http://www.xxx.com/img.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

## 5.4 关键字精确查询

因为category为text类型，会进行分词所以查不出来"小米"

```
GET http://127.0.0.1:9200/my_index/_search
{
    "query": {
        "term": {
            "category": {
                "value": "小"
            }
        }
    }
}
```

## 5.5 多关键字

```
{
    "query": {
        "terms": {
            "category": ["小","米"]
        }
    }
}
```

## 5.6 指定查询字段

```
{
    "_source":["title","category"],
    "query": {
        "terms": {
            "title": ["米"]
        }
    }
}
返回：
{
    "took": 10,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 1.0,
                "_source": {//返回指定字段
                    "title": "小米电视",
                    "category": "小米"
                }
            }
        ]
    }
}
```

## 5.7 过滤字段

- includes ：来指定想要显示的字段

- excludes ：来指定不想要显示的字段

```
{
    "_source":{
        "includes":["title","category"]
    },
    "query": {
        "terms": {
            "category": ["小"]
        }
    }
}
结果：
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 1.0,
                "_source": {
                    "title": "小米电视",
                    "category": "小米"
                }
            }
        ]
    }
}
```

## 5.8 组合查询

bool把各种其它查询通过 must（必须 ）、 must_ not（必须不）、  should（应该）的方式进行组合

```
{
    "query": {
        "bool":{
            "must":{
                "match":{
                    "title":"小米"
                }
            },
            "must_not":{
                "match":{
                    "category":"魅族"
                }
            },
            "should":[
                {
                    "match":{
                        "price": 3999.00
                    }
                }
            ]
        }
    }
}
结果：
{
    "took": 8,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.3646431,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 1.3646431,
                "_source": {
                    "title": "小米电视",
                    "category": "小米",
                    "images": "http://www.xxx.com/img.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

## 5.9 范围查询

![image-20210707145858849](ElasticSerach入门.assets/image-20210707145858849.png)

```
{
    "query": {
        "range":{
            "price":{
                "gte":3000,
                "lte":4000
            }
        }
    }
}
结果
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 1.0,
                "_source": {
                    "title": "小米电视",
                    "category": "小米",
                    "images": "http://www.xxx.com/img.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

## 5.10 模糊查询

```
{
    "query": {
        "fuzzy":{
            "title":{
                "value":"小之家", 
                "fuzziness":2  //因为通过分词后"小米电视"会有一个“小”的关键词，所以通过“小” 去掉2个字符就能匹配
            }
        }
    }
}
“fuziness”属性。它可以被设置为“0”， “1”， “2”或“auto”。“auto”是推荐的选项，它会根据查询词的长度定义距离。
编辑距离是将一个术语转换为另一个术语所需的一个字符更改的次数。 这些更改可以包括：

更改字符（box→fox）
删除字符（black→lack）
插入字符（sic→sick）
转置两个相邻字符（act→cat）
为了找到相似的词，模糊查询会在指定的编辑距离内创建搜索词的所有可能变化或扩展的集合。 查询然后返回每个扩展的完全匹配。
```

返回包含与搜索词类似的词的文档，该词由Levenshtein编辑距离度量。

* value：查询的关键字

* boost：查询的权值，默认值是1.0

* min_similarity：设置匹配的最小相似度，默认值0.5，对于字符串，取值0-1(包括0和1)；对于数值，取值可能大于1；对于日期取值为1d,1m等，1d等于1天

* prefix_length：指明区分词项的共同前缀长度，默认是0

包括以下几种情况：

- 更改角色（big→pig）

- 删除字符（good→god）

- 插入字符（god→good）

- 调换两个相邻字符（god→dog） 

## 5.11 排序

```
{
    "query": {
        "fuzzy":{
            "title":{
                "value":"小米之",
                "fuzziness":2
            }
        }
    },
    "sort":{
        "price":{
            "order":"desc"
        }
    }
}
```

## 5.12 前缀查询

```
匹配包含具有指定前缀的项（not analyzed）的字段的文档。前缀查询对应 Lucene 的 PrefixQuery 。
{
    "query": {
        "prefix": {
            "title": {
                "value": "小",
                "boost": 2.0
            }
        }
    }
}
```

## 5.13 正则表达式查询

regexp （正则表达式）查询允许您使用正则表达式进行项查询。

```
{
    "query": {
        "regexp":{
            "title":{
                "value":"小.*",
                "boost":1.2
            }
        }
    }
}
结果：
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1.2,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 1.2,
                "_source": {
                    "title": "小米电视",
                    "category": "小米",
                    "images": "http://www.xxx.com/img.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

## 5.14 通配符查询

匹配与通配符表达式具有匹配字段的文档（**not analyzed**）。支持的通配符是 “*”，它匹配任何字符序列（包括空字符）；还有 “？”，它匹配任何单个字符。

为了防止极慢的通配符查询，通配符项不应以通配符 “*” 或 “？” 开头。

```
{
    "query": {
        "wildcard": {
            "title": {
                "value": "小*",
                "boost": 2.0
            }
        }
    }
}
结果：
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 2.0,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 2.0,
                "_source": {
                    "title": "小米电视",
                    "category": "小米",
                    "images": "http://www.xxx.com/img.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```

5.15 批量查询

```
http://127.0.0.1:9200/_mget
{
   "docs" : [
      {
         "_index" : "my_index",
         "_type" :  "_doc",
         "_id" :    "1c2E8oIBA-cIUcUQNoEC"
      },
      {
         "_index" : "my_index",
         "_type" :  "_doc",
         "_id" :    "18318oIBA-cIUcUQd4G5",
         "_source": "title"
      }
   ]
}
结果：
{
    "docs": [
        {
            "_index": "my_index",
            "_type": "_doc",
            "_id": "1c2E8oIBA-cIUcUQNoEC",
            "_version": 1,
            "_seq_no": 5,
            "_primary_term": 1,
            "found": true,
            "_source": {
                "title": "小米电视",
                "category": "小米",
                "images": "http://www.xxx.com/img.jpg",
                "price": 3999.00
            }
        },
        {
            "_index": "my_index",
            "_type": "_doc",
            "_id": "18318oIBA-cIUcUQd4G5",
            "_version": 1,
            "_seq_no": 8,
            "_primary_term": 1,
            "found": true,
            "_source": {
                "title": "乐视电视"
            }
        }
    ]
}
```

## 5.15 批量增删改

`_bulk` 操作将文档的增删改查一系列操作，通过以此请求全部做完，减少网络传输次数

```
http://127.0.0.1:9200/_bulk
{"delete": {"_index": "article", "_id": 6}}
{"create": {"_index": "article", "_id": 7}}
{"title": "我是批量操作中创建的数据，ID是7 "}
{"update": {"_index": "article", "_id": 5}}
{"doc": {"content": "我在批量操作中进行了修改，但是PHP依然是最好的语言"}}

```

总结：

-  delete：删除一个文档，只要1个json串就可以了
-  create：相当于强制创建  PUT /index/type/id/_create 
-  index：普通的put操作，可以是创建文档，也可以是全量替换文档
-  update：执行的是局部更新partial update操作
-  使用json格式发送数据（postman中报错不要紧），最后一行也需要换行 

结果：(因为我这里没有这些索引)

```
{
    "took": 559,
    "errors": true,
    "items": [
        {
            "delete": {
                "_index": "article",
                "_type": "_doc",
                "_id": "6",
                "_version": 1,
                "result": "not_found",
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 0,
                "_primary_term": 1,
                "status": 404
            }
        },
        {
            "create": {
                "_index": "article",
                "_type": "_doc",
                "_id": "7",
                "_version": 1,
                "result": "created",
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 1,
                "_primary_term": 1,
                "status": 201
            }
        },
        {
            "update": {
                "_index": "article",
                "_type": "_doc",
                "_id": "5",
                "status": 404,
                "error": {
                    "type": "document_missing_exception",
                    "reason": "[_doc][5]: document missing",
                    "index_uuid": "gFoQVQArTMWaE2eir-pnlw",
                    "shard": "0",
                    "index": "article"
                }
            }
        }
    ]
}
```

## 5.16 分页查询

```
http://127.0.0.1:9200/my_index/_search?size=5&from=10

或者
{
    "query": {
        "range":{
            "price":{
                "gte":3000,
                "lte":4000
            }
        }
    },
    "from":10,
    "size":5
}
```

## 5.17 查询计划

explain可以用来反映本次查询的一些信息，也可以用来定位查询错误

```
http://127.0.0.1:9200/my_index/_validate/query?explain
{
    "query": {
        "range":{
            "price1":{
                "gte":3000,
                "lte":4000
            }
        }
    }
}
结果：
{
    "_shards": {
        "total": 1,
        "successful": 1,
        "failed": 0
    },
    "valid": true,
    "explanations": [
        {
            "index": "my_index",
            "valid": true,
            "explanation": "MatchNoDocsQuery(\"User requested \"match_none\" query.\")"
        }
    ]
}


参数：
{
    "query": {
        "range":{
            "price":{
                "gte":3000,
                "lte":4000
            }
        }
    }
}
正确结果：
{
    "_shards": {
        "total": 1,
        "successful": 1,
        "failed": 0
    },
    "valid": true,
    "explanations": [
        {
            "index": "my_index",
            "valid": true,
            "explanation": "price:[3000.0 TO 4000.0]"
        }
    ]
}
```

## 5.18 滚动搜索

现在有一个需求，把某个索引中1亿条数据下载下来存到数据库中。

如果一次查询出来，极有可能导致内存溢出，因此需要分批查询。除了我们上面说的分页查询以外，还可以使用滚动搜索技术 `scroll`

scroll搜索会在第一次搜索时，保留一个当时的快照，之后只会基于这个快照提供数据，这个时间段如果发生了数据的变更，用户不会感知。

每次发送scroll请求，我们还需要指定一个scroll参数和一个时间窗口。每次请求只要在这个时间窗口内完成就可以了 。

```
http://127.0.0.1:9200/my_index/_search?scroll=1m
{
  "query": {
    "match_all": {}
  },
  "size": 1
}
结果：
{
    "_scroll_id": "FGluY2x1ZGVfY29udGV4dF91dWlkDXF1ZXJ5QW5kRmV0Y2gBFmZwckJQRGI2UmhlNEpJUThYeVVZSlEAAAAAAAAAghZpenB6Mm8yalNXLWRXTEJPUUd0T3lB",
    "took": 4,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "18318oIBA-cIUcUQd4G5",
                "_score": 1.0,
                "_source": {
                    "title": "乐视电视",
                    "category": "乐视",
                    "images": "http://www.xxx.com/img.jpg",
                    "price": 3599.00
                }
            }
        ]
    }
}
```

下次访问：

```
http://127.0.0.1:9200/_search/scroll
{"scroll":"1m","scroll_id":"FGluY2x1ZGVfY29udGV4dF91dWlkDXF1ZXJ5QW5kRmV0Y2gBFmZwckJQRGI2UmhlNEpJUThYeVVZSlEAAAAAAAAAgxZpenB6Mm8yalNXLWRXTEJPUUd0T3lB"}

结果：
{
    "_scroll_id": "FGluY2x1ZGVfY29udGV4dF91dWlkDXF1ZXJ5QW5kRmV0Y2gBFmZwckJQRGI2UmhlNEpJUThYeVVZSlEAAAAAAAAAgxZpenB6Mm8yalNXLWRXTEJPUUd0T3lB",
    "took": 8,
    "timed_out": false,
    "terminated_early": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "my_index",
                "_type": "_doc",
                "_id": "1c2E8oIBA-cIUcUQNoEC",
                "_score": 1.0,
                "_source": {
                    "title": "小米电视",
                    "category": "小米",
                    "images": "http://www.xxx.com/img.jpg",
                    "price": 3999.00
                }
            }
        ]
    }
}
```



# 六、ES的并发

![image-20200912223339251](ElasticSerach入门.assets/20200912223339.png)

(1) 查询指定

```
http://127.0.0.1:9200/my_index/_doc/1c2E8oIBA-cIUcUQNoEC
{
    "_index": "my_index",
    "_type": "_doc",
    "_id": "1c2E8oIBA-cIUcUQNoEC",
    "_version": 1,
    "_seq_no": 5,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "title": "小米电视",
        "category": "小米",
        "images": "http://www.xxx.com/img.jpg",
        "price": 3999.00
    }
}
```

(2) 更新

```
http://127.0.0.1:9200/my_index/_doc/1c2E8oIBA-cIUcUQNoEC?if_seq_no=5&if_primary_term=1
{
   "title":"小米电视",
   "category":"小米",
   "images":"http://www.xxx.com/img.jpg",
   "price":3999.00
}
```

结果：

```
{
    "_index": "my_index",
    "_type": "_doc",
    "_id": "1c2E8oIBA-cIUcUQNoEC",
    "_version": 3,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 10,
    "_primary_term": 1
}
```

(3) 另一个线程更新或者再次更新

```
http://127.0.0.1:9200/my_index/_doc/1c2E8oIBA-cIUcUQNoEC?if_seq_no=5&if_primary_term=1
{
   "title":"小米电视",
   "category":"小米",
   "images":"http://www.xxx.com/img.jpg",
   "price":3999.00
}
```

结果：

```
{
    "error": {
        "root_cause": [
            {
                "type": "version_conflict_engine_exception",
                "reason": "[1c2E8oIBA-cIUcUQNoEC]: version conflict, required seqNo [9], primary term [1]. current document has seqNo [10] and primary term [1]",
                "index_uuid": "BFJ0A1gdR3aUKSAM2m9xNg",
                "shard": "0",
                "index": "my_index"
            }
        ],
        "type": "version_conflict_engine_exception",
        "reason": "[1c2E8oIBA-cIUcUQNoEC]: version conflict, required seqNo [9], primary term [1]. current document has seqNo [10] and primary term [1]",
        "index_uuid": "BFJ0A1gdR3aUKSAM2m9xNg",
        "shard": "0",
        "index": "my_index"
    },
    "status": 409
}
```

# 七、分词器 analyzer

## 7.1 分词器配置

下载地址：https://github.com/medcl/elasticsearch-analysis-ik/releases

下载后解压到ES目录的 `plugins/ik中`

```
http://192.168.66.11:9200/_analyze
{
  "text": "北京市朝阳区人民群众"
}
响应：
{
    "tokens": [
        {
            "token": "北",
            "start_offset": 0,
            "end_offset": 1,
            "type": "<IDEOGRAPHIC>",
            "position": 0
        },
        {
            "token": "京",
            "start_offset": 1,
            "end_offset": 2,
            "type": "<IDEOGRAPHIC>",
            "position": 1
        },
        {
            "token": "市",
            "start_offset": 2,
            "end_offset": 3,
            "type": "<IDEOGRAPHIC>",
            "position": 2
        },
        {
            "token": "朝",
            "start_offset": 3,
            "end_offset": 4,
            "type": "<IDEOGRAPHIC>",
            "position": 3
        },
        {
            "token": "阳",
            "start_offset": 4,
            "end_offset": 5,
            "type": "<IDEOGRAPHIC>",
            "position": 4
        },
        {
            "token": "区",
            "start_offset": 5,
            "end_offset": 6,
            "type": "<IDEOGRAPHIC>",
            "position": 5
        },
        {
            "token": "人",
            "start_offset": 6,
            "end_offset": 7,
            "type": "<IDEOGRAPHIC>",
            "position": 6
        },
        {
            "token": "民",
            "start_offset": 7,
            "end_offset": 8,
            "type": "<IDEOGRAPHIC>",
            "position": 7
        },
        {
            "token": "群",
            "start_offset": 8,
            "end_offset": 9,
            "type": "<IDEOGRAPHIC>",
            "position": 8
        },
        {
            "token": "众",
            "start_offset": 9,
            "end_offset": 10,
            "type": "<IDEOGRAPHIC>",
            "position": 9
        }
    ]
}
```

指定上面下下来的分词器：

```
{
  "analyzer": "ik_max_word",
  "text": "北京市朝阳区人民群众"
}
响应：
{
    "tokens": [
        {
            "token": "北京市",
            "start_offset": 0,
            "end_offset": 3,
            "type": "CN_WORD",
            "position": 0
        },
        {
            "token": "北京",
            "start_offset": 0,
            "end_offset": 2,
            "type": "CN_WORD",
            "position": 1
        },
        {
            "token": "市",
            "start_offset": 2,
            "end_offset": 3,
            "type": "CN_CHAR",
            "position": 2
        },
        {
            "token": "朝阳区",
            "start_offset": 3,
            "end_offset": 6,
            "type": "CN_WORD",
            "position": 3
        },
        {
            "token": "朝阳",
            "start_offset": 3,
            "end_offset": 5,
            "type": "CN_WORD",
            "position": 4
        },
        {
            "token": "区",
            "start_offset": 5,
            "end_offset": 6,
            "type": "CN_CHAR",
            "position": 5
        },
        {
            "token": "人民群众",
            "start_offset": 6,
            "end_offset": 10,
            "type": "CN_WORD",
            "position": 6
        },
        {
            "token": "人民",
            "start_offset": 6,
            "end_offset": 8,
            "type": "CN_WORD",
            "position": 7
        },
        {
            "token": "群众",
            "start_offset": 8,
            "end_offset": 10,
            "type": "CN_WORD",
            "position": 8
        }
    ]
}
```

- ik_max_word: 会将文本做最细粒度的拆分。
- ik_smart: 会做最粗粒度的拆分。

## 7.2 配置文件

ik配置文件地址：plugins/ik/config目录

- IKAnalyzer.cfg.xml：用来配置自定义词库
- main.dic：ik原生内置的中文词库，总共有 **27万多条** ，只要是这些单词，都会被分在一起
- preposition.dic: 介词
- quantifier.dic：放了一些单位相关的词，量词
- suffix.dic：放了一些后缀
- surname.dic：中国的姓氏
- stopword.dic：英文停用词

最重要的就是下面两个配置，我们一般也只关注这两个

1. main.dic：包含了原生的中文词语，会按照这个里面的词语去分词
2. stopword.dic：包含了英文的停用词

停用词在分词的时候会被直接干掉。

## 7.3 自定义词库

每年都会出现一些流行词，比如 鬼畜、空耳、老铁、666、233等，一般不会出现在ik内置的词库中，需要自己进行扩展。

自己补充自己的最新的词语，到ik的词库里面

IKAnalyzer.cfg.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict">custom/mydict.dic;custom/single_word_low_freq.dic</entry>
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords">custom/ext_stopword.dic</entry>
	<!--用户可以在这里配置远程扩展字典 -->
	<!-- <entry key="remote_ext_dict">words_location</entry> -->
	<!--用户可以在这里配置远程扩展停止词字典-->
	<!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>
```

在custom目录下创建mydict.dic文件，输入自定义的词语

补充自己的词语，然后需要重启es，才能生效

自己建立停用词库：比如了，的，啥，么，我们可能并不想去建立索引，让人家搜索

custom/ext_stopword.dic，已经有了常用的中文停用词，可以补充自己的停用词，然后重启es

# 八、聚合操作

就是在查询完某些数据之后，进行group by等操作，并进行一系列的统计。可以参照mysql sql

## 8.1 group by

```
http://192.168.66.11:9200/my_index/_search
{
  "size": 0, # 不查询任何数据，因为聚合操作主要是为了统计，查询数据并没有意义
  "query": {# 查询所有
    "match_all": {} 
  }, 
  "aggs": {
    "group_by_model": { # 相当于给count(*)起一个别名
      "terms": { "field": "price" } #根据哪个字段聚合
    }
  }
}
```

## 8.2 avg

```
http://192.168.66.11:9200/my_index/_search
{
  "size": 0, 
  "query": {
    "match_all": {} 
  }, 
  "aggs": {
    "avg": { 
      "avg": { "field": "price" }
    }
  }
}
```

## 8.3 max

```
http://192.168.66.11:9200/my_index/_search
{
  "size": 0, 
  "query": {
    "match_all": {} 
  }, 
  "aggs": {
    "max": { 
      "max": { "field": "price" }
    }
  }
}
```

## 8.4 min

```
http://192.168.66.11:9200/my_index/_search
{
  "size": 0, 
  "query": {
    "match_all": {} 
  }, 
  "aggs": {
    "group_by_model": { 
      "min": { "field": "price" }
    }
  }
}
```

# 九. 评分机制 TF\IDF

## 9.1 算法介绍

relevance score算法，简单来说，就是计算出，一个索引中的文本，与搜索文本，他们之间的关联匹配程度。

Elasticsearch使用的是 term frequency/inverse document frequency算法，简称为TF/IDF算法。TF词频(Term Frequency)，IDF逆向文件频率(Inverse Document Frequency)

**Term frequency**：搜索文本中的各个词条在field文本中出现了多少次，出现次数越多，就越相关。

![image-20200913001418228](https://ydsmarkdown.oss-cn-beijing.aliyuncs.com/md/20200913001418.png)

举例：搜索请求：hello world

doc1 : hello you and me,and world is very good. 

doc2 : hello,how are you

**Inverse document frequency**：搜索文本中的各个词条在整个索引的所有文档中出现了多少次，出现的次数越多，就越不相关.

![image-20200913001308194](https://ydsmarkdown.oss-cn-beijing.aliyuncs.com/md/20200913001308.png)



举例：搜索请求：hello world

doc1 :  hello ,today is very good

doc2 : hi world ,how are you

整个index中1亿条数据。hello的document 1000个，有world的document  有100个。

doc2 更相关

**Field-length norm**：field长度，field越长，相关度越弱

举例：搜索请求：hello world

doc1 : {"title":"hello article","content ":"balabalabal 1万个"}

doc2 : {"title":"my article","content ":"balabalabal 1万个,world"}

