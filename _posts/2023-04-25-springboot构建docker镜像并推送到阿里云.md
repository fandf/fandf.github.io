---
layout: post
title: springboot构建docker镜像并推送到阿里云
date: 2023-04-25
tags: [k8s]
description: 明天开始放5.1假
---


## 1.构建springboot项目

工程目录如下

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc6aa29055da4f7d86e369be08601955~tplv-k3u1fbpfcp-watermark.image?)


UserController
```java
package com.fandf.test.controller;  
  
import org.springframework.web.bind.annotation.GetMapping;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RestController;  
  
/**  
* @author fandongfeng  
*/  
@RestController  
@RequestMapping("/api/v1/user")  
public class UserController {  
  
  
    @GetMapping("name")  
    public String getName() {  
        return "zhangsan";  
    }  
  
  
}
```

application.yml
```yaml
server:  
  port: 9001
```

pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<project xmlns="http://maven.apache.org/POM/4.0.0"  
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">  
    <parent>  
        <artifactId>SpringCloudLearnin</artifactId>  
        <groupId>com.fandf</groupId>  
        <version>1.0.0</version>  
    </parent>  
    <modelVersion>4.0.0</modelVersion>  

    <artifactId>fdf-test</artifactId>  

    <properties>  
        <maven.compiler.source>8</maven.compiler.source>  
        <maven.compiler.target>8</maven.compiler.target>  
    </properties>  

    <dependencies>  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-web</artifactId>  
        </dependency>  
    </dependencies>  


    <build>  
        <plugins>  
            <plugin>  
                <groupId>org.springframework.boot</groupId>  
                <artifactId>spring-boot-maven-plugin</artifactId>  
                <executions>  
                    <execution>  
                        <goals>  
                            <goal>repackage</goal>  
                        </goals>  
                    </execution>  
                </executions>  
                <configuration>  
                    <!-- 指定启动类 -->  
                    <mainClass>com.fandf.test.Application</mainClass>  
                </configuration>  
            </plugin>  
            <plugin>  
                <groupId>io.fabric8</groupId>  
                <artifactId>docker-maven-plugin</artifactId>  
                <configuration>  
                    <images>  
                        <image>  
                            <!-- 命名空间/仓库名称:镜像版本号-->  
                            <name>fandf/${project.name}:${project.version}</name>  
                            <alias>${project.name}</alias>  
                            <build>  
                            <!-- 指定dockerfile文件的位置-->  
                                <dockerFile>${project.basedir}/Dockerfile</dockerFile>  
                                <buildOptions>  
                                    <!-- 网络的配置，与宿主主机共端口号-->  
                                    <network>host</network>  
                                </buildOptions>  
                            </build>  
                        </image>  
                    </images>  
                </configuration>  
                <executions>  
                    <execution>  
                        <id>docker-exec</id>  
                        <!-- 绑定mvn deploy阶段，当执行mvn deploy时 就会执行docker build 和docker push-->  
                        <phase>deploy</phase>  
                        <goals>  
                            <goal>build</goal>  
                        <!-- <goal>push</goal>-->  
                        </goals>  
                    </execution>  
                </executions>  
            </plugin>  
        </plugins>  
    </build>  
  
</project>
```


插件如果无法下载，可引入依赖
```
<dependency>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.40.1</version>
</dependency>
```


## 2.编写Dockerfile
````
# 基础镜像使用Java
FROM java:8
# 作者
MAINTAINER fandongfeng
# VOLUME 指定了临时文件目录为/tmp。
# 其效果是在主机 /var/lib/docker 目录下创建了一个临时文件，并链接到容器的/tmp
VOLUME /tmp
# 将jar包添加到容器中并更名为app.jar
# 此处可以把具体的jar包名称写出来，我这里直接用*号代替了
ADD target/*.jar app.jar
# 指定容器需要映射到主机的端口
EXPOSE 9091
ENTRYPOINT ["java","-jar","app.jar"]
````

项目maven plugins会有docker插件，点击docekr build


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa3e51214e1f476296dc81481c8085b6~tplv-k3u1fbpfcp-watermark.image?)

镜像已被上传到本地

```shell
C:\Users\Administrator>docker images
REPOSITORY                           TAG                                                     IMAGE ID       CREATED             SIZE
fandf/fdf-test                       1.0.0                                                   1a4b935b4d57   About an hour ago   684MB
java                                 8                                                       d23bdf5b1b1b   6 years ago         643MB
```

## 3.推送镜像到阿里云

登录阿里云管控台 https://cr.console.aliyun.com/cn-chengdu/instances  
选择容器镜像服务

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec9f5026a84e46deb75e24a5df044f0e~tplv-k3u1fbpfcp-watermark.image?)

新建镜像仓库

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/120dce67a7b84571af1989eb1f92672a~tplv-k3u1fbpfcp-watermark.image?)


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3ce042b1d804b7cba50b2f1fb34a5fa~tplv-k3u1fbpfcp-watermark.image?)

按照提示上传镜像

```shell
C:\Users\Administrator>docker login --username=17602117026 registry.cn-chengdu.aliyuncs.com
Password:
Login Succeeded

