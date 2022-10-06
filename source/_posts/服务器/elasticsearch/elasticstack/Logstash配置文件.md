# Logstash学习教程

# 一、介绍

logstash是一个数据抽取工具，将数据从一个地方转移到另一个地方。logstash之所以功能强大和流行，还与其丰富的过滤器插件是分不开的，过滤器提供的并不单单是过滤的功能，还可以对进入过滤器的原始数据进行复杂的逻辑处理，甚至添加独特的事件到后续流程中。

Logstash配置文件有如下三部分组成，其中input、output部分是必须配置，filter部分是可选配置，而filter就是过滤器插件，可以在这部分实现各种日志过滤功能。

# 二、安装

```
wget -c https://artifacts.elastic.co/downloads/logstash/logstash-7.17.5-linux-x86_64.tar.gz
```

# 三、开始

logback的 `logstash-sample.conf` 一般不进行配置，而是在config目录下新建一个配置文件进行配置。配置完毕后，通过下面命令指定配置文件启动

```sh
logstash.bat -f ../config/配置文件
```

## 3.1 input输入插件

**读取文件**

logstash使用一个名为filewatch的ruby gem库来监听文件变化,并通过一个叫.sincedb的数据库文件来记录被监听的日志文件的读取进度（时间戳），这个sincedb数据文件的默认路径在 <path.data>/plugins/inputs/file下面，文件名类似于.sincedb_123456，而<path.data>表示logstash插件存储目录，默认是LOGSTASH_HOME/data。

input通常有以下几个配置项，只能选用其中一种

- file：从文件系统中读取一个文件，很像UNIX命令 “tail -0a”
- syslog：监听514端口，按照RFC3164标准解析日志数据
- redis：从redis服务器读取数据，支持channel(发布订阅)和list模式。redis一般在Logstash消费集群中作为"broker"角色，保存events队列共Logstash消费。
- tcp：从网络中获取数据
- stdin：标准输如，从控制台获取
- 其他扩展，如jdbc

```
stdin { } # 从控制台中输入来源


file { # 从文件中来
    path => "/data/*" #单一文件
     #监听文件的多个路径
    path => ["/data/*.log","F:/*.log"]
    #排除不想监听的文件
    exclude => "1.log"
    
    #添加自定义的字段
    add_field => {"test"=>"test"}
    #增加标签
    tags => "tag1"

    #设置新事件的标志
    delimiter => "\n"

    #设置多长时间扫描目录，发现新文件
    discover_interval => 15
    #设置多长时间检测文件是否修改
    stat_interval => 1

     #监听文件的起始位置，默认是end
    start_position => beginning

    #监听文件读取信息记录的位置
    sincedb_path => "\/test.txt"
    #设置多长时间会写入读取的位置信息
    sincedb_write_interval => 15
}


syslog { # 系统日志方式
    type => "system-syslog"  # 定义类型
    port => 10514    # 定义监听端口
}

```

## 3.2 output输出插件

output用于向外输出，一个事件可以经过多个输出，而一旦所有输出处理完成，整个事件就执行完成。 一些常用的输出包括（输出的模式可以选用多种）：

- file：  表示将数据写入磁盘上的文件。
- elasticsearch：表示将数据发送给Elasticsearch。Elasticsearch可以高效方便和易于查询的保存数据。
- stdout：输出到控制台

（1）输出到标准输出(stdout)

```
output {
    stdout {
        codec => rubydebug
    }
}
```

（2）保存为文件（file）

```
output {
    file {
        path => "/data/log/%{+yyyy-MM-dd}/%{host}_%{+HH}.log"
    }
}

```

（3）输出到elasticsearch

```
output {
    elasticsearch {
        hosts => ["192.168.1.1:9200"]
        index => "logstash-%{+YYYY.MM.dd}"       
    }
}

```

- hosts：是一个数组类型的值，后面跟的值是elasticsearch节点的地址与端口，默认端口是9200。可添加多个地址。
- index：写入elasticsearch的索引的名称，这里可以使用变量。Logstash提供了%{+YYYY.MM.dd}这种写法。在语法解析的时候，看到以+ 号开头的，就会自动认为后面是时间格式，尝试用时间格式来解析后续字符串。这种以天为单位分割的写法，可以很容易的删除老的数据或者搜索指定时间范围内的数据。此外，注意索引名中不能有大写字母。

