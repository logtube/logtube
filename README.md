# Logtube

Logtube 是专门为结构化日志设计的日志系统。

# 日志事件

和其他日志设施不同，Logtube 将每条日志视作为一个对象，而不是一条裸字符串。

## 标准格式

标准格式用于日志**最终存储**使用。

```javascript
{
    "timestamp" : "2006-01-02T15:04:05.999Z07:00",  // timestamp,   required, RFC3339 with miliseconds
    "hostname"  : "example.com",                    // hostname,    required, hostname
    "env"       : "staging",                        // environment, required, environment name
    "project"   : "project",                        // project,     required, project name
    "topic"     : "topic",                          // topic,       required, topic name, log levels such as "debug", "info" are basically topics
    "crid"      : "0987654321af",                   // crid,        optional, correlation id
    "message"   : "this is message body",           // message,     optional, plain message body
    "keyword"   : "keyword1,keyword2",              // keyword,     optional, comma seperated keywords

    "x_custom_field1": "custom field value"         // custom fields, optional, all custom field should begin with "x_"
}
```

其中 `env`, `project`, `topic` 必须满足正则表达式 `/[0-9a-zA-Z_-]/`，实际操作中，不符合的字符会被`_`替换。

## 压缩格式

日志事件的字段可以进一步压缩，压缩格式用于日志的产生，文件存储和传输等中间步骤，标准格式和压缩格式是完全等价的。

```javascript
{
    "t": 1553501437616,                     // timestamp,   required, UNIX epoch in milliseconds
    "h": "example.com",                     // hostname,    required, hostname
    "e": "staging",                         // environment, required, environment name
    "p": "project",                         // project,     required, project name
    "o": "topic",                           // topic,       required, topic name, log levels such as "debug", "info" are basically topics
    "c": "0987654321af",                    // crid,        optional, correlation id
    "m": "this is message body",            // message,     optional, plain message body
    "k": "keyword1,keyword2",               // keyword,     optional, comma seperated keywords
    "x": {                                  // custom fields, optional
        "custom_field1": "custom value"    
    }
}
```

# 日志文件格式

Logtube 规定了日志文件的格式和存储路径，并保持与 Filebeat 的兼容。

## 存储路径

`env`, `topic`, `project` 信息保存在了日志路径的最后三段中。

```
/usr/local/tomcat/logs/[env]/[topic]/[project].2019-03-27.log
```

## 纯文本日志文件格式

对于没有自定义字段的日志而言，建议使用此格式，以保持可读性。

时间戳格式经过精心设计，包含毫秒和时区，且为固定长度。

```
[2019-03-27 12:32:22.324 +0800] CRID[0123456780af] KEYWORD[keyword1,keyword2] message body
```

## 结构化日志文件格式

对于包含自定义字段的日志，使用此格式来保持与 filebeat 的兼容性。

```
[2019-03-27 12:32:22.324 +0800] {"c":"0123456780af", "k":"keyword1,keyword2", "x":{ "duration": 121 }}
```

# 日志传输

Logtube 支持 2 中传输方式，Redis 和 SPTP.

## Redis

通过 Redis LIST 传输有两种模式，一种是直接传输压缩格式的日志事件，另一种是传输 Filebeat 读取日志文件产生的事件。

先前的日志文件路径规范保证了 `env`, `topic`, `project` 信息会通过 Filebeat 事件的 `source` 字段回传。

```
RPUSH logtube [压缩格式的 Event]
```

```
RPUSH logtube.filebeat [Filebeat 读取日志文件后产生的 Event]
```

## SPTP

通过 SPTP 传输压缩格式的日志事件，详见 https://github.com/yankeguo/sptp

SPTP 只适合于内网环境，使用 UDP 封包重组后进行日志传输。

因为 UDP 不会产生回压，且容易丢失，所以只适合实时传输，不适合历史数据批量传输。

# 日志存储

Logtube 收集到日志后，可以写入 ElasticSearch （生产环境）或者统一写入文件（k8s 测试环境）。

# 开发者

Yanke Guo <guoyk.cn@gmail.com>, MIT License

