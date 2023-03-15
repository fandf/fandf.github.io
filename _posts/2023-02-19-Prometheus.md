---
layout: post
title: docker部署Prometheus+Grafana+node-exporter
date: 2023-02-19
tags: [docker,运维]
description: 保持学习，保持热爱。
---
## Prometheus
### Prometheus介绍
Prometheus（普罗米修斯）是一个开源的系统监控和报警系统。Google SRE的书内也曾提到跟他们BorgMon监控系统相似的实现是Prometheus。现在最常见的Kubernetes容器管理系统中，通常会搭配Prometheus进行监控。

Prometheus基本原理是通过HTTP协议周期性抓取被监控组件的状态，这样做的好处是任意组件只要提供HTTP接口就可以接入监控系统，不需要任何SDK或者其他的集成过程。这样做非常适合虚拟化环境比如VM或者Docker 。

Prometheus应该是为数不多的适合Docker、Kubernetes环境的监控系统之一。

输出被监控组件信息的HTTP接口被叫做exporter 。目前互联网公司常用的组件大部分都有exporter可以直接使用，比如Varnish、Haproxy、Nginx、MySQL、Linux 系统信息 (包括磁盘、内存、CPU、网络等等)，具体支持的源看：<https://github.com/prometheus>。

**高效的存储，每个采样数据占3.5 bytes左右，300万的时间序列，30s间隔，保留60天，消耗磁盘大概200G**。

### Prometheus组件
* `Prometheus Server`: 用于收集和存储时间序列数据
* `Client Library`: 客户端库，检测应用程序代码，当Prometheus抓取实例的HTTP端点时，客户端库会将所有跟踪的metrics指标的当前状态发送到prometheus server端。
* `Exporters`: prometheus支持多种exporter，通过exporter可以采集metrics数据，然后发送到prometheus server端，所有向promtheus server提供监控数据的程序都可以被称为exporter
* `Alertmanager`: 从 Prometheus server 端接收到 alerts 后，会进行去重，分组，并路由到相应的接收方，发出报警，常见的接收方式有：电子邮件，微信，钉钉, slack等。
* `Grafana`：监控仪表盘，可视化监控数据
* `pushgateway`: 各个目标主机可上报数据到pushgateway，然后prometheus server统一从pushgateway拉取数据。


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea70ca281ac844abac20098c8bca5c0c~tplv-k3u1fbpfcp-watermark.image?)

从上图可发现，Prometheus整个生态圈组成主要包括prometheus server，Exporter，pushgateway，alertmanager，grafana，Web ui界面，Prometheus server由三个部分组成，Retrieval，Storage，PromQL
-   `Retrieval`负责在活跃的target主机上抓取监控指标数据
-   `Storage`存储主要是把采集到的数据存储到磁盘中
-   `PromQL`是Prometheus提供的查询语言模块。

### Prometheus工作流程
1）Prometheus server可定期从活跃的（up）目标主机上（target）拉取监控指标数据，目标主机的监控数据可通过**配置静态job**或者**服务发现**的方式被prometheus server采集到，这种方式默认的pull方式拉取指标；也可通过pushgateway把采集的数据上报到prometheus server中；还可通过一些组件自带的exporter采集相应组件的数据；

2）Prometheus server把采集到的监控指标数据保存到本地磁盘或者数据库；

3）Prometheus采集的监控指标数据按时间序列存储，通过配置报警规则，把触发的报警发送到alertmanager

4）Alertmanager通过配置报警接收方，发送报警到邮件，微信或者钉钉等

5）Prometheus 自带的web ui界面提供PromQL查询语言，可查询监控数据

6）Grafana可接入prometheus数据源，把监控数据以图形化形式展示出来

## 实战
环境

| 服务器 | 部署应用 |
| --- | --- |
| 本地mac | Prometheus,Grafana,node-exporter |
| centos | node-exporter |

