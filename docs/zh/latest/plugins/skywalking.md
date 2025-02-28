---
title: skywalking
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

[**SkyWalking**](https://github.com/apache/skywalking) 使用其原生的 Nginx LUA tracer 提供分布式追踪、拓扑分析以及服务和 URI 级别的 metrics 信息。

服务端目前支持 HTTP 和 gRPC 两种协议，在 APISIX 中目前只支持 HTTP 协议。

## 属性

| 名称         | 类型   | 必选项 | 默认值   | 有效值       | 描述                                                  |
| ------------ | ------ | ------ | -------- | ------------ | ----------------------------------------------------- |
| sample_ratio | number | 必须   | 1        | [0.00001, 1] | 采样的比例                                            |

## 如何启用

首先，你需要在 `config.yaml` 里面启用 SkyWalking 插件：

```
# 加到 config.yaml
plugins:
  - ... # plugin you need
  - skywalking
```

然后重载 APISIX，这样会创建一个后台定时器，向 SkyWalking OAP 服务定期上报数据。

下面是一个示例，在指定的 route 上开启了 SkyWalking 插件:

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uris": [
        "/uid/*"
    ],
    "plugins": {
        "skywalking": {
            "sample_ratio": 1
        }
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "10.110.149.175:8089": 1
        }
    }
}'
```

你也可以通过 web 界面来完成上面的操作，先增加一个 route，然后在插件页面中添加 SkyWalking 插件：

![plugin_skywalking](../../../assets/images/plugin/skywalking-1.png)

## 如何设置 Endpoint

我们可以通过指定 `conf/config.yaml` 中的配置来指定 endpoint：

| 名称         | 类型   | 默认值   | 描述                                                  |
| ------------ | ------ | -------- | ----------------------------------------------------- |
| service_name | string |  "APISIX" | SkyWalking 上报的 service 名称                                 |
| service_instance_name | string | "APISIX Instance Name" | SkyWalking 上报的 service 实例名, 如果期望直接获取本机主机名则设置为 `$hostname` |
| endpoint_addr | string | "http://127.0.0.1:12800" | SkyWalking 的 HTTP endpoint 地址，例如：http://127.0.0.1:12800 |
| report_interval | integer | 使用 SkyWalking 客户端内置的值 | 上报时间间隔，单位是秒|

配置示例:

```yaml
plugin_attr:
  skywalking:
    service_name: APISIX
    service_instance_name: "APISIX Instance Name"
    endpoint_addr: http://127.0.0.1:12800
```

## 测试插件

### 运行 SkyWalking 实例

#### 例子:

1. 启动 SkyWalking OAP 服务:
    - SkyWalking 默认使用 H2 存储，可直接启动

        ```shell
        sudo docker run --name skywalking -d -p 1234:1234 -p 11800:11800 -p 12800:12800 --restart always apache/skywalking-oap-server:8.7.0-es6
        ```

    - 也许你会更倾向于使用 Elasticsearch 存储
        1. 则需要先安装 Elasticsearch:

            ```shell
            sudo docker run -d --name elasticsearch -p 9200:9200 -p 9300:9300 --restart always -e "discovery.type=single-node" elasticsearch:6.7.2
            ```

        2. 【可选】安装 ElasticSearch 管理界面 elasticsearch-hq

            ```shell
            sudo docker run -d --name elastic-hq -p 5000:5000 --restart always elastichq/elasticsearch-hq
            ```

        3. 启动 SkyWalking OAP 服务：

            ```shell
            sudo docker run --name skywalking -d -p 1234:1234 -p 11800:11800 -p 12800:12800 --restart always --link elasticsearch:elasticsearch -e SW_STORAGE=elasticsearch -e SW_STORAGE_ES_CLUSTER_NODES=elasticsearch:9200 apache/skywalking-oap-server:8.7.0-es6
            ```

2. SkyWalking Web UI：
    1. 启动管理系统：

        ```shell
        sudo docker run --name skywalking-ui -d -p 8080:8080 --link skywalking:skywalking -e SW_OAP_ADDRESS=skywalking:12800 --restart always apache/skywalking-ui
        ```

    2. 打开 Web UI 页面
        在浏览器里面输入 http://10.110.149.175:8080 如出现如下界面，则表示安装成功

        ![plugin_skywalking](../../../assets/images/plugin/skywalking-3.png)

3. 测试示例:
    - 通过访问 APISIX，访问上游服务

        ```bash
        $ curl -v http://10.110.149.192:9080/uid/12
        HTTP/1.1 200 OK
        OK
        ...
        ```

    - 打开浏览器，访问 SkyWalking 的 UI 页面：

        ```bash
        http://10.110.149.175:8080/
        ```

        可以看到服务拓扑图\
        ![plugin_skywalking](../../../assets/images/plugin/skywalking-4.png)\
        可以看到服务追踪列表\
        ![plugin_skywalking](../../../assets/images/plugin/skywalking-5.png)

## 禁用插件

当你想禁用一条路由/服务上的 SkyWalking 插件的时候，很简单，在插件的配置中把对应的 JSON 配置删除即可，无须重启服务，即刻生效：

```shell
$ curl http://127.0.0.1:9080/apisix/admin/routes/1  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uris": [
        "/uid/*"
    ],
    "plugins": {
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "10.110.149.175:8089": 1
        }
    }
}'
```

现在就已经移除了 SkyWalking 插件了。其他插件的开启和移除也是同样的方法。

如果你想完全禁用 SkyWalking 插件，比如停掉后台上报数据的定时器，需要在 `config.yaml` 里把插件注释掉：

```yaml
plugins:
  - ... # plugin you need
  #- skywalking
```

然后重载 APISIX 即可。

## 上游服务为 SpringBoot 的示例代码

```java
package com.lenovo.ai.controller;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import javax.servlet.http.HttpServletRequest;

/**
 * @author cyxinda
 * @create 2020-05-29 14:02
 * @desc skywalking test controller
 **/
@RestController
public class TestController {
    @RequestMapping("/uid/{count}")
    public String getUidList(@PathVariable("count") String countStr, HttpServletRequest request) {
        System.out.println("counter:::::"+countStr);
       return "OK";
    }
}

```

启动服务的时候，需要配置 SkyWalking agent。

通过 `agent/config/agent.config` 修改配置

```shell
agent.service_name=yourservername
collector.backend_service=10.110.149.175:11800
```

启动服务脚本：

```shell
nohup java -javaagent:/root/skywalking/app/agent/skywalking-agent.jar \
-jar /root/skywalking/app/app.jar \
--server.port=8089 \
2>&1 > /root/skywalking/app/logs/nohup.log &
```
