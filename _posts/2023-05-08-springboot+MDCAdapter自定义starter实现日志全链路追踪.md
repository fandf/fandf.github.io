---
layout: post
title: springboot+MDCAdapter自定义starter实现日志全链路追踪
date: 2023-05-05
tags: [链路追踪,springboot]
description: 打工人太难了
---



## MDC
MDC（Mapped Diagnostic Context，映射调试上下文）是日志系统提供的一种方便在多线程条件下记录日志的功能

## 使用场景
一个常用的场景就是Web服务器中给每个请求都分配一个独特的请求id，所有的日志都会打印这个请求id，这样一个请求下的所有日志信息都可以很方便的找到。

## MDCAdapter
```java
package org.slf4j.spi;  
  
import java.util.Map;  
  
public interface MDCAdapter {  
    void put(String var1, String var2);  

    String get(String var1);  

    void remove(String var1);  

    void clear();  

    Map<String, String> getCopyOfContextMap();  

    void setContextMap(Map<String, String> var1);  
}
```
MDCAdapter其实就是做一些简单的kv操作

## 自定义log-starter

整体结构


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68bad93c0e2e4a6188eff7fbcb254284~tplv-k3u1fbpfcp-watermark.image?)

### pom引入依赖
```xml
<dependencies>  
    <dependency>
        <groupId>org.projectlombok</groupId>  
        <artifactId>lombok</artifactId>  
        <version>1.18.24</version>  
        </dependency>
    <dependency>  
        <groupId>org.slf4j</groupId>  
        <artifactId>slf4j-api</artifactId>  
        <version>1.7.34</version>
    </dependency>  
    <dependency>  
        <groupId>ch.qos.logback</groupId>  
        <artifactId>logback-core</artifactId>  
    </dependency>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter</artifactId>  
    </dependency>  
    <!-- feign传递traceId -->
    <dependency>  
        <groupId>org.springframework.cloud</groupId>  
        <artifactId>spring-cloud-context</artifactId>  
    </dependency>  
    <dependency>  
        <groupId>javax.servlet</groupId>  
        <artifactId>javax.servlet-api</artifactId>  
        <version>4.0.1</version>
        <!-- optional=true 子项目不会包含这个依赖 -->  
        <optional>true</optional>  
    </dependency>  
    <!-- web传递traceId -->
    <dependency>  
        <groupId>org.springframework</groupId>  
        <artifactId>spring-web</artifactId>  
        <!-- optional=true 子项目不会包含这个依赖 -->  
        <optional>true</optional>  
    </dependency>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-configuration-processor</artifactId>  
        <optional>true</optional>  
    </dependency>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-jdbc</artifactId>  
        <optional>true</optional>  
    </dependency>  
    <dependency>  
        <groupId>cn.hutool</groupId>  
        <artifactId>hutool-all</artifactId>  
    </dependency>  
    <!-- TransmittableThreadLocal实现父子线程之间的数据传递 -->
    <dependency>  
        <groupId>com.alibaba</groupId>  
        <artifactId>transmittable-thread-local</artifactId>  
        <version>2.13.0</version>
    </dependency>  
    <!-- feign传递traceId -->
    <dependency>  
        <groupId>io.github.openfeign</groupId>  
        <artifactId>feign-core</artifactId>  
        <optional>true</optional>  
    </dependency>  
    <!-- dubbo传递traceId -->
    <dependency>  
        <groupId>org.apache.dubbo</groupId>  
        <artifactId>dubbo</artifactId>  
        <optional>true</optional>  
    </dependency>  
    <dependency>  
        <groupId>com.github.xiaoymin</groupId>  
        <artifactId>knife4j-spring-boot-starter</artifactId>  
    </dependency>  
    <dependency>  
        <groupId>net.logstash.logback</groupId>  
        <artifactId>logstash-logback-encoder</artifactId>  
        <version>5.3</version>
    </dependency>  
</dependencies>
```
#### TtlMDCAdapter
重构{@link LogbackMDCAdapter}类，搭配TransmittableThreadLocal实现父子线程之间的数据传递

新建包org.slf4j(注意此包名不可更改)

```java
package org.slf4j;  
  
import ch.qos.logback.classic.util.LogbackMDCAdapter;  
import com.alibaba.ttl.TransmittableThreadLocal;  
import org.slf4j.spi.MDCAdapter;  
  
import java.util.Collections;  
import java.util.HashMap;  
import java.util.Map;  
import java.util.Set;  
  
/**  
* 重构{@link LogbackMDCAdapter}类，搭配TransmittableThreadLocal实现父子线程之间的数据传递  
* @author fandongfeng  
* @date 2022/6/26 12:33  
*/  
  
public class TtlMDCAdapter implements MDCAdapter {  
  
    private final ThreadLocal<Map<String, String>> copyOnInheritThreadLocal = new TransmittableThreadLocal<>();  

    private static final int WRITE_OPERATION = 1;  
    private static final int MAP_COPY_OPERATION = 2;  

    static TtlMDCAdapter mtcMDCAdapter;  

    /**  
    * keeps track of the last operation performed  
    */  
    private final ThreadLocal<Integer> lastOperation = new ThreadLocal<>();  

    static {  
        mtcMDCAdapter = new TtlMDCAdapter();  
        //包名必须一致，否则这里非public不能赋值  
        MDC.mdcAdapter = mtcMDCAdapter;  
    }  

    public static MDCAdapter getInstance() {  
        return mtcMDCAdapter;  
    }  

    private Integer getAndSetLastOperation(int op) {  
        Integer lastOp = lastOperation.get();  
        lastOperation.set(op);  
        return lastOp;  
    }  

    private static boolean wasLastOpReadOrNull(Integer lastOp) {  
        return lastOp == null || lastOp == MAP_COPY_OPERATION;  
    }  

    private Map<String, String> duplicateAndInsertNewMap(Map<String, String> oldMap) {  
        Map<String, String> newMap = Collections.synchronizedMap(new HashMap<>());  
        if (oldMap != null) {  
            // we don't want the parent thread modifying oldMap while we are  
            // iterating over it  
            synchronized (oldMap) {  
                newMap.putAll(oldMap);  
            }  
    }  

    copyOnInheritThreadLocal.set(newMap);  
        return newMap;  
    }  

    @Override  
    public void put(String key, String val) {  
        if (key == null) {  
            throw new IllegalArgumentException("key cannot be null");  
        }  

        Map<String, String> oldMap = copyOnInheritThreadLocal.get();  
        Integer lastOp = getAndSetLastOperation(WRITE_OPERATION);  

        if (wasLastOpReadOrNull(lastOp) || oldMap == null) {  
            Map<String, String> newMap = duplicateAndInsertNewMap(oldMap);  
            newMap.put(key, val);  
        } else {  
            oldMap.put(key, val);  
        }  
    }  

    @Override  
    public String get(String key) {  
        final Map<String, String> map = copyOnInheritThreadLocal.get();  
        if ((map != null) && (key != null)) {  
            return map.get(key);  
        } else {  
            return null;  
        }  
    }  

    @Override  
    public void remove(String key) {  
        if (key == null) {  
            return;  
        }  
        Map<String, String> oldMap = copyOnInheritThreadLocal.get();  
        if (oldMap == null) {  
            return;  
        }  

        Integer lastOp = getAndSetLastOperation(WRITE_OPERATION);  

        if (wasLastOpReadOrNull(lastOp)) {  
            Map<String, String> newMap = duplicateAndInsertNewMap(oldMap);  
            newMap.remove(key);  
        } else {  
            oldMap.remove(key);  
        }  
    }  

    @Override  
    public void clear() {  
        lastOperation.set(WRITE_OPERATION);  
        copyOnInheritThreadLocal.remove();  
    }  

    @Override  
    public Map<String, String> getCopyOfContextMap() {  
        Map<String, String> hashMap = copyOnInheritThreadLocal.get();  
        if (hashMap == null) {  
            return null;  
        } else {  
            return new HashMap<>(hashMap);  
        }  
    }  

    @Override  
    public void setContextMap(Map<String, String> contextMap) {  
        lastOperation.set(WRITE_OPERATION);  

        Map<String, String> newMap = Collections.synchronizedMap(new HashMap<>());  
        newMap.putAll(contextMap);  

        // the newMap replaces the old one for serialisation's sake  
        copyOnInheritThreadLocal.set(newMap);  
    }  


    /**  
    * Get the current thread's MDC as a map. This method is intended to be used  
    * internally.  
    */  
    public Map<String, String> getPropertyMap() {  
        lastOperation.set(MAP_COPY_OPERATION);  
        return copyOnInheritThreadLocal.get();  
    }  

    /**  
    * Returns the keys in the MDC as a {@link Set}. The returned value can be  
    * null.  
    */  
    public Set<String> getKeys() {  
        Map<String, String> map = getPropertyMap();  

        if (map != null) {  
            return map.keySet();  
        } else {  
            return null;  
        }  
    }  
  
}
```

### 新建TtlMDCAdapterInitializer
初始化TtlMDCAdapter实例，并替换MDC中的adapter对象

```java
package com.fandf.log.config;  
  
import org.slf4j.TtlMDCAdapter;  
import org.springframework.context.ApplicationContextInitializer;  
import org.springframework.context.ConfigurableApplicationContext;  
  
/**  
* 初始化TtlMDCAdapter实例，并替换MDC中的adapter对象  
* @author fandongfeng  
* @date 2022/6/26 12:32  
*/  
  
public class TtlMDCAdapterInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {  
    @Override  
    public void initialize(ConfigurableApplicationContext applicationContext) {  
        //加载TtlMDCAdapter实例  
        TtlMDCAdapter.getInstance();  
    }  
}
```

### 新建MDCTraceUtils
日志追踪工具类

```java
package com.fandf.log.trace;  
  
import cn.hutool.core.util.IdUtil;  
import cn.hutool.core.util.StrUtil;  
import com.alibaba.ttl.TransmittableThreadLocal;  
import org.slf4j.MDC;  
  
import java.util.concurrent.atomic.AtomicInteger;  
  
/**  
* 日志追踪工具类  
* @author fandongfeng  
* @date 2022/6/26 20:54  
*/  
  
public class MDCTraceUtils {  
  
    /**  
    * 追踪id的名称  
    */  
    public static final String KEY_TRACE_ID = "traceId";  
    /**  
    * 块id的名称  
    */  
    public static final String KEY_SPAN_ID = "spanId";  

    /**  
    * 日志链路追踪id信息头  
    */  
    public static final String TRACE_ID_HEADER = "x-traceId-header";  
    /**  
    * 日志链路块id信息头  
    */  
    public static final String SPAN_ID_HEADER = "x-spanId-header";  

    /**  
    * filter的优先级，值越低越优先  
    */  
    public static final int FILTER_ORDER = -1;  

    private static final TransmittableThreadLocal<AtomicInteger> spanNumber = new TransmittableThreadLocal<>();  

    /**  
    * 创建traceId并赋值MDC  
    */  
    public static void addTrace() {  
        String traceId = createTraceId();  
        MDC.put(KEY_TRACE_ID, traceId);  
        MDC.put(KEY_SPAN_ID, "0");  
        initSpanNumber();  
    }  

    /**  
    * 赋值MDC  
    */  
    public static void putTrace(String traceId, String spanId) {  
        MDC.put(KEY_TRACE_ID, traceId);  
        MDC.put(KEY_SPAN_ID, spanId);  
        initSpanNumber();  
    }  

    /**  
    * 获取MDC中的traceId值  
    */  
    public static String getTraceId() {  
        return MDC.get(KEY_TRACE_ID);  
    }  
    /**  
    * 获取MDC中的spanId值  
    */  
    public static String getSpanId() {  
        return MDC.get(KEY_SPAN_ID);  
    }  

    /**  
    * 清除MDC的值  
    */  
    public static void removeTrace() {  
        MDC.remove(KEY_TRACE_ID);  
        MDC.remove(KEY_SPAN_ID);  
        spanNumber.remove();  
    }  

    /**  
    * 创建traceId  
    */  
    public static String createTraceId() {  
        return IdUtil.getSnowflake().nextIdStr();  
    }  

    public static String getNextSpanId() {  
        return StrUtil.format("{}.{}", getSpanId(), spanNumber.get().incrementAndGet());  
    }  

    private static void initSpanNumber() {  
        spanNumber.set(new AtomicInteger(0));  
    }  
  
}
```
### 配置类 TraceProperties
日志链路追踪配置
```java
package com.fandf.log.properties;  
  
import lombok.Getter;  
import lombok.Setter;  
import org.springframework.boot.context.properties.ConfigurationProperties;  
import org.springframework.cloud.context.config.annotation.RefreshScope;  
  
/**  
* 日志链路追踪配置  
* @author fandongfeng  
* @date 2022/6/26 20:53  
*/  
@Setter  
@Getter  
@ConfigurationProperties(prefix = "fdf.trace")  
@RefreshScope  
public class TraceProperties {  
  
    /**  
    * 是否开启日志链路追踪  
    */  
    private Boolean enable = false;  
  
}
```

