---
layout: post
title: 通过elastalert2对elk中的日志进行邮件报警
date: 2023-02-16
tags: [docker,运维]
description: 保持学习，保持热爱。
---

### elastAlert介绍
ElastAlert2是一个简单的插件，用于从Elasticsearch中的数据中发出异常或其他感兴趣的模式的警报。  
这个插件的话，是支持docker部署及k8上部署的。

[elastalert2源码](https://github.com/jertel/elastalert2) \
[elastalert2文档](https://elastalert2.readthedocs.io/en/latest/)
### 环境说明

| 软件 | 版本 |
| --- | --- |
| centos | 7.3 |
| elasticserach | 8.5.3 |
| elastAlert | 2.10.0 |


### 下载elastalert2镜像
```docker
docker pull jertel/elastalert2:2.10.0
```
### 创建挂载文件地址
```shell
cd /usr/local/elk
mkdir elastalert
cd elastalert
mkdir -p data elastalert_modules rules
```
### 创建elastalert配置文件
```shell
vi /usr/local/elk/elastalert/elastalert.yaml
```

```yaml
# 用来加载rule的目录
rules_folder: /opt/elastalert/rules

# 用来设置定时向elasticsearch发送请求，也就是告警执行的频率
run_every:
  minutes: 1

# 用来设置请求里时间字段的范围
buffer_time:
  minutes: 15

es_host: 192.168.0.33
es_port: 9200

# elastalert产生的日志在elasticsearch中的创建的索引
writeback_index: error_log_email
# 失败重试的时间限制
alert_time_limit:
  minutes: 15
```

### 创建报警规则邮件
```shell
vi /usr/local/elk/elastalert/rule/erroremail.yaml
```

```yaml
es_host: 192.168.0.33
es_port: 9200
#rule name 必须是独一的，不然会报错
name: "errorLogeMail"
#选择某一种数据验证方式,支持很多种数据验证方式
# elastalert默认支持的规则有以下几种：
# any：只要有匹配就报警；
# blacklist：compare_key字段的内容匹配上 blacklist数组里任意内容；
# whitelist：compare_key字段的内容一个都没能匹配上whitelist数组里内容；
# change：在相同query_key条件下，compare_key字段的内容，在 timeframe范围内 发送变化；
# frequency：在相同 query_key条件下，timeframe 范围内有num_events个被过滤出 来的异常；我的实验就是参考了这个规则配置的
# spike：在相同query_key条件下，前后两个timeframe范围内数据量相差比例超过spike_height。其中可以通过spike_type设置具体涨跌方向是- up,down,both 。还可以通过threshold_ref设置要求上一个周期数据量的下限，threshold_cur设置要求当前周期数据量的下限，如果数据量不到下限，也不触发；
# flatline：timeframe 范围内，数据量小于threshold 阈值；
# new_term：fields字段新出现之前terms_window_size(默认30天)范围内最多的terms_size (默认50)个结果以外的数据；
# cardinality：在相同 query_key条件下，timeframe范围内cardinality_field的值超过 max_cardinality 或者低于min_cardinality
type: "any"
#这个index 是指再kibana 里边的index  支持正则 log-*
index: "log1"
#时间触发的次数
num_events: 1
#和上边的参数关联，也就是说在1分钟内出发1次会报警
realert:
  minutes: 1
# terms_size: 50
# timeframe:
#   minutes: 1
timestamp_field: "@timestamp"
timestamp_type: "iso"
use_strftime_index: false
alert_subject: "error log email"
alert_subject_args:
  - "message"
  - "@log_name"
alert_text: "you have a error message"
alert_text_args:
  - "message"
filter:
#逻辑组合
- bool:
    #必须存在
    must:
      - match:
          level: "ERROR"
    # #必须不存在，即过滤掉的
    # must_not:
    #   - match:
    #       stackTrace: "org.apache.catalina.connector.ClientAbortException: java.io.IOException: Broken pipe"
    #   - match:    
    #       message: "[SUCCESS]"
alert:
- "email"
#报警邮箱的smtp server

smtp_host: "smtp.163.com"

#报警邮箱的smtp 端口

smtp_port: 25

#需要把认证信息写到额外配置文件里，需要user和password两个属性

smtp_auth_file: /opt/elastalert/smtp_auth_file.yaml
email_reply_to: "fdfjava@163.com"
from_addr: "fdfjava@163.com"
email:
- "fdfjava@163.com"
```

### 创建邮箱授信文件
```shell
vi /usr/local/elk/elastalert/smtp_auth_file.yaml
```

```yaml
user: "fdfjava@163.com"
#邮箱授权码，不是密码
password: "xxxxxxxxxxxxx"
```

### 启动镜像
```
docker run -itd --name elastalert2 \
-v /usr/local/elk/elastalert/data:/opt/elastalert/data \
-v /usr/local/elk/elastalert/elastalert.yaml:/opt/elastalert/config.yaml \
-v /usr/local/elk/elastalert/smtp_auth_file.yaml:/opt/elastalert/smtp_auth_file.yaml \
-v /usr/local/elk/elastalert/rules:/opt/elastalert/rules \
-v /usr/local/elk/elastalert/elastalert_modules:/opt/elastalert/elastalert_modules  \
-e ELASTICSEARCH_HOST="192.168.0.33" \
-e ELASTICSEARCH_PORT=9200 \
-e CONTAINER_TIMEZONE="Asia/Shanghai"  \
-e SET_CONTAINER_TIMEZONE=True \
-e TZ="Asia/Shanghai" \
-e SET_CONTAINER_TIMEZONE=True \
-e ELASTALERT_BUFFER_TIME=10  \
-e ELASTALERT_RUN_EVERY=1  \
jertel/elastalert2:2.10.0
```
### 测试
调用api产生异常日志
![bea9c68720710f1e5453b226d7b55ea.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/689597de4e624882811194efba3ec407~tplv-k3u1fbpfcp-watermark.image?)


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d298833a8fc4ecc8b30814fd2445df1~tplv-k3u1fbpfcp-watermark.image?)