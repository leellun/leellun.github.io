---
title: Kibana学习教程
date: 2022-07-25 18:14:02
categories:
  - 服务器
  - elasticsearch
tags:
  - kibana
---

# 一、介绍

Kibana是一个针对Elasticsearch的开源分析及可视化平台，用来搜索、查看交互存储在Elasticsearch索引中的数据。使用Kibana，可以通过各种图表进行高级数据分析及展示。

Kibana让海量数据更容易理解。它操作简单，基于浏览器的用户界面可以快速创建仪表板（dashboard）实时显示Elasticsearch查询动态。

设置Kibana非常简单。无需编码或者额外的基础架构，几分钟内就可以完成Kibana安装并启动Elasticsearch索引监测。

# 二、安装

```
wget -c https://artifacts.elastic.co/downloads/kibana/kibana-7.17.5-linux-x86_64.tar.gz
```

# 三、配置

```
vi config/kibana.yml
```

```
server.port: 5601 # 端口
server.host: "192.168.66.11" #配置Kibana的远程访问
#配置es访问地址
elasticsearch.hosts: ["http://192.168.66.11:9200","http://192.168.66.21:9200","http://192.168.66.22:9200"]
# 索引
kibana.index: ".kibana"
#汉化访问
i18n.locale: "zh-CN"
```

```
nohup bin/kinaba &
```

访问 http://localhost:5601 进入Dev Tools界面，就可以操作ES， 并且拥有代码提示

