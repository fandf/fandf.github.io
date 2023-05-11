---
layout: post
title: springboot整合elk7.17.3
date: 2022-8-20
tags: [docker, springboot]
description: 保持学习，保持热爱。
---

[elk环境搭建](http://fandf.top/2022/07/10/ELK7.17.3%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/) \
[idea打包镜像参考](http://fandf.top/2022/07/28/IDEA%E6%89%93%E5%8C%85docker%E9%95%9C%E5%83%8F/)


logback.xml添加下面内容
```
<!--LogStash访问host-->
<springProperty name="LOG_STASH_HOST" scope="context" source="logstash.host" defaultValue="localhost"/>


<!--接口访问记录日志输出到LogStash-->
<appender name="LOG_STASH_RECORD" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>${LOG_STASH_HOST}:4560</destination>
    <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
        <providers>
            <timestamp>
                <timeZone>Asia/Shanghai</timeZone>
            </timestamp>
            <!--自定义日志输出格式-->
            <pattern>
                <pattern>
                    {
                    "project": "spring-cloud-learning",
                    "level": "%level",
                    "service": "${APP_NAME:-}",
                    "class": "%logger",
                    "traceId": "%X{traceId}",
                    "spanId": "%X{spanId}",
                    "message": "%message"
                    }
                </pattern>
            </pattern>
        </providers>
    </encoder>
    <!--当有多个LogStash服务时，设置访问策略为轮询-->
    <connectionStrategy>
        <roundRobin>
            <connectionTTL>5 minutes</connectionTTL>
        </roundRobin>
    </connectionStrategy>
</appender>


<logger name="com.fandf.log.aspect.WebLogAspect" level="DEBUG">
    <appender-ref ref="LOG_STASH_RECORD"/>
</logger>


```

springboot项目配置文件添加
```
#logstash ip，mac本地连接方式
logstash:
  host: host.docker.internal
```

打包成镜像后执行
```
docker run -p 9005:9005 --name user-service \
--link logstash:logstash \
-v /etc/localtime:/etc/localtime \
-d fandf/user-service:1.0.0
```
我的controller

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/98a4a7bd4f4d4e6e98b85202ec661467~tplv-k3u1fbpfcp-watermark.image?)

调用接口\
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67d5dac186fd4fe6ad9d31966551a97d~tplv-k3u1fbpfcp-watermark.image?)

登录kibana创建索引
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42ece61216004d1d8b69cd897267e5bd~tplv-k3u1fbpfcp-watermark.image?)

description 为swagger注解描述， traceId为全局链路id
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1576dd4d3ea94677ad0c1ed7c931a852~tplv-k3u1fbpfcp-watermark.image?)

[源码地址](https://github.com/fandf/SpringCloudLearning/tree/master/fdf-demo/seata-demo/user-service)
* 只测试elk，可以在源码 application-docker.yml里面删除seata和nacos相关配置即可