---
title: tcp-logger
keywords:
  - Apache APISIX
  - API 网关
  - Plugin
  - TCP Logger
description: 本文介绍了 API 网关 Apache APISIX 如何使用 tcp-logger 插件将日志数据发送到 TCP 服务器。
---

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

## 描述

`tcp-logger` 插件可用于将日志数据发送到 TCP 服务器。

该插件还实现了将日志数据以 JSON 格式发送到监控工具或其它 TCP 服务的能力。

## 属性

| 名称             | 类型     | 必选项  | 默认值 | 有效值  | 描述                                             |
| ---------------- | ------- | ------ | ------ | ------- | ------------------------------------------------ |
| host             | string  | 是     |        |         | TCP 服务器的 IP 地址或主机名。                     |
| port             | integer | 是     |        | [0,...] | 目标端口。                                        |
| timeout          | integer | 否     | 1000   | [1,...] | 发送数据超时间。                                   |
| log_format       | object  | 否   |          |         | 以 JSON 格式的键值对来声明日志格式。对于值部分，仅支持字符串。如果是以 `$` 开头，则表明是要获取 [APISIX 变量](../apisix-variable.md) 或 [NGINX 内置变量](http://nginx.org/en/docs/varindex.html)。 |
| tls              | boolean | 否     | false  |         | 用于控制是否执行 SSL 验证。                        |
| tls_options      | string  | 否     |        |         | TLS 选项。                                        |
| include_req_body | boolean | 否     |        |         | 当设置为 `true` 时，日志中将包含请求体。           |

该插件支持使用批处理器来聚合并批量处理条目（日志/数据）。这样可以避免插件频繁地提交数据，默认情况下批处理器每 `5` 秒钟或队列中的数据达到 `1000` 条时提交数据，如需了解批处理器相关参数设置，请参考 [Batch-Processor](../batch-processor.md#配置)。

### 默认日志格式示例

```json
{
  "response": {
    "status": 200,
    "headers": {
      "server": "APISIX/3.7.0",
      "content-type": "text/plain",
      "content-length": "12",
      "connection": "close"
    },
    "size": 118
  },
  "server": {
    "version": "3.7.0",
    "hostname": "localhost"
  },
  "start_time": 1704527628474,
  "client_ip": "127.0.0.1",
  "service_id": "",
  "latency": 102.9999256134,
  "apisix_latency": 100.9999256134,
  "upstream_latency": 2,
  "request": {
    "headers": {
      "connection": "close",
      "host": "localhost"
    },
    "size": 59,
    "method": "GET",
    "uri": "/hello",
    "url": "http://localhost:1984/hello",
    "querystring": {}
  },
  "upstream": "127.0.0.1:1980",
  "route_id": "1"
}
```

## 插件元数据

| 名称             | 类型    | 必选项 | 默认值        | 有效值  | 描述                                             |
| ---------------- | ------- | ------ | ------------- | ------- | ------------------------------------------------ |
| log_format       | object  | 否    |  |         | 以 JSON 格式的键值对来声明日志格式。对于值部分，仅支持字符串。如果是以 `$` 开头。则表明获取 [APISIX 变量](../apisix-variable.md) 或 [NGINX 内置变量](http://nginx.org/en/docs/varindex.html)。 |

:::info 注意

该设置全局生效。如果指定了 `log_format`，则所有绑定 `tcp-logger` 的路由或服务都将使用该日志格式。

:::

以下示例展示了如何通过 Admin API 配置插件元数据：

```shell
curl http://127.0.0.1:9180/apisix/admin/plugin_metadata/tcp-logger \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "log_format": {
        "host": "$host",
        "@timestamp": "$time_iso8601",
        "client_ip": "$remote_addr"
    }
}'
```

配置完成后，你将在日志系统中看到如下类似日志：

```json
{"@timestamp":"2023-01-09T14:47:25+08:00","route_id":"1","host":"localhost","client_ip":"127.0.0.1"}
```

## 启用插件

你可以通过以下命令在指定路由中启用该插件：

```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
      "plugins": {
            "tcp-logger": {
                 "host": "127.0.0.1",
                 "port": 5044,
                 "tls": false,
                 "batch_max_size": 1,
                 "name": "tcp logger"
            }
       },
      "upstream": {
           "type": "roundrobin",
           "nodes": {
               "127.0.0.1:1980": 1
           }
      },
      "uri": "/hello"
}'
```

## 测试插件

现在你可以向 APISIX 发起请求：

```shell
curl -i http://127.0.0.1:9080/hello
```

```
HTTP/1.1 200 OK
...
hello, world
```

## 删除插件

当你需要删除该插件时，可通过以下命令删除相应的 JSON 配置，APISIX 将会自动重新加载相关配置，无需重启服务：

```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/hello",
    "plugins": {},
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:1980": 1
        }
    }
}'
```
