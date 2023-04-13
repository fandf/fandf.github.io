---
layout: post
title: Kubernetes单节点环境安装
date: 2023-04-13
tags: [k8s]
description: 又是加班的一天
---

## linux安装k8s单节点

**配置最低2核4g**

```shell
#安装kubectl
yum install -y kubectl-1.18.0

#安装minikube
curl -LO https://storage.googleapis.com/minikube/releases/v1.18.1/minikube-linux-amd64 && sudo install minikube-linux-amd64 /usr/local/bin/minikube


#检查minikube安装是否成功
minikube  version


#启动minikube
minikube start --image-mirror-country='cn'  --driver=docker --force --kubernetes-version=1.18.1 --registry-mirror=https://registry.docker-cn.com

#查看集群信息
kubectl cluster-info
#查看节点信息
kubectl get nodes
```

>欢迎关注个人公众号【好好学技术】交流学习

## mac安装k8s单节点
docker desktop版本
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c80e2b431ffe4ac5915ef99e1d6c03ca~tplv-k3u1fbpfcp-watermark.image?)



查看自己的 Kubernetes版本，然后去阿里云下载对应的版本
>https://github.com/AliyunContainerService/k8s-for-docker-desktop

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c25d1a3cc07a4445a3a70a7841ee594f~tplv-k3u1fbpfcp-watermark.image?)

````shell
$ git clone https://github.com/AliyunContainerService/k8s-for-docker-desktop.git
$ cd k8s-for-docker-desktop

# 查看版本镜像；需和docker desktop 中 Kubernetes版本一直
$ cat images.properties

# 执行脚本安装K8S相关镜像
$ ./load_images.sh

````

可选操作: 为 Kubernetes 配置 CPU 和 内存资源，建议分配 4GB 或更多内存。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b08df9920bd48eb836d7435eab1004b~tplv-k3u1fbpfcp-watermark.image?)

勾选下面两项


![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed0dba4a10024cbe93282423c4acd39c~tplv-k3u1fbpfcp-watermark.image?)

点击右下角 Apply & Restart 等待安装成功即可。  
docker还需下载一部分镜像,因此过程比较慢，只要没失败就行。  
若多次安装失败，建议更换低版本的docker desktop


启动成功后左下角会变为绿色，running状态，并且新建很多容器

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b0375582f8264d95a893b32a445110e5~tplv-k3u1fbpfcp-watermark.image?)


验证K8s

````shell
// 查询有哪些集群：
 $ kubectl config get-contexts
   CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
   *         docker-desktop   docker-desktop   docker-desktop   

 // 切换k8s的上下文状态到docker-desktop
 $ kubectl config use-context docker-desktop
   Switched to context "docker-desktop".

 // 验证集群状态
 $ kubectl cluster-info
   Kubernetes master is running at https://kubernetes.docker.internal:6443
   KubeDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
 
 // 查看节点信息
 $ kubectl get nodes
   NAME             STATUS   ROLES    AGE     VERSION
   docker-desktop   Ready    master   5m33s   v1.19.3
````