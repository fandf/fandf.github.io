---
layout: post
title: RateLimiter+AOP自定义注解限流
date: 2023-04-10
tags: [限流]
description: 又是加班的一天
---

## RateLimiter简介

### springboot集成RateLimiter

pom.xml引入guava依赖

```xml
<dependency>  
    <groupId>com.google.guava</groupId>  
    <artifactId>guava</artifactId>  
    <version>31.1-jre</version>  
</dependency>
```
### RateLimiter常用api
RateLimiter.create(double permitsPerSecond)
- 指定速率，每秒产生permitsPerSecond个令牌

rateLimiter.acquire(int permits);
- 返回获取permits个令牌所需的时间

rateLimiter.tryAcquire(1);
- 获取1个令牌，返回false获取失败，true获取成功

rateLimiter.tryAcquire(1, 2, TimeUnit.SECONDS)
- 获取1个令牌，最多等待两秒。返回false获取失败，true获取成功

```java
package com.fandf.test.ratelimter;  
  
import com.google.common.util.concurrent.RateLimiter;  
  
import java.util.concurrent.TimeUnit;  
  
/**  
* @author fandongfeng  
* @date 2023/4/9 19:55  
*/  
public class RateLimiterTest {  
  
    public static void main(String[] args) {  
        //指定速率，每秒产生一个令牌  
        RateLimiter rateLimiter = RateLimiter.create(1);  
        System.out.println(rateLimiter.getRate()); 
        //修改为每秒产生2个令牌
        rateLimiter.setRate(2);  
        System.out.println(rateLimiter.getRate());  
  
        //while (true) {  
            //获取2个令牌所需要的时间  
            double acquire = rateLimiter.acquire(2);  
            //输出结果， 第一次0秒，后面每次等待接近0.5秒的时间  
            //0.0  
            //0.995198  
            //0.9897  
            //0.999413  
            //0.998272  
            //0.99202  
            //0.993757  
            //0.996198  
            //0.99523  
            //0.99532  
            //0.992674  
            System.out.println(acquire);  
       // }  
            //获取1个令牌  
            boolean result = rateLimiter.tryAcquire(1);  
            System.out.println(result);  
            //获取1个令牌,最多等待两秒  
            boolean result2 = rateLimiter.tryAcquire(1, 2, TimeUnit.SECONDS);  
            System.out.println(result2);
 
    }  
  
}
```

## RateLimiter+AOP实现自定义注解限流

### 自定义注解Limiter

```java

package com.fandf.test.ratelimter;  
  
import java.util.concurrent.TimeUnit;  
  
/**  
* 限流注解  
*  
* @author fandongfeng  
* @date 2023/4/9 20:17  
*/  
@Target(ElementType.METHOD)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented
public @interface Limiter {  
    /**
    * 不进行限流
    */
    int NOT_LIMITED = 0;  

    /**  
    * qps (每秒并发量)  
    */  
    double qps() default NOT_LIMITED;  

    /**  
    * 超时时长,默认不等待  
    */  
    int timeout() default 0;  

    /**  
    * 超时时间单位,默认毫秒  
    */  
    TimeUnit timeUnit() default TimeUnit.MILLISECONDS;  

    /**  
    * 返回错误信息  
    */  
    String msg() default "系统忙，请稍后再试";  
}

```

### 新增限流切面

pom引入aop依赖
```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-aop</artifactId>  
</dependency>
```


