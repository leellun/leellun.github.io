---
title: Beego框架app.conf配置
date: 2024-01-16 20:18:02
categories:
  - 后端
  - go
tags:
  - beego
---

# Beego框架 app.conf配置

beego 目前支持 INI、XML、JSON、YAML格式的配置文件解析，但是默认采用了 INI 格式解析，用户可以通过简单的配置就可以获得很大的灵活性。

beego 默认会解析当前应用下的 conf/app.conf 文件。

BConfig 就是 beego 里面的默认的配置，你也可以直接通过beego.BConfig.AppName="beepkg"这样来修改，和上面的配置效果一样，只是一个在代码里面写死了，而配置文件就会显得更加灵活。

| 配置参数               | 说明                                                         | 默认值      |
| ---------------------- | ------------------------------------------------------------ | ----------- |
| appname                | 应用名                                                       | beego       |
| runMode                | 应用的运行模式 可选值为 prod, dev 或者 test. 默认是 dev,     | dev         |
| httpaddr               | 应用监听地址，默认为空，监听所有的网卡 IP                    | -           |
| httpport               | 应用监听端口                                                 | 8080        |
| autorender             | 是否模板自动渲染，默认值为 true，对于 API 类型的应用，应用需要把该选项设置为 false，不需要渲染模板。 | true        |
| recoverpanic           | 是否异常恢复，默认值为 true，即当应用出现异常的情况，通过 recover 恢复回来，而不会导致应用异常退出。 | true        |
| viewspath              | 模板路径，默认值是 views。                                   | views       |
| routerCaseSensitive    | 是否路由忽略大小写匹配，默认是 true，区分大小写              | true        |
| serverName             | beego 服务器默认在请求的时候输出 server 为 beego。           | beego       |
| copyRequestBody        | 是否允许在 HTTP 请求时，返回原始请求体数据字节，默认为 false （GET or HEAD or 上传文件请求除外）。 | false       |
| enableGzip             | 是否开启 gzip 支持，默认为 false 不支持 gzip，一旦开启了 gzip，那么在模板输出的内容会进行 gzip 或者 zlib 压缩，根据用户的 Accept-Encoding 来判断。 | false       |
| maxMemory              | 文件上传默认内存缓存大小，默认值是 1 << 26(64M)。            | 64M         |
| enableErrorsShow       | 是否显示系统错误信息，默认为 true。                          | true        |
| enableErrorsRender     | 是否将错误信息进行渲染，默认值为 true，即出错会提示友好的出错页面，对于 API 类型的应用可能需要将该选项设置为 false 以阻止在 dev 模式下不必要的模板渲染信息返回。 | true        |
| enableDocs             | 是否开启文档内置功能，默认是 false                           | false       |
| flashName              | Flash 数据设置 Cookie 的名称，默认是 BEEGO_FLASH             | BEEGO_FLASH |
| flashSeperator         | Flash 数据的分隔符，默认是 BEEGOFLASH                        | BEEGOFLASH  |
| directoryIndex         | 是否开启静态目录的列表显示，默认不显示目录，返回 403 错误。  | false       |
| staticDir              | 静态文件目录设置, 可配置单个或多个目录                       | static      |
| staticExtensionsToGzip | 允许哪些后缀名的静态文件进行 gzip 压缩，默认支持 .css 和 .js |             |
| templateLeft           | 模板左标签，默认值是                                         |             |
| templateRight          | 模板右标签，默认值是                                         |             |
| enableXSRF             | 是否开启 XSRF，默认为 false，不开启。                        | false       |
| xSRFKEY                | XSRF 的 key 信息，默认值是 beegoxsrf。 EnableXSRF＝true 才有效 | beegoxsrf   |
| xSRFExpire             | XSRF 过期时间，默认值是 0，不过期。                          | 0           |

更多配置：[Beego框架：参数配置 - 范斯猫 (fansimao.com)](https://www.fansimao.com/777043.html)

## 多环境配置

```
appname = bee
httpaddr = "127.0.0.1"
httpport = 9090
runmode ="dev"
autorender = false
recoverpanic = false
viewspath = "views"

[dev]
httpport = 8080
[prod]
httpport = 8088
[test]
httpport = 8888
```

读取配置

```
beego.AppConfig.String("appname")
```

读取不同模式下配置参数的方法是“模式::配置参数名”，比如：

```
beego.AppConfig.String("dev::httpport")
```

## 多个配置文件

app.conf:

```
appname = bee
httpaddr = "127.0.0.1"
httpport = 9090

include "app2.conf"
app2.conf

runmode ="dev"
autorender = false
recoverpanic = false
viewspath = "views"
[dev]
httpport = 8080
[test]
httpport = 8888
```

app2.conf:

```
[prod]
httpport = 8088
```

## 支持环境变量配置

```
appname = bee
httpaddr = "127.0.0.1"
httpport = "${ProPort||9090}"

runmode ="dev"
autorender = false
recoverpanic = false
viewspath = "views"
```

带环境参数启动(指定运行模式为dev)

```
# 在 Linux 或 macOS 中
BEEGO_RUNMODE=dev bee run

# 在 Windows 中
set BEEGO_RUNMODE=dev && bee run
```

## 加载自己的配置文件

配置文件路径，默认是应用程序对应的目录下的 conf/app.conf，用户可以在程序代码中加载自己的配置文件

```
beego.LoadAppConfig("ini", "conf/app2.conf")
```

