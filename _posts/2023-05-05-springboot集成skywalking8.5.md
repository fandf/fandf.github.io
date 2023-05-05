---
layout: post
title: springboot构建docker镜像并推送到阿里云
date: 2023-05-05
tags: [k8s]
description: 收假第二天
---


## skywalking是什么

skywalking是一款国产的开源框架，它具有分布式链路追踪、性能指标分析、应用和服务依赖的分析等功能。

官网：https://skywalking.apache.org/  
github: https://github.com/apache/skywalking  
下载地址： https://skywalking.apache.org/downloads/  
中文文档：https://skyapm.github.io/document-cn-translation-of-skywalking/

## skywalking整体架构


整体可以分为上下左右四个部分。


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7838fc712c14868957d88a3db6037b9~tplv-k3u1fbpfcp-watermark.image?)

- 上部分（skywalking-agent）
>负责从应用程序中收集链路信息，然后将链路信息发送到skywalkingOAP处理器
- 下部分（skywalking OAP）
>负责接收skywalking-agent发送过来的Tracing数据信息，Analysis Core对这些数据信息进行分析，把分析的数据存储到外部的存储器当中，Query Core提供查询数据的功能
- 左部分（skywalking UI）
>负责展示链路信息等数据
- 右部分（数据存储）
>负责存储skywalking分析后的数据

## docker部署skywalking

### docker部署es
```shell

mkdir -p /Users/dongfengfan/docker/es/config
mkdir -p /Users/dongfengfan/docker/es/data
chmod 777 -R /Users/dongfengfan/docker/es

vi /Users/dongfengfan/docker/es/config/es.yml


http.cors.enabled: true
http.cors.allow-origin: "*"
 
#节点名称
node.name: "node-1"
#节点ip 单机默认回环地址 集群必须绑定真实ip
network.host: 0.0.0.0
#集群名称
cluster.name: es-cluster


#启动运行
docker run -d --name my_es -p 9200:9200 -p 9300:9300 \
  -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms256m -Xmx256m" \
  -v /Users/dongfengfan/docker/es/config/es.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
  -v /Users/dongfengfan/docker/es/data \
  -v /Users/dongfengfan/docker/es/data/plugins:/usr/share/elasticsearch/plugins elasticsearch:7.17.3
  
 ```

运行成功后访问节点信息
 ```shell
curl http://127.0.0.1:9200/_cat/nodes\?v\=true\&pretty
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
172.17.0.2           88          38  66    1.18    0.32     0.11 cdfhilmrstw *      node-1

curl http://127.0.0.1:9200
{
  "name" : "node-1",
  "cluster_name" : "es-cluster",
  "cluster_uuid" : "WCuTwa0lSxqLGd5QcPsgmQ",
  "version" : {
    "number" : "7.17.3",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "5ad023604c8d7416c9eb6c0eadb62b14e766caff",
    "build_date" : "2022-04-19T08:11:19.070913226Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```

### docker部署skywalking-oap-server

```shell
~ ➤ docker run --name my_oap -d -e TZ=Asia/Shanghai -p 12800:12800 -p 11800:11800 --link my_es -e SW_STORAGE=elasticsearch7 -e SW_STORAGE_ES_CLUSTER_NODES=my_es:9200 apache/skywalking-oap-server:8.5.0-es7
Unable to find image 'apache/skywalking-oap-server:8.5.0-es7' locally
8.5.0-es7: Pulling from apache/skywalking-oap-server
532819f3e44c: Pull complete
aea5e724a03c: Pull complete
5d755eafed5c: Pull complete
a1b90b87dc8e: Pull complete
4f4fb700ef54: Pull complete
42c01aa30672: Pull complete
Digest: sha256:4a5eb8b26093023e987729b453ea09f1610a12321c40ad2fcad183e6da0037e4
Status: Downloaded newer image for apache/skywalking-oap-server:8.5.0-es7
39d50d30c72ac6c040bfc5d71418b631ad0ddb1a2b608d64c0e2e7186ec98096
```