```java
package com.fandf.test.ratelimter;  
  
import com.google.common.util.concurrent.RateLimiter;  
import lombok.extern.slf4j.Slf4j;  
import org.aspectj.lang.ProceedingJoinPoint;  
import org.aspectj.lang.annotation.Around;  
import org.aspectj.lang.annotation.Aspect;  
import org.aspectj.lang.reflect.MethodSignature;  
import org.springframework.stereotype.Component;  
  
import java.lang.reflect.Method;  
import java.util.concurrent.ConcurrentHashMap;  
import java.util.concurrent.ConcurrentMap;  
  
/**  
* 限流切面  
* @author fandongfeng  
* @date 2023/4/9 20:19  
*/  
@Slf4j  
@Aspect  
@Component  
public class RateLimiterAspect {  
    /**  
    * key: 类全路径+方法名  
    */  
    private static final ConcurrentMap<String, RateLimiter> RATE_LIMITER_CACHE = new ConcurrentHashMap<>();  
  
  
    @Around("@within(limiter) || @annotation(limiter)")  
    public Object pointcut(ProceedingJoinPoint point, Limiter limiter) throws Throwable {  
        MethodSignature signature = (MethodSignature) point.getSignature();  
        Method method = signature.getMethod();  
        if (limiter != null && limiter.qps() > Limiter.NOT_LIMITED) {  
            double qps = limiter.qps();  
            //这个key可以根据具体需求配置,例如根据ip限制,或用户  
            String key = method.getDeclaringClass().getName() + method.getName();  
            if (RATE_LIMITER_CACHE.get(key) == null) {  
                // 初始化 QPS  
                RATE_LIMITER_CACHE.put(key, RateLimiter.create(qps));  
            }  
            // 尝试获取令牌  
            if (RATE_LIMITER_CACHE.get(key) != null && !RATE_LIMITER_CACHE.get(key).tryAcquire(limiter.timeout(), limiter.timeUnit())) {  
                log.error("触发限流操作{}", key);  
                throw new RuntimeException(limiter.msg());  
            }  
        }  
        return point.proceed();  
    }  
}
```

## 测试限流

### 新建controller

```java
package com.fandf.test.ratelimter;  
  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.web.bind.annotation.GetMapping;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RestController;  
  
/**  
* @author fandongfeng  
* @date 2023/4/9 20:25  
*/  
@RestController  
@RequestMapping("limiter")  
@Slf4j  
public class UserController {  
  
    @GetMapping  
    @Limiter(qps = 1, msg = "您已被限流！")  
        public String getUserName() {  
        String userName = "zhangsan";  
        log.info("userName = {}", userName);  
        return userName;  
    }  
  
}
```

打印结果

```text
20:51:40.867 logback [http-nio-8080-exec-5] INFO  c.f.test.ratelimter.UserController - userName = zhangsan
20:51:41.700 logback [http-nio-8080-exec-6] INFO  c.f.test.ratelimter.UserController - userName = zhangsan
20:51:42.375 logback [http-nio-8080-exec-7] INFO  c.f.test.ratelimter.UserController - userName = zhangsan
20:51:43.029 logback [http-nio-8080-exec-8] INFO  c.f.test.ratelimter.UserController - userName = zhangsan
20:51:43.647 logback [http-nio-8080-exec-9] ERROR c.f.t.ratelimter.RateLimiterAspect - 触发限流操作com.fandf.test.ratelimter.UserControllergetUserName
20:51:43.648 logback [http-nio-8080-exec-9] ERROR o.a.c.c.C.[.[.[.[dispatcherServlet] - Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: 您已被限流！] with root cause
java.lang.RuntimeException: 您已被限流！
	at com.fandf.test.ratelimter.RateLimiterAspect.pointcut(RateLimiterAspect.java:45)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs(AbstractAspectJAdvice.java:634)
	at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(AbstractAspectJAdvice.java:624)
	at org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(AspectJAroundAdvice.java:72)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:175)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:753)
	at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:97)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:753)
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:698)
	at com.fandf.test.ratelimter.UserController$$EnhancerBySpringCGLIB$$ee09d79c.getUserName(<generated>)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:205)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:150)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:117)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:895)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:808)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1067)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:963)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:655)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:764)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:227)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:117)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:117)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:117)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:197)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:540)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:135)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:357)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:382)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:895)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1732)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1191)
	at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:748)
```