本地机器部署Prometheus+Grafana+node-exporter，远程服务器也部署一个node-exporter。
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dec1b1e9623646df8b3605c737b4bf0c~tplv-k3u1fbpfcp-watermark.image?)
### 部署node-exporter
本地机器环境搭建
```docker

# 本地创建node-exporter，生产可加上 --restart=always 
docker run -d -p 9100:9100 \
  -v "/Users/dongfengfan/docker/node-exporter/proc:/host/proc:ro" \
  -v "/Users/dongfengfan/docker/node-exporter/sys:/host/sys:ro" \
  -v "/Users/dongfengfan/docker/node-exporter/:/rootfs:ro" \
  --name node-exporter \
  prom/node-exporter
 
# 远程centos创建node-exporter，生产可加上 --restart=always 
docker run -d -p 9100:9100 \
  -v "/root/docker/node-exporter/proc:/host/proc:ro" \
  -v "/root/docker/node-exporter/sys:/host/sys:ro" \
  -v "/root/docker/node-exporter/:/rootfs:ro" \
  --net="host" \
  --name node-exporter \
  prom/node-exporter
```
查看node-exporter是否部署成功
访问 http://127.0.0.1:9100/metrics

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d6aa1c9a544491ba76ea6cd6f557308~tplv-k3u1fbpfcp-watermark.image?)


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43fc695deac14c85b0d79905348d91bc~tplv-k3u1fbpfcp-watermark.image?)

### 部署Prometheus
创建 prometheus.yml
```yaml
global:
  scrape_interval:     60s
  evaluation_interval: 60s
 
scrape_configs:
  - job_name: prometheus
    static_configs:
        #本地服务器加端口
      - targets: ['localhost:9090']
        labels:
          instance: prometheus
 
  - job_name: localhost-node-exporter
    static_configs:
        #监控本地服务器ip+端口，因为是本地docker启动，所以ip使用host.docker.internal
      - targets: ['host.docker.internal:9100']
        labels:
          instance: localhost-node-exporter
  - job_name: myaliyun
    static_configs:
      #监控远程服务器
      - targets: ['139.196.45.28:9100']
        labels:
          instance: myaliyun
 ```
启动prometheus
```docker 
docker run  -d \
  -p 9090:9090 \
  -v /Users/dongfengfan/docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml  \
  --name prometheus \
  prom/prometheus
```
访问http://127.0.0.1:9090/targets,三个任务已经全部监控上了

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58be30ddb56244c88c6d42f04c9dff9e~tplv-k3u1fbpfcp-watermark.image?)

### 部署Grafana
```docker
mkdir /Users/dongfengfan/docker/grafana

docker run -d \
  -p 3000:3000 \
  --name=grafana \
  -v /Users/dongfengfan/docker/grafana:/var/lib/grafana \
  --name grafana \
  grafana/grafana
```
访问 http://127.0.0.1:3000/  
默认会先跳转到登录页面，默认的用户名和密码都是admin  
登录之后，它会要求你重置密码。你还可以再输次admin密码

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/811775f3647a4a9c844d03647c40bec6~tplv-k3u1fbpfcp-watermark.image?)
进入首页点击 Add data source

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc5a1bbda8af428ba7dd5c69663b33c0~tplv-k3u1fbpfcp-watermark.image?)
选择Prometheus

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f06d11a9002d4cc29cafb05391111985~tplv-k3u1fbpfcp-watermark.image?)
配置url为Prometheus地址，我这里因为Grafana和Prometheus都部署在本地docker,所以访问ip使用host.docker.internal
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e07036c24d74ef2ab07ce0f9242404e~tplv-k3u1fbpfcp-watermark.image?)
拉到最下面选择 save & test

### # Node Exporter for Prometheus Dashboard 中文版
访问地址：https://grafana.com/grafana/dashboards/8919/revisions  
我下载了两个比较下有啥不同

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5a56f50e6434b33a12f7b649799f3f5~tplv-k3u1fbpfcp-watermark.image?)

导入数据

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb804a6c775a4ad59a7e5c34cb4af669~tplv-k3u1fbpfcp-watermark.image?)
上传模版
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/765ee27038074689a225a7813f3c4d88~tplv-k3u1fbpfcp-watermark.image?)
到这里基本就结束了，我们来看一下效果。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfb4d11b44d54b66ad7a5df6137022ae~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/680b287936114a7f838db1969c932064~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d80ca853b75f4c5d932024c803c55e36~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1904429b0ae04f3c98f767b5d52a5ef1~tplv-k3u1fbpfcp-watermark.image?)

后面有时间再出一篇文章，如何监控mongo,redis,mysql等中间件。