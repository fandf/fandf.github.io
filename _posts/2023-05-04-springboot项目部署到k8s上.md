---
layout: post
title: springboot项目部署到k8s上
date: 2023-05-04
tags: [k8s,springboot]
description: 收假了，五天好快啊
---


## springboot部署到k8s步骤
- springboot项目打包镜像部署到镜像仓库
- 登录私有镜像仓库，拉去镜像
- 创建deployment
- 暴露服务访问端口

    

上篇文章已讲过 [springboot构建docker镜像并推送到阿里云](https://juejin.cn/post/7225803401660710949)

## 创建secret
登录私有仓库需要创建secret，存储docker registry的认证信息

### 创建secret
```shell
~$ kubectl create secret docker-registry fdf-docker-secret --docker-server=registry.cn-chengdu.aliyuncs.com --docker-username=17602117026 --docker-password=用户密码
secret/fdf-docker-secret created
~$
~$
~$ kubectl get secret
NAME                TYPE                             DATA   AGE
fdf-docker-secret   kubernetes.io/dockerconfigjson   1      15s
```
创建好之后，后续需要挂载到pod上

### 创建deployment的yaml文件

快速创建一个deployment,导出yaml文件

```shell
~$ kubectl create deployment k8sdemo --image=registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0 --dry-run=client -o yaml > k8sdemo.yaml
~$ cat k8sdemo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: k8sdemo
  name: k8sdemo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8sdemo
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: k8sdemo
    spec:
      containers:
      - image: registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0
        name: k8s-test
        resources: {}
status: {}
~$
```
修改k8sdemo.yaml文件
```yaml
#1.修改副本数量为2 
#2.挂在secret

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: k8sdemo
  name: k8sdemo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: k8sdemo
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: k8sdemo
    spec:
      imagePullSecrets:
        - name: fdf-docker-secret
      containers:
      - image: registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0
        name: k8s-test
        resources: {}
status: {}
```

### 创建deployment
```shell
~$ kubectl apply -f k8sdemo.yaml
deployment.apps/k8sdemo created
~$ kubectl get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
k8sdemo   0/2     2            0           12s
~$ kubectl get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
k8sdemo   2/2     2            2           24s
~$ kubectl get pod
NAME                       READY   STATUS    RESTARTS   AGE
k8sdemo-65d45fb49f-l4kx5   1/1     Running   0          28s
k8sdemo-65d45fb49f-pqsjw   1/1     Running   0          28s
```

### 创建service, nodePort

```shell
#port 服务端口    target-port pod端口

~$ kubectl expose deploy k8sdemo --port=9001 --target-port=9001 --type=NodePort
service/k8sdemo exposed
~$
~$ kubectl get svc k8sdemo
NAME      TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
k8sdemo   NodePort   10.96.10.79   <none>        9001:31921/TCP   12s
~$ curl http://127.0.0.1:31921/api/v1/user/name
zhangsan                                                                                                                            
```
可以看到serviced的ip为10.96.10.79  对外端口为 31921，安全组需开放该端口才能访问

到这里我们服务部署就算完成了，看一下所有的节点
```shell
~$ kubectl get secret,pod,svc,deploy
NAME                       TYPE                             DATA   AGE
secret/fdf-docker-secret   kubernetes.io/dockerconfigjson   1      52m

NAME                           READY   STATUS    RESTARTS   AGE
pod/k8sdemo-65d45fb49f-l4kx5   1/1     Running   0          39m
pod/k8sdemo-65d45fb49f-pqsjw   1/1     Running   0          39m

NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/k8sdemo      NodePort    10.96.10.79   <none>        9001:31921/TCP   5m51s
service/kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP          301d

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8sdemo   2/2     2            2           39m
~$

```