## 3.4 简单示例

（1）控制栏输入输出

```
input {
    stdin { } # 从控制台中输入来源
}

output {
    stdout {
        codec => rubydebug
    }
}
```

结果：

```
234234234234234234
{
    "@timestamp" => 2022-09-01T14:44:04.277Z,
       "message" => "234234234234234234",
      "@version" => "1",
          "host" => "k8s-master01"
}
```

## 3.5 filter过滤器插件

filter是可选配置，用来过滤从input中读取的数据，一般用于日志处理

### Grok 正则捕获

grok是一个十分强大的 `filter` 插件，他可以通过正则解析任意文本，将非结构化日志数据弄成结构化和方便查询的结构。他是目前logstash 中解析非结构化日志数据最好的方式。

Grok 的语法规则是：

```
%{语法: 语义}
```

例如输入的内容为：

```
172.16.213.132 [07/Feb/2019:16:24:19 +0800] "GET / HTTP/1.1" 403 5039
```

%{IP:ip}匹配模式将获得的结果为：ip: 192.168.0.1

%{HTTPDATE:timestamp}匹配模式将获得的结果为：timestamp: 07/Feb/2018:16:24:19 +0800

而%{QS:referrer}匹配模式将获得的结果为：referrer: "GET / HTTP/1.1"

下面是一个组合匹配模式，它可以获取上面输入的所有内容：

```
%{IP:clientip}\ \[%{HTTPDATE:timestamp}\]\ %{QS:referrer}\ %{NUMBER:response}\ %{NUMBER:bytes}
```

通过上面这个组合匹配模式，我们将输入的内容分成了五个部分，即五个字段，将输入内容分割为不同的数据字段，这对于日后解析和查询日志数据非常有用，这正是使用grok的目的。

例子：

```
input{
    stdin{}
}
filter{
    grok{
        match => ["message","%{IP:clientip}\ \[%{HTTPDATE:timestamp}\]\ %{QS:referrer}\ %{NUMBER:response}\ %{NUMBER:bytes}"]
    }
}
output{
    stdout{
        codec => "rubydebug"
    }
}
```

输入输出内容：

```
172.16.213.132 [07/Feb/2019:16:24:19 +0800] "GET / HTTP/1.1" 403 5039
{
          "host" => "k8s-master01",
       "message" => "172.16.213.132 [07/Feb/2019:16:24:19 +0800] \"GET / HTTP/1.1\" 403 5039",
      "@version" => "1",
      "clientip" => "172.16.213.132",
      "referrer" => "\"GET / HTTP/1.1\"",
     "timestamp" => "07/Feb/2019:16:24:19 +0800",
      "response" => "403",
         "bytes" => "5039",
    "@timestamp" => 2022-09-01T14:48:40.999Z
}
```

**grok 内置类型**

```
USERNAME ：[a-zA-Z0-9._-]+
USER ：%{USERNAME}
INT ：(?:[+-]?(?:[0-9]+))
BASE10NUM ：(?<![0-9.+-])(?>[+-]?(?:(?:[0-9]+(?:\.[0-9]+)?)|(?:\.[0-9]+)))
NUMBER ：(?:%{BASE10NUM})
BASE16NUM ：(?<![0-9A-Fa-f])(?:[+-]?(?:0x)?(?:[0-9A-Fa-f]+))
BASE16FLOAT ：\b(?<![0-9A-Fa-f.])(?:[+-]?(?:0x)?(?:(?:[0-9A-Fa-f]+(?:\.[0-9A-Fa-f]*)?)|(?:\.[0-9A-Fa-f]+)))\b

POSINT ：\b(?:[1-9][0-9]*)\b
NONNEGINT ：\b(?:[0-9]+)\b
WORD ：\b\w+\b
NOTSPACE ：\S+
SPACE ：\s*
DATA ：.*?
GREEDYDATA ：.*
QUOTEDSTRING ：(?>(?<!\\)(?>"(?>\\.|[^\\"]+)+"|""|(?>'(?>\\.|[^\\']+)+')|''|(?>`(?>\\.|[^\\`]+)+`)|``))
UUID ：[A-Fa-f0-9]{8}-(?:[A-Fa-f0-9]{4}-){3}[A-Fa-f0-9]{12}

# Networking
MAC ：(?:%{CISCOMAC}|%{WINDOWSMAC}|%{COMMONMAC})
CISCOMAC ：(?:(?:[A-Fa-f0-9]{4}\.){2}[A-Fa-f0-9]{4})
WINDOWSMAC ：(?:(?:[A-Fa-f0-9]{2}-){5}[A-Fa-f0-9]{2})
COMMONMAC ：(?:(?:[A-Fa-f0-9]{2}:){5}[A-Fa-f0-9]{2})
IPV6 ：((([0-9A-Fa-f]{1,4}:){7}([0-9A-Fa-f]{1,4}|:))|(([0-9A-Fa-f]{1,4}:){6}(:[0-9A-Fa-f]{1,4}|((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3})|:))|(([0-9A-Fa-f]{1,4}:){5}(((:[0-9A-Fa-f]{1,4}){1,2})|:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3})|:))|(([0-9A-Fa-f]{1,4}:){4}(((:[0-9A-Fa-f]{1,4}){1,3})|((:[0-9A-Fa-f]{1,4})?:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:))|(([0-9A-Fa-f]{1,4}:){3}(((:[0-9A-Fa-f]{1,4}){1,4})|((:[0-9A-Fa-f]{1,4}){0,2}:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:))|(([0-9A-Fa-f]{1,4}:){2}(((:[0-9A-Fa-f]{1,4}){1,5})|((:[0-9A-Fa-f]{1,4}){0,3}:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:))|(([0-9A-Fa-f]{1,4}:){1}(((:[0-9A-Fa-f]{1,4}){1,6})|((:[0-9A-Fa-f]{1,4}){0,4}:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:))|(:(((:[0-9A-Fa-f]{1,4}){1,7})|((:[0-9A-Fa-f]{1,4}){0,5}:((25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)(\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]?\d)){3}))|:)))(%.+)?
IPV4 (?<![0-9])(?:(?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})[.](?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})[.](?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})[.](?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2}))(?![0-9])
IP ：(?:%{IPV6}|%{IPV4})
HOSTNAME ：\b(?:[0-9A-Za-z][0-9A-Za-z-]{0,62})(?:\.(?:[0-9A-Za-z][0-9A-Za-z-]{0,62}))*(\.?|\b)
HOST ：%{HOSTNAME}
IPORHOST ：(?:%{HOSTNAME}|%{IP})
HOSTPORT ：%{IPORHOST}:%{POSINT}

# paths
PATH ：(?:%{UNIXPATH}|%{WINPATH})
UNIXPATH ：(?>/(?>[\w_%!$@:.,-]+|\\.)*)+
TTY ：(?:/dev/(pts|tty([pq])?)(\w+)?/?(?:[0-9]+))
WINPATH ：(?>[A-Za-z]+:|\\)(?:\\[^\\?*]*)+
URIPROTO ：[A-Za-z]+(\+[A-Za-z+]+)?
URIHOST ：%{IPORHOST}(?::%{POSINT:port})?
# uripath comes loosely from RFC1738, but mostly from what Firefox
# doesn't turn into %XX
URIPATH ：(?:/[A-Za-z0-9$.+!*'(){},~:;=@#%_\-]*)+
#URIPARAM \?(?:[A-Za-z0-9]+(?:=(?:[^&]*))?(?:&(?:[A-Za-z0-9]+(?:=(?:[^&]*))?)?)*)?
URIPARAM ：\?[A-Za-z0-9$.+!*'|(){},~@#%&/=:;_?\-\[\]]*
URIPATHPARAM ：%{URIPATH}(?:%{URIPARAM})?
URI ：%{URIPROTO}://(?:%{USER}(?::[^@]*)?@)?(?:%{URIHOST})?(?:%{URIPATHPARAM})?

# Months: January, Feb, 3, 03, 12, December
MONTH ：\b(?:Jan(?:uary)?|Feb(?:ruary)?|Mar(?:ch)?|Apr(?:il)?|May|Jun(?:e)?|Jul(?:y)?|Aug(?:ust)?|Sep(?:tember)?|Oct(?:ober)?|Nov(?:ember)?|Dec(?:ember)?)\b
MONTHNUM ：(?:0?[1-9]|1[0-2])
MONTHNUM2 ：(?:0[1-9]|1[0-2])
MONTHDAY ：(?:(?:0[1-9])|(?:[12][0-9])|(?:3[01])|[1-9])

# Days: Monday, Tue, Thu, etc...
DAY ：(?:Mon(?:day)?|Tue(?:sday)?|Wed(?:nesday)?|Thu(?:rsday)?|Fri(?:day)?|Sat(?:urday)?|Sun(?:day)?)

# Years?
YEAR ：(?>\d\d){1,2}
HOUR ：(?:2[0123]|[01]?[0-9])
MINUTE ：(?:[0-5][0-9])
# '60' is a leap second in most time standards and thus is valid.
SECOND ：(?:(?:[0-5]?[0-9]|60)(?:[:.,][0-9]+)?)
TIME ：(?!<[0-9])%{HOUR}:%{MINUTE}(?::%{SECOND})(?![0-9])
# datestamp is YYYY/MM/DD-HH:MM:SS.UUUU (or something like it)
DATE_US ：%{MONTHNUM}[/-]%{MONTHDAY}[/-]%{YEAR}
DATE_EU ：%{MONTHDAY}[./-]%{MONTHNUM}[./-]%{YEAR}
ISO8601_TIMEZONE ：(?:Z|[+-]%{HOUR}(?::?%{MINUTE}))
ISO8601_SECOND ：(?:%{SECOND}|60)
TIMESTAMP_ISO8601 ：%{YEAR}-%{MONTHNUM}-%{MONTHDAY}[T ]%{HOUR}:?%{MINUTE}(?::?%{SECOND})?%{ISO8601_TIMEZONE}?
DATE ：%{DATE_US}|%{DATE_EU}
DATESTAMP ：%{DATE}[- ]%{TIME}
TZ ：(?:[PMCE][SD]T|UTC)
DATESTAMP_RFC822 ：%{DAY} %{MONTH} %{MONTHDAY} %{YEAR} %{TIME} %{TZ}
DATESTAMP_RFC2822 ：%{DAY}, %{MONTHDAY} %{MONTH} %{YEAR} %{TIME} %{ISO8601_TIMEZONE}
DATESTAMP_OTHER ：%{DAY} %{MONTH} %{MONTHDAY} %{TIME} %{TZ} %{YEAR}
DATESTAMP_EVENTLOG ：%{YEAR}%{MONTHNUM2}%{MONTHDAY}%{HOUR}%{MINUTE}%{SECOND}

# Syslog Dates: Month Day HH:MM:SS
SYSLOGTIMESTAMP ：%{MONTH} +%{MONTHDAY} %{TIME}
PROG ：(?:[\w._/%-]+)
SYSLOGPROG ：%{PROG:program}(?:\[%{POSINT:pid}\])?
SYSLOGHOST ：%{IPORHOST}
SYSLOGFACILITY ：<%{NONNEGINT:facility}.%{NONNEGINT:priority}>
HTTPDATE ：%{MONTHDAY}/%{MONTH}/%{YEAR}:%{TIME} %{INT}

# Shortcuts
QS ：%{QUOTEDSTRING}

# Log formats
SYSLOGBASE ：%{SYSLOGTIMESTAMP:timestamp} (?:%{SYSLOGFACILITY} )?%{SYSLOGHOST:logsource} %{SYSLOGPROG}:
COMMONAPACHELOG ：%{IPORHOST:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-)
COMBINEDAPACHELOG ：%{COMMONAPACHELOG} %{QS:referrer} %{QS:agent}

# Log Levels
LOGLEVEL ：([Aa]lert|ALERT|[Tt]race|TRACE|[Dd]ebug|DEBUG|[Nn]otice|NOTICE|[Ii]nfo|INFO|[Ww]arn?(?:ing)?|WARN?(?:ING)?|[Ee]rr?(?:or)?|ERR?(?:OR)?|[Cc]rit?(?:ical)?|CRIT?(?:ICAL)?|[Ff]atal|FATAL|[Ss]evere|SEVERE|EMERG(?:ENCY)?|[Ee]merg(?:ency)?)
```

### 时间处理(Date)

date插件是对于排序事件和回填旧数据尤其重要，它可以用来转换日志记录中的时间字段，变成LogStash::Timestamp对象，然后转存到@timestamp字段里。

下面是date插件的一个配置示例（这里仅仅列出filter部分）：

```
filter {
    grok {
        match => ["message", "%{HTTPDATE:timestamp}"] # 提取HTTPDATE赋值给timestamp
    }
    date {
        match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"] # 提取timestamp并转换，然后赋值给target指向的timestamp
        target => "timestamp"
    }
}

```

### 数据修改(Mutate)

- 正则表达式替换匹配字段

gsub可以通过正则表达式替换字段中匹配到的值，只对字符串字段有效，下面是一个关于mutate插件中gsub的示例（仅列出filter部分）：

```
filter {
    mutate {
        gsub => ["filed_name_1", "/" , "_"]
    }
}

```

这个示例表示将filed_name_1字段中所有"/"字符替换为"_"。

- 分隔符分割字符串为数组

split可以通过指定的分隔符分割字段中的字符串为数组，下面是一个关于mutate插件中split的示例（仅列出filter部分）：

```
filter {
    mutate {
        split => ["filed_name_2", "|"]
    }
}

```

这个示例表示将filed_name_2字段以"|"为区间分隔为数组。

- 重命名字段

rename可以实现重命名某个字段的功能，下面是一个关于mutate插件中rename的示例（仅列出filter部分）：

```
filter {
    mutate {
        rename => { "old_field" => "new_field" }
    }
}

```

这个示例表示将字段old_field重命名为new_field。

- 删除字段

remove_field可以实现删除某个字段的功能，下面是一个关于mutate插件中remove_field的示例（仅列出filter部分）：

```
filter {
    mutate {
        remove_field  =>  ["timestamp"]
    }
}

```

这个示例表示将字段timestamp删除。

- GeoIP 地址查询归类

```
filter {
    geoip {
        source => "ip_field"
    }
}

```

- 综合例子：

```
input {
  # stdin {}

  file {
    path => "/home/es/elastic-stack/logstash/example/test.log"
    start_position => beginning
        #添加自定义的字段
    add_field => {"test"=>"test"}
    #增加标签
    stat_interval => 1
  }
}

filter {
  grok {
    # 127.0.0.1 - - [11/Nov/2022:17:10:00 +0800] "GET / HTTP/1.1" 200 2384 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:70.0) Gecko/20100101 Firefox/70.0"
    match => ["message", "%{IP:clientIp}\ \-\ \-\ \[%{HTTPDATE:timestamp}\]\ %{QS:referrer}\ %{NUMBER:response}\ %{NUMBER:bytes}\ %{GREEDYDATA:ua}"]
  }
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "timestamp"
  }
  mutate {
    gsub => ["ua", "\"-\" ", ""]
    gsub => ["ua", "\"", ""]
    gsub => ["ua", "\r", ""]
    gsub => ["referrer", "\"", ""]

    split => ["clientIp", "."]

    rename => ["timestamp", "create_time"]
    remove_field => ["message"]
  }
}

output {
  stdout {
    codec => rubydebug
  }
  # file {
  #   path => "H:/logstash-7.3.0/logdata/%{host}_%{+HH}.log"
  # }
  # elasticsearch {
  #   hosts => ["39.102.41.53:9200"]
  #   index => "nginxlog-%{+YYYY.MM.dd}"
  # }
}

```

# 三 、配置文件

```
# ---------- Node identity ----------

# 节点名称，默认主机名
node.name: test


# ---------- Data path ----------

# 数据存储路径，默认LOGSTASH_HOME/data
path.data:


# ---------- Pipeline Settings ----------

# pipeline ID，默认main
pileline.id: main

# 输出通道的工作workers数据量，默认cpu核心数
pipeline.workers:

# 单个工作线程尝试执行其过滤器和输出之前将从输入手机的最大事件数量，默认125
pipeline.batch.size: 125

# 将较小的批处理分派给管道之前，等待的毫秒数，默认50ms
pipeline.batch.delay: 50

# 此值为true时，即使内存中仍然有运行中事件，也会强制Logstash在关机期间退出
pipeline.unsafe_shutdown: false

# 管道事件排序
# 可选项: auto,true,false,默认auto
pipeline.ordered: auto


# ---------- Pipeline Configuration Settings ----------

# 配置文件路径
path.config:

# 主管道的管道配置字符串
config.string:

# 该值为true时，检查配置是否有效，然后退出，默认false
config.test_and_exit: false

# 该值为true时，会定期检查配置是否已更改，并在更改后重新加载配置，默认false
config.reload.automatic: false

# 检查配置文件更改的时间间隔，默认3s
config.reload.interval: 3s

# 该值为true时，将完整编译的配置显示为调试日志消息，默认为false
config.debug: false

# 该值为true时，开启转移
config.support_escapes: false


# ---------- HTTP API Settings ----------

# 是否开启htp访问，默认true
http.enabled: true

# 绑定主机地址，可以是ip，主机名，默认127.0.0.1
http.host: 127.0.0.1

# 服务监听端口，可以是单个端口，也可以是范围端口，默认9600-9700
http.port: 9600-9700


# ---------- Module Settings ----------

# 模块定义，必须为数组
# 模块变量名格式必须为var.PLUGIN_TYPE.PLUGIN_NAME.KEY
modules:
    - name: MODULE_NAME
        var.PLUGINTYPE1.PLUGINNAME1.KEY1: VALUE
        var.PLUGINTYPE1.PLUGINNAME1.KEY2: VALUE
        var.PLUGINTYPE2.PLUGINNAME1.KEY1: VALUE
        var.PLUGINTYPE3.PLUGINNAME3.KEY1: VALUE


# ---------- Queuing Settings ----------

# 事件缓冲的内部排队模型，可选项：memory，persisted，默认memory
queue.type: memory

# 启用持久队列(queue.type: persisted)后将在其中存储数据文件的目录路径
# 默认path.data/queue
path.queue:

# 启用持久队列(queue.type: persisted)后，队列中未读事件的最大数量
# 默认0
queue.max_events: 0

# 启用持久队列(queue.type: persisted)后，队列的总容量，单位字节，默认1024mb
queue.max_bytes: 1024mb

# 启用持久队列(queue.type: persisted)后，在强制检查点之前的最大ACKed事件数，默认1024
queue.checkpoint.acks: 1024

# 启用持久队列(queue.type: persisted)后，在强制检查点之前的最大书面时间数，默认1024
queue.checkpoint.writes: 1024

# 启用持久队列(queue.type: persisted)后，执行检查点的时间间隔，单位ms，默认1000ms
queue.checkpoint.interval: 1000


# ---------- Dead-Letter Queue Settings ----------

# 是否启用插件支持的DLQ功能的标志，默认false
dead_letter_queue.enable: false

# dead_letter_queue.enable为true时，每个死信队列的最大大小
# 若死信队列的大小超出该值，则被删除，默认1024mb
dead_letter_queue.max_bytes: 1024mb

# 死信队列存储路径，默认path.data/dead_letter_queue
path.dead_letter_queue:


# ---------- Debugging Settings ----------

# 日志输出级别，选项：fatal，error，warn，info，debug，trace，默认info
log.level: info

# 日志格式，选项：json，plain，默认plain
log.format:

# 日志路径，默认LOGSTASH_HOME/logs
path.logs:


# ---------- Other Settings ----------

# 插件存储路径
path.plugins: []

# 是否启用每个管道在不同日志文件中的日志分隔
# 默认false
pipeline.separate_logs: false

```

