---
title: rocketmq acl权限控制
date: 2022-05-24 18:14:02
categories:
  - 服务器
  - mq
tags:
  - rocketmq
---

## 1 介绍

acl权限控制主要为rabbitmq提供topic资源级别的访问控制。Broker端对AccessKey所拥有的权限进行校验，校验不过，抛出异常；权限控制属性包括Topic访问权限、IP白名单和AccessKey和SecretKey签名等。

## 2 权限控制定义

| 权限 | 含义              |
| ---- | ----------------- |
| DENY | 拒绝              |
| ANY  | PUB 或者 SUB 权限 |
| PUB  | 发送权限          |
| SUB  | 订阅权限          |

## 3 相关属性

| 字段                       | 取值                         | 含义                    |
| -------------------------- | ---------------------------- | ----------------------- |
| globalWhiteRemoteAddresses | \*;192.168.\*.\*;192.168.0.1 | 全局IP白名单            |
| accessKey                  | 字符串                       | Access Key              |
| secretKey                  | 字符串                       | Secret Key              |
| whiteRemoteAddress         | \*;192.168.\*.\*;192.168.0.1 | 用户IP白名单            |
| admin                      | true;false                   | 是否管理员账户          |
| defaultTopicPerm           | DENY;PUB;SUB;PUB\|SUB        | 默认的Topic权限         |
| defaultGroupPerm           | DENY;PUB;SUB;PUB\|SUB        | 默认的ConsumerGroup权限 |
| topicPerms                 | topic=权限                   | 各个Topic的权限         |
| groupPerms                 | group=权限                   | 各个ConsumerGroup的权限 |

更多：conf/plain_acl.yml

```yaml
globalWhiteRemoteAddresses:
  - 10.10.103.*
  - 192.168.66.*

accounts:
  - accessKey: RocketMQ
    secretKey: 12345678
    whiteRemoteAddress:
    admin: false
    defaultTopicPerm: DENY
    defaultGroupPerm: SUB
    topicPerms:
      - topicA=DENY
      - topicB=PUB|SUB
      - topicC=SUB
    groupPerms:
      # the group should convert to retry topic
      - groupA=DENY
      - groupB=PUB|SUB
      - groupC=SUB

  - accessKey: rocketmq2
    secretKey: 12345678
    whiteRemoteAddress: 192.168.1.*
    # if it is admin, it could access all resources
    admin: true	
```

## 4 权限控制的集群部署

```
#acl权限开启
aclEnable=true
```

```
public class RocketUtils {
    public static final String NAMESRVADDR="192.168.66.70:8876;192.168.66.70:8878;192.168.66.70:8879";
    // acl用户名和密码
    public static final String ACL_ACCESS_KEY = "RocketMQ";
    public static final String ACL_SECRET_KEY = "12345678";
    /**
     * 将账号密码 在发送消息的时候携带过去
     *
     * @return
     */
    public static RPCHook getAclRPCHook() {
        return new AclClientRPCHook(new SessionCredentials(ACL_ACCESS_KEY, ACL_SECRET_KEY));
    }
}
// 实例化消息生产者Producer
DefaultMQProducer producer = new DefaultMQProducer(CONSUMER_GROUP1, RocketUtils.getAclRPCHook());
```

## 更多源码：

https://github.com/leellun/rocketmq-learn.git



