### WebTraceFilter
web拦截器，传递traceId
```java
package com.fandf.log.trace;  
  
import cn.hutool.core.util.StrUtil;  
import com.fandf.log.properties.TraceProperties;  
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;  
import org.springframework.core.annotation.Order;  
import org.springframework.stereotype.Component;  
import org.springframework.web.filter.OncePerRequestFilter;  
  
import javax.annotation.Resource;  
import javax.servlet.FilterChain;  
import javax.servlet.ServletException;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import java.io.IOException;  
  
/**  
* web拦截器，传递traceId  
* @author fandongfeng  
* @date 2022/6/26 20:57  
*/  
@Component  
@ConditionalOnClass(value = {HttpServletRequest.class, OncePerRequestFilter.class})  
@Order(value = MDCTraceUtils.FILTER_ORDER)  
public class WebTraceFilter extends OncePerRequestFilter {  
  
    @Resource  
    private TraceProperties traceProperties;  

    @Override  
    protected boolean shouldNotFilter(HttpServletRequest request) {  
        return !traceProperties.getEnable();  
    }  

    @Override  
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,  
        FilterChain filterChain) throws ServletException, IOException {  
        try {  
            String traceId = request.getHeader(MDCTraceUtils.TRACE_ID_HEADER);  
            String spanId = request.getHeader(MDCTraceUtils.SPAN_ID_HEADER);  
            if (StrUtil.isEmpty(traceId)) {  
                MDCTraceUtils.addTrace();  
            } else {  
                MDCTraceUtils.putTrace(traceId, spanId);  
            }  
            filterChain.doFilter(request, response);  
        } finally {  
            MDCTraceUtils.removeTrace();  
        }  
    }  
}
```

### FeignTraceConfig
feign拦截器，传递traceId

```java
package com.fandf.log.trace;  
  
import cn.hutool.core.util.StrUtil;  
import com.fandf.log.properties.TraceProperties;  
import feign.RequestInterceptor;  
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
  
import javax.annotation.Resource;  
  
/**  
* feign拦截器，传递traceId  
* @author fandongfeng  
* @date 2022/6/26 20:52  
*/  
@Configuration  
@ConditionalOnClass(value = {RequestInterceptor.class})  
public class FeignTraceConfig {  
  
    @Resource  
    private TraceProperties traceProperties;  

    @Bean  
    public RequestInterceptor feignTraceInterceptor() {  
        return template -> {  
            if (traceProperties.getEnable()) {  
                //传递日志traceId  
                String traceId = MDCTraceUtils.getTraceId();  
                if (StrUtil.isNotEmpty(traceId)) {  
                    template.header(MDCTraceUtils.TRACE_ID_HEADER, traceId);  
                    template.header(MDCTraceUtils.SPAN_ID_HEADER, MDCTraceUtils.getNextSpanId());  
                }  
            }  
        };  
    }  
  
}
```

### DubboTraceFilter
Dubbo过滤器，传递traceId
```java
package com.fandf.log.trace;  
  
import cn.hutool.core.util.StrUtil;  
import org.apache.dubbo.common.constants.CommonConstants;  
import org.apache.dubbo.common.extension.Activate;  
import org.apache.dubbo.rpc.*;  
  
/**  
* Dubbo拦截器，传递traceId  
* @author fandongfeng  
* @date 2022/6/26 20:59  
*/  
@Activate(group = {CommonConstants.PROVIDER, CommonConstants.CONSUMER}, order = MDCTraceUtils.FILTER_ORDER)  
public class DubboTraceFilter implements Filter {  
  
    /**  
    * 服务消费者：传递traceId给下游服务  
    * 服务提供者：获取traceId并赋值给MDC  
    */  
    @Override  
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {  
        boolean isProviderSide = RpcContext.getContext().isProviderSide();  
        if (isProviderSide) { //服务提供者逻辑  
            String traceId = invocation.getAttachment(MDCTraceUtils.KEY_TRACE_ID);  
            String spanId = invocation.getAttachment(MDCTraceUtils.KEY_SPAN_ID);  
            if (StrUtil.isEmpty(traceId)) {  
                MDCTraceUtils.addTrace();  
            } else {  
                MDCTraceUtils.putTrace(traceId, spanId);  
            }  
        } else { //服务消费者逻辑  
            String traceId = MDCTraceUtils.getTraceId();  
            if (StrUtil.isNotEmpty(traceId)) {  
                invocation.setAttachment(MDCTraceUtils.KEY_TRACE_ID, traceId);  
                invocation.setAttachment(MDCTraceUtils.KEY_SPAN_ID, MDCTraceUtils.getNextSpanId());  
            }  
        }  
        try {  
            return invoker.invoke(invocation);  
        } finally {  
            if (isProviderSide) {  
                MDCTraceUtils.removeTrace();  
            }  
        }  
    }  
}
```

### LogAutoConfigure
自动注入类

```java
@ComponentScan  
@EnableConfigurationProperties({TraceProperties.class})  
public class LogAutoConfigure {
}

```

### com.alibaba.dubbo.rpc.Filter

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1705027db5c844cb85798a225c6c1795~tplv-k3u1fbpfcp-watermark.image?)
```
logTrace=com.fandf.log.trace.DubboTraceFilter
```

### spring.factories

```java
org.springframework.context.ApplicationContextInitializer=\  
com.fandf.log.config.TtlMDCAdapterInitializer  
  
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\  
com.fandf.log.LogAutoConfigure
```

