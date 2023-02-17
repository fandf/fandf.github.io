---
layout: post
title: IDEA打包docker镜像
date: 2022-07-28
tags: [docker]
description: 保持学习，保持热爱。
---

环境搭建及源码地址：https://github.com/fandf/SpringCloudLearning/tree/master/fdf-demo/nacos

### 1.pom.xml添加如下配置
````xml
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
                <mainClass>com.fandf.nacos.NacosApplication</mainClass>
            </configuration>
        </plugin>
        <plugin>
            <groupId>io.fabric8</groupId>
            <artifactId>docker-maven-plugin</artifactId>
            <version>0.40.1</version>
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
                    <!-- 绑定mvn install阶段，当执行mvn install时 就会执行docker build 和docker push-->
                    <phase>deploy</phase>
                    <goals>
                        <goal>build</goal>
                        <!--                            <goal>push</goal>-->
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
````
插件如果无法下载，可引入依赖
```
<dependency>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.40.1</version>
</dependency>
```


### 2.编写Dockerfile
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
EXPOSE 8081
ENTRYPOINT ["java","-jar","app.jar"]
````

项目maven plugins会有docker插件，点击docekr build

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a669d19ad2540bcaa9ce8aa51d1a033~tplv-k3u1fbpfcp-watermark.image?)

镜像已被上传到本地

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/722b846fceff4b3883a6c31c00a91dcd~tplv-k3u1fbpfcp-watermark.image?)

运行后，nacos也发现了该服务


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4c3cce926576468ab380e7da7a8c8d3b~tplv-k3u1fbpfcp-watermark.image?)


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34a7603af66547cc908745a29f09d3a6~tplv-k3u1fbpfcp-watermark.image?)