Logging in with your password grants your terminal complete access to your account.
For better security, log in with a limited-privilege personal access token. Learn more at https://docs.docker.com/go/access-tokens/

C:\Users\Administrator>docker images
REPOSITORY                           TAG                                                     IMAGE ID       CREATED             SIZE
fandf/fdf-test                       1.0.0                                                   1a4b935b4d57   About an hour ago   684MB
java                                 8                                                       d23bdf5b1b1b   6 years ago         643MB

C:\Users\Administrator>docker tag 1a4b935b4d57 registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0

C:\Users\Administrator>docker push registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0
The push refers to repository [registry.cn-chengdu.aliyuncs.com/fandf/k8s-test]
83d067b86ae9: Pushed
35c20f26d188: Pushed
c3fe59dd9556: Pushed
6ed1a81ba5b6: Pushed
a3483ce177ce: Pushed
ce6c8756685b: Pushed
30339f20ced0: Pushed
0eb22bfb707d: Pushed
a2ae92ffcd29: Pushed
1.0.0: digest: sha256:44a137e54a48e52252ff9ce64714de8311d206f1ffaf0f6ad39dd01ab1fe0455 size: 2212


C:\Users\Administrator>docker images
REPOSITORY                                        TAG                                                     IMAGE ID       CREATED             SIZE
fandf/fdf-test                                    1.0.0                                                   1a4b935b4d57   About an hour ago   684MB
registry.cn-chengdu.aliyuncs.com/fandf/k8s-test   1.0.0                                                   1a4b935b4d57   About an hour ago   684MB
java                                              8                                                       d23bdf5b1b1b   6 years ago         643MB

C:\Users\Administrator>
```
删除掉本地镜像，从阿里云拉取镜像并运行

```shell
C:\Users\Administrator>docker images
REPOSITORY                                        TAG                                                     IMAGE ID       CREATED         SIZE
fandf/fdf-test                                    1.0.0                                                   1a4b935b4d57   2 hours ago     684MB
registry.cn-chengdu.aliyuncs.com/fandf/k8s-test   1.0.0                                                   1a4b935b4d57   2 hours ago     684MB
java                                              8                                                       d23bdf5b1b1b   6 years ago     643MB

C:\Users\Administrator>docker rmi registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0
Untagged: registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0
Untagged: registry.cn-chengdu.aliyuncs.com/fandf/k8s-test@sha256:44a137e54a48e52252ff9ce64714de8311d206f1ffaf0f6ad39dd01ab1fe0455

C:\Users\Administrator>docker images
REPOSITORY                           TAG                                                     IMAGE ID       CREATED         SIZE
fandf/fdf-test                       1.0.0                                                   1a4b935b4d57   3 hours ago     684MB
java                                 8                                                       d23bdf5b1b1b   6 years ago     643MB

C:\Users\Administrator>docker pull registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0
1.0.0: Pulling from fandf/k8s-test
Digest: sha256:44a137e54a48e52252ff9ce64714de8311d206f1ffaf0f6ad39dd01ab1fe0455
Status: Downloaded newer image for registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0
registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0

C:\Users\Administrator>docker images
REPOSITORY                                        TAG                                                     IMAGE ID       CREATED         SIZE
fandf/fdf-test                                    1.0.0                                                   1a4b935b4d57   3 hours ago     684MB
registry.cn-chengdu.aliyuncs.com/fandf/k8s-test   1.0.0                                                   1a4b935b4d57   3 hours ago     684MB
java                                              8                                                       d23bdf5b1b1b   6 years ago     643MB

C:\Users\Administrator>docker run -d -p 9001:9001 registry.cn-chengdu.aliyuncs.com/fandf/k8s-test:1.0.0
495b53c2b820bccd7ec34737a58604838cdbe8e0030b9eed57c63d5f81404bd2

C:\Users\Administrator>curl 127.0.0.1:9001/api/v1/user/name
zhangsan
```