### logback.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<configuration>  
    <contextName>${APP_NAME}</contextName>  
    <springProperty name="APP_NAME" scope="context" source="spring.application.name"/>  
    <springProperty name="LOG_FILE" scope="context" source="logging.file" defaultValue="./logs/application/${APP_NAME}"/>  
    <springProperty name="LOG_POINT_FILE" scope="context" source="logging.file" defaultValue="./logs/point"/>  
    <springProperty name="LOG_AUDIT_FILE" scope="context" source="logging.file" defaultValue="./logs/audit"/>  
    <springProperty name="LOG_MAX_FILESIZE" scope="context" source="logback.filesize" defaultValue="50MB"/>  
    <springProperty name="LOG_FILE_MAX_DAY" scope="context" source="logback.filemaxday" defaultValue="7"/>  
    <springProperty name="ServerIP" scope="context" source="spring.cloud.client.ip-address" defaultValue="0.0.0.0"/>  
    <springProperty name="ServerPort" scope="context" source="server.port" defaultValue="0000"/>  
    <!--LogStash访问host-->  
    <springProperty name="LOG_STASH_HOST" scope="context" source="logstash.host" defaultValue="localhost"/>  

    <!-- 彩色日志 -->  
    <!-- 彩色日志依赖的渲染类 -->  
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />  
    <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />  
    <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />  

    <!-- 彩色日志格式 -->  
    <property name="CONSOLE_LOG_PATTERN"  
    value="[${APP_NAME}:${ServerIP}:${ServerPort}] %clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(%level){blue} %clr(${PID}){magenta} %clr([%X{traceId}-%X{spanId}]){yellow} %clr([%thread]){orange} %clr(%-40.40logger{39}){cyan} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}" />  
    <property name="CONSOLE_LOG_PATTERN_NO_COLOR" value="[${APP_NAME}:${ServerIP}:${ServerPort}] %d{yyyy-MM-dd HH:mm:ss.SSS} %level ${PID} [%X{traceId}-%X{spanId}] [%thread] %-40.40logger{39} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}" />  

    <!-- 控制台日志 -->  
    <appender name="StdoutAppender" class="ch.qos.logback.core.ConsoleAppender">  
        <withJansi>true</withJansi>  
        <encoder>  
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>  
            <charset>UTF-8</charset>  
        </encoder>  
    </appender>  
    <!-- 按照每天生成常规日志文件 -->  
    <appender name="FileAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">  
        <file>${LOG_FILE}/${APP_NAME}.log</file>  
        <encoder>  
            <pattern>${CONSOLE_LOG_PATTERN_NO_COLOR}</pattern>  
            <charset>UTF-8</charset>  
        </encoder>  
        <!-- 基于时间的分包策略 -->  
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">  
            <fileNamePattern>${LOG_FILE}/${APP_NAME}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>  
            <!--保留时间,单位:天-->  
            <maxHistory>${LOG_FILE_MAX_DAY}</maxHistory>  
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">  
                <maxFileSize>${LOG_MAX_FILESIZE}</maxFileSize>  
            </timeBasedFileNamingAndTriggeringPolicy>  
        </rollingPolicy>  
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">  
            <level>INFO</level>  
        </filter>  
    </appender>  



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
                    "service": "${APP_NAME}",  
                    "env":"${Environment}",  
                    "ip": "${ServerIP}:${ServerPort}",  
                    "class": "%logger",  
                    "level": "%level",  
                    "traceId": "%X{traceId}",  
                    "thread": "%thread",  
                    "spanId": "%X{spanId}",  
                    "message": "%message",  
                    "exception":"%exception"  
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


   
    <appender name="file_async" class="ch.qos.logback.classic.AsyncAppender">  
        <discardingThreshold>0</discardingThreshold>  
        <appender-ref ref="FileAppender"/>  
    </appender>  
    
  
  
  
    <logger name="org.springframework.web" level="INFO"/>  
    <logger name="org.springframework" level="ERROR"/>  
    <logger name="org.apache.commons" level="ERROR"/>  
    <logger name="springfox.documentation" level="ERROR"/>  
    <logger name="com.ulisesbocchio" level="ERROR"/>  
    <logger name="com.alibaba.nacos" level="WARN"/>  
    <logger name="org.apache.ibatis" level="WARN"/>  
    <logger name="org.mybatis" level="WARN"/>  
    <logger name="org.quartz" level="ERROR"/>  
    <logger name="org.redisson" level="ERROR"/>  
    <logger name="io.netty" level="ERROR"/>  
    <logger name="com.baomidou.mybatisplus" level="ERROR"/>  
    <!--myibatis log configure-->  
    <logger name="com.apache.ibatis" level="TRACE"/>  
    <logger name="com.zaxxer.hikari" level="ERROR"/>  
    <logger name="com.alibaba.nacos" level="ERROR"/>  
    <logger name="java.sql.Connection" level="DEBUG"/>  
    <logger name="java.sql.Statement" level="DEBUG"/>  
    <logger name="java.sql.PreparedStatement" level="DEBUG"/>  
  
  
    <root level="INFO">  
        <appender-ref ref="StdoutAppender"/>  
        <appender-ref ref="file_async"/>  
    </root>  
</configuration>
```

## springboot项目集成log-starter

pom引入依赖
```java
<dependencies>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-web</artifactId>  
    </dependency>  
    <dependency>  
        <groupId>com.fandf</groupId>  
        <artifactId>log-starter</artifactId>  
    </dependency>  
</dependencies>
```

application.yml

```yaml
spring:  
  application:  
    name: fdf-test