### docker部署skywalking-ui
```shell
~ ➤ docker run -d --name my_skywalking-ui \
-e TZ=Asia/Shanghai \
-p 8080:8080 \
--link my_oap \
-e SW_OAP_ADDRESS=my_oap:12800 \
apache/skywalking-ui:8.5.0
Unable to find image 'apache/skywalking-ui:8.5.0' locally
8.5.0: Pulling from apache/skywalking-ui
532819f3e44c: Already exists
aea5e724a03c: Already exists
5d755eafed5c: Already exists
cc751b38495f: Pull complete
4f4fb700ef54: Pull complete
c6cd42db658e: Pull complete
Digest: sha256:2490903d19c236e2ccb7c5b4e4cdd1e5874acf0065f5bdfc84ff484e9b872603
Status: Downloaded newer image for apache/skywalking-ui:8.5.0
58b69c053a551f2dc173f3bdb1fbd06428ea1a1d918ca014dcdb097cc30d54fb
```
skywalking-ui访问地址 127.0.0.1:8080


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db824b39a84a4f51be199cd89cb01bef~tplv-k3u1fbpfcp-watermark.image?)

查看es索引 http://127.0.0.1:9200/_cat/indices?v=true&pretty
```shell
health status index                                             uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   sw_profile_task_segment_snapshot-20230502         pc8CECjiTIWq7bK-V8nw_A   1   1          0            0       226b           226b
yellow open   sw_service_instance_relation_server_side-20230502 NjuqEXPgTAWUP7_-p4rJrw   1   1          0            0       226b           226b
yellow open   sw_metrics-histogram-20230502                     VQFQnQxMQeGqkCIJFenbPA   1   1          0            0       226b           226b
yellow open   sw_metrics-apdex-20230502                         2EBPiMDGSnK4TQdYn62ngA   1   1          0            0       226b           226b
yellow open   sw_meter-avg-20230502                             v1yjQnaMTM6gu6B4vrSGew   1   1          0            0       226b           226b
yellow open   sw_metrics-cpm-20230502                           PtRxg0YyRbiWt5EOH3-fUw   1   1          0            0       226b           226b
green  open   .geoip_databases                                  NGkslaWoT7yfxDfrnGMujA   1   0         42            0       40mb           40mb
yellow open   sw_metrics-doubleavg-20230502                     E1XmOWI3RsCfGjxkMHNjJw   1   1          0            0       226b           226b
yellow open   sw_ui_template                                    vfLAxFJISq-jtTd6qryJMw   1   1         11            0     53.1kb         53.1kb
green  open   sw_segment-20230502                               Hgbnq5YcS16Kyc5sSY2vXg   5   0          0            0      1.1kb          1.1kb
yellow open   sw_alarm_record-20230502                          9xiyiK0KQnmntsSYDM8rng   1   1          0            0       226b           226b
green  open   sw_zipkin_span-20230502                           CVRUaRuNSPmngw5ELJqKag   5   0          0            0      1.1kb          1.1kb
yellow open   sw_network_address_alias-20230502                 8N9qdS8JTFSnjUDXJqDx6w   1   1          0            0       226b           226b
yellow open   sw_instance_traffic-20230502                      jTcdJrtiSuOo1EAthpZpXQ   1   1          0            0       226b           226b
yellow open   sw_metrics-percent-20230502                       tIzHFaLmRzS41AJjyM4doA   1   1          0            0       226b           226b
yellow open   sw_profile_task_log-20230502                      cqwo2saVQAm61dGElctJZw   1   1          0            0       226b           226b
yellow open   sw_service_relation_client_side-20230502          KpKYOtkoTTegGxHxP8_faA   1   1          0            0       226b           226b
yellow open   sw_metrics-max-20230502                           WdVPFdrIRX-0QFFIYoSMkg   1   1          0            0       226b           226b
yellow open   sw_metrics-percentile-20230502                    am5_fvzFRfePc9oVkaATsw   1   1          0            0       226b           226b
yellow open   sw_service_relation_server_side-20230502          K6Y23_HMRR2e7CfMcqfWng   1   1          0            0       226b           226b
yellow open   sw_service_traffic-20230502                       0o8ye7RQT4WZcgiEYkkHFg   1   1          0            0       226b           226b
yellow open   sw_metrics-sum-20230502                           x07vw6n-T4OKkbCtEfowxA   1   1          0            0       226b           226b
yellow open   sw_profile_task-20230502                          xFVBJzkVQSKt14BHzFNAig   1   1          0            0       226b           226b
yellow open   sw_events-20230502                                qqgkFBm-RxuKrYTAxCSbAA   1   1          0            0       226b           226b
yellow open   sw_endpoint_relation_server_side-20230502         bT51Y4OdSk6r9HhOzYkR-g   1   1          0            0       226b           226b
yellow open   sw_top_n_database_statement-20230502              SKBdlR63TVyaey8ImfjZgw   1   1          0            0       226b           226b
yellow open   sw_metrics-longavg-20230502                       OXz2N4w5Tx2pA8NPG1r-Eg   1   1          0            0       226b           226b
yellow open   sw_endpoint_traffic-20230502                      dRe1IUEVQ1KzQ9yS8-wvrQ   1   1          0            0       226b           226b
yellow open   sw_service_instance_relation_client_side-20230502 9WFHpKYzQ0-6Us_RKDOEnQ   1   1          0            0       226b           226b
green  open   sw_browser_error_log-20230502                     ML055qUwSBiIMr4JhM3HVg   5   0          0            0      1.1kb          1.1kb
yellow open   sw_metrics-rate-20230502                          x3WRDgM8QNGLtOR1gXX09Q   1   1          0            0       226b           226b
green  open   sw_log-20230502                                   bvXnTlyPRI-BC2L_Gaotmw   5   0          0            0      1.1kb          1.1kb
```

## springboot集成skywalking

### skywalking agent探针
-   探针表示集成到目标系统中的代理或 SDK 库,即收集并格式化数据, 并发送到后端 包括链路追踪和性能指标  
    下载地址：https://archive.apache.org/dist/skywalking/8.5.0/apache-skywalking-apm-8.5.0.tar.gz

### skywalking agent使用

- 优先级：探针-> JVM配置-> 环境变量配置 -> agent.config（优先级低）
-   第一种-javaagent:/path/to/skywalking-agent.jar={config1}={value1},{config2}={value2}
```shell
-javaagent:../skywalking-agent.jar=agent.service_name=fdf-user,collector.backend_service=127.0.0.1:11800
```
-   第二种：-Dskywalking.[option1]=[value2]
```
-javaagent: ../skywalking-agent.jar -Dskywalking.agent.service_name=fdf-user -Dskywalking.collector.backend_service=127.0.0.1:11800
``` 
-   agent.service_name：客户端服务名，在apm系统中显示的服务名称
-   collector.backend_service：skywalking上传的服务地址

### idea配置skywalking agent

vm options增加参数

-javaagent:/Users/dongfengfan/Desktop/skywalking-agent/skywalking-agent.jar -Dskywalking.agent.service_name=fdf-test -Dskywalking.collector.backend_service=127.0.0.1:11800


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10e8862b78314881984c9e6497eedb7c~tplv-k3u1fbpfcp-watermark.image?)

项目启动后多调用几次接口。存在一定的延迟

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77780f761b2f43ef8cc5e3f39ffdb2f2~tplv-k3u1fbpfcp-watermark.image?)


#### SkyWalking链路追踪配置
pom.xml添加依赖
```
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>8.5.0</version>
</dependency>
```
logback.xml添加打印traceId
```xml
<appender name="console" class="ch.qos.logback.core.ConsoleAppender">  
    <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">  
        <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">  
            <Pattern>${logEnv} %d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] [%thread] %-5level %logger{36} -%msg%n  
            </Pattern>  
        </layout>  
    </encoder>  
</appender>
```

调用接口，查看打印日志
```
logEnv_IS_UNDEFINED 2023-05-02 16:23:11.203 [TID:ae035ab4388a42839e42453f0d72c137.70.16830157911950001] [http-nio-9002-exec-2] INFO  c.f.test.controller.UserController -收到请求信息 name = 好好学技术
logEnv_IS_UNDEFINED 2023-05-02 16:23:14.919 [TID:ae035ab4388a42839e42453f0d72c137.71.16830157949150001] [http-nio-9002-exec-3] INFO  c.f.test.controller.UserController -收到请求信息 name = 好好学技术
logEnv_IS_UNDEFINED 2023-05-02 16:24:33.587 [TID:ae035ab4388a42839e42453f0d72c137.74.16830158735810001] [http-nio-9002-exec-6] INFO  c.f.test.controller.UserController -收到请求信息 name = 好好学技术
logEnv_IS_UNDEFINED 2023-05-02 16:24:34.765 [TID:ae035ab4388a42839e42453f0d72c137.76.16830158747640001] [http-nio-9002-exec-7] INFO  c.f.test.controller.UserController -收到请求信息 name = 好好学技术
```
也可以通过logstash将日志上送到elk，方便排查日志