server:  
  port: 9002  
fdf:  
  trace:  
    enable: true
```

UserController

```java
package com.fandf.test.controller;  
  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.web.bind.annotation.GetMapping;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RestController;  
  
import javax.annotation.Resource;  
  
/**  
* @author fandongfeng  
*/  
@RestController  
@RequestMapping("/api/v1/user")  
@Slf4j  
public class UserController {  
    
  
  
    @GetMapping("name")  
    public String getName() {  
        log.info("收到请求信息 {}", System.currentTimeMillis());  
        return "好好写技术";  
    }  
  
  
  
}
```
启动项目，访问接口  127.0.0.1:9002/api/v1/user/name

查看控制台输出

```java

[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:25.206 INFO 4402 [-] [http-nio-9002-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       Initializing Spring DispatcherServlet 'dispatcherServlet'
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:25.206 INFO 4402 [-] [http-nio-9002-exec-1] o.s.web.servlet.DispatcherServlet        Initializing Servlet 'dispatcherServlet'
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:25.207 INFO 4402 [-] [http-nio-9002-exec-1] o.s.web.servlet.DispatcherServlet        Completed initialization in 1 ms
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:25.254 INFO 4402 [1653388221926813696-0] [http-nio-9002-exec-1] c.fandf.test.controller.UserController   收到请求信息 1683033445254
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:25.288 DEBUG 4402 [1653388221926813696-0] [http-nio-9002-exec-1] com.fandf.log.aspect.WebLogAspect        {"startTime":1683033445244,"spendTime":12,"basePath":"http://127.0.0.1:9002","uri":"/api/v1/user/name","url":"http://127.0.0.1:9002/api/v1/user/name","method":"GET","result":"好好写技术"}
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:27.323 INFO 4402 [1653388230697103360-0] [http-nio-9002-exec-3] c.fandf.test.controller.UserController   收到请求信息 1683033447323
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:27.327 DEBUG 4402 [1653388230697103360-0] [http-nio-9002-exec-3] com.fandf.log.aspect.WebLogAspect        {"startTime":1683033447323,"spendTime":1,"basePath":"http://127.0.0.1:9002","uri":"/api/v1/user/name","url":"http://127.0.0.1:9002/api/v1/user/name","method":"GET","result":"好好写技术"}
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:27.828 INFO 4402 [1653388232811032576-0] [http-nio-9002-exec-4] c.fandf.test.controller.UserController   收到请求信息 1683033447828
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:27.829 DEBUG 4402 [1653388232811032576-0] [http-nio-9002-exec-4] com.fandf.log.aspect.WebLogAspect        {"startTime":1683033447828,"spendTime":0,"basePath":"http://127.0.0.1:9002","uri":"/api/v1/user/name","url":"http://127.0.0.1:9002/api/v1/user/name","method":"GET","result":"好好写技术"}
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:28.113 INFO 4402 [1653388234010603520-0] [http-nio-9002-exec-5] c.fandf.test.controller.UserController   收到请求信息 1683033448113
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:28.114 DEBUG 4402 [1653388234010603520-0] [http-nio-9002-exec-5] com.fandf.log.aspect.WebLogAspect        {"startTime":1683033448113,"spendTime":0,"basePath":"http://127.0.0.1:9002","uri":"/api/v1/user/name","url":"http://127.0.0.1:9002/api/v1/user/name","method":"GET","result":"好好写技术"}
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:28.291 INFO 4402 [1653388234752995328-0] [http-nio-9002-exec-6] c.fandf.test.controller.UserController   收到请求信息 1683033448291
[fdf-test:0.0.0.0:9002] 2023-05-02 21:17:28.292 DEBUG 4402 [1653388234752995328-0] [http-nio-9002-exec-6] com.fandf.log.aspect.WebLogAspect        {"startTime":1683033448291,"spendTime":0,"basePath":"http://127.0.0.1:9002","uri":"/api/v1/user/name","url":"http://127.0.0.1:9002/api/v1/user/name","method":"GET","result":"好好写技术"}

```
