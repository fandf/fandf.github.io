---
layout: post
title: springboot集成规则引擎drools
date: 2023-05-05
tags: [链路追踪,springboot]
description: 打工人太难了
---


## 规则引擎概述
规则引擎的主要思想是将应用程序中的业务决策部分分离出来，并使用预定义的语义模板编写业务决策（业务规则），由用户或开发者在需要时进行配置、管理。

## 使用场景

比如商城购物，满300减100，满500减200等等,而且这些规则有可能随时会变动的。如果实现这个需求，正常情况下我们怎么做呢？

### if...else
伪代码
```java
if(amount >= 300) {
    amount -= 100;
} else if(amount >= 500) {
    amount -= 200;
}
```

### 策略模式
伪代码
```java
interface Strategy {
    reductionAmount(int amount);
}

class Strategy1 {
    reductionAmount(int amount);
}

class Strategy2 {
    reductionAmount(int amount);
}
```

以上方式都可以实现功能，但是如果折扣经常变动呢？不同的商品有不同的规则呢？而且这些规则与代码严重耦合，每次规则变动都需要重新测试，部署。

## Drools介绍

drools是一款由JBoss组织提供的基于java语言开发的开源规则引擎，可以将复杂且多变的业务规则从硬编码中解放出来，以规则脚本的形式存放在文件或特定的存储介质中（如存放在数据库中），使得业务规则的变更不需要修改项目代码、重启服务器就可以在线上环境立即生效。

drools官网：[https://www.drools.org/](https://www.drools.org)

drools中文网：[Drools中文网 | 基于java的功能强大的开源规则引擎](http://www.drools.org.cn/)

drools源码下载地址：[https://github.com/kiegroup/drools](https://github.com/kiegroup/drools)

## springboot集成drools案例

### 业务场景
| 订单金额 | 优惠金额 |
| --- | --- |
| 100-200 | 10 |
| 200-500 | 20 |
| 500-1000 | 50 |
| 1000-2000 | 100 |
| 2000-5000 | 300 |
| 5000-10000 | 500 |
| 10000以上 | 1000 |


### 创建springboot项目
创建maven工程并导入drools相关依赖
```xml
<!-- https://mvnrepository.com/artifact/org.drools/drools-compiler -->  
<dependency>  
    <groupId>org.drools</groupId>  
    <artifactId>drools-compiler</artifactId>  
    <version>7.73.0.Final</version>  
</dependency>  
<!-- https://mvnrepository.com/artifact/org.drools/drools-mvel -->  
<dependency>  
    <groupId>org.drools</groupId>  
    <artifactId>drools-mvel</artifactId>  
    <version>7.73.0.Final</version>  
</dependency>
<!-- 测试 -->
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-test</artifactId>  
</dependency>
```
resources/META-INF/kmodule.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<kmodule xmlns="http://jboss.org/kie/6.0.0/kmodule">  
  
    <!--  
    name:指定kbase的名称，可以任意，但是需要唯一  
    packages:指定规则文件的目录，需要根据实际情况填写，否则无法加载到规则文件  
    default:指定当前kbase是否为默认  
    -->  
    <kbase name="fdf" packages="com.fandf.rules" default="true">  
        <!--  
        name:指定ksession的名称，可以任意，但需要唯一  
        default:指定当前session是否为默认  
        -->  
        <ksession name="fdf-rule" default="true"/>  
    </kbase>  
</kmodule>
```
创建实体order
```java
package com.fandf.test.entity;  
  
import lombok.Data;  
  
import java.math.BigDecimal;  
  
/**  
* @author fandongfeng  
* @date 2023/5/3 19:17  
*/  
@Data  
public class Order {  
  
  
    /**  
    * 订单优惠前价格  
    */  
    private BigDecimal originalPrice;  
    /**  
    * 订单优惠后价格  
    */  
    private BigDecimal realPrice;  
  
}
```

创建规则文件resources/rules/orderDiscount.drl
```java
// 订单优惠规则 须与kmodule.xml的package一致  
package com.fandf.rules  
import com.fandf.test.entity.Order  
import java.math.BigDecimal  
  
// 规则一：100-200 优惠 10  
rule "order_discount_1"  
    when  
        $order: Order(originalPrice >= 100 && originalPrice < 200) // 匹配模式，到规则引擎中（工作内存）查找Order对象，命名为$order  
    then  
        $order.setRealPrice($order.getOriginalPrice().subtract(BigDecimal.valueOf(10)));  
        System.out.println("成功匹配到规则一，订单金额优惠10元");  
end  
  
// 规则二：200-500 优惠 20  
rule "order_discount_2"  
    when  
        $order: Order(originalPrice >= 200 && originalPrice < 500)  
    then  
        $order.setRealPrice($order.getOriginalPrice().subtract(BigDecimal.valueOf(20)));  
        System.out.println("成功匹配到规则二，订单金额优惠20元");  
end  
  
// 规则三：500-1000 优惠 50  
rule "order_discount_3"  
    when  
        $order: Order(originalPrice >= 500 && originalPrice < 1000)  
    then  
        $order.setRealPrice($order.getOriginalPrice().subtract(BigDecimal.valueOf(50)));  
        System.out.println("成功匹配到规则三，订单金额优惠50元");  
end  
// 规则四：1000-2000 优惠 100  
rule "order_discount_4"  
    when  
        $order: Order(originalPrice >= 1000 && originalPrice < 2000)  
    then  
        $order.setRealPrice($order.getOriginalPrice().subtract(BigDecimal.valueOf(100)));  
        System.out.println("成功匹配到规则四，订单金额优惠100元");  
end  
// 规则五：2000-5000 优惠 300  
rule "order_discount_5"  
    when  
        $order: Order(originalPrice >= 2000 && originalPrice < 5000)  
    then  
        $order.setRealPrice($order.getOriginalPrice().subtract(BigDecimal.valueOf(300)));  
        System.out.println("成功匹配到规则五，订单金额优惠300元");  
end  
// 规则六：5000-10000 优惠 500  
rule "order_discount_6"  
    when  
        $order: Order(originalPrice >= 5000 && originalPrice < 10000)  
    then  
        $order.setRealPrice($order.getOriginalPrice().subtract(BigDecimal.valueOf(500)));  
        System.out.println("成功匹配到规则六，订单金额优惠500元");  
end  
// 规则七：10000以上 优惠 1000  
rule "order_discount_7"  
    when  
        $order: Order(originalPrice >= 10000)  
    then  
        $order.setRealPrice($order.getOriginalPrice().subtract(BigDecimal.valueOf(1000)));  
        System.out.println("成功匹配到规则七，订单金额优惠1000元");  
end
```

编写测试类OrderTest
```java
package com.fandf.test.entity;  
  
import com.fandf.test.Application;  
import org.junit.jupiter.api.Test;  
import org.kie.api.KieServices;  
import org.kie.api.runtime.KieContainer;  
import org.kie.api.runtime.KieSession;  
import org.springframework.boot.test.context.SpringBootTest;  
  
import java.math.BigDecimal;  
  
@SpringBootTest(classes = Application.class)  
public class OrderTest {  
  
    @Test  
    public void test(){  
        KieServices kieServices = KieServices.Factory.get();  
        // 获取Kie容器对象 默认容器对象  
        KieContainer kieContainer = kieServices.newKieClasspathContainer();  
        // 从Kie容器对象中获取会话对象（默认session对象  
        KieSession kieSession = kieContainer.newKieSession();  

        Order order = new Order();  
        order.setOriginalPrice(BigDecimal.valueOf(180));  
        // 将order对象插入工作内存  
        kieSession.insert(order);  
        // 匹配对象  
        // 激活规则，由drools框架自动进行规则匹配。若匹配成功，则执行  
        kieSession.fireAllRules();  
        System.out.println("优惠前价格：" + order.getOriginalPrice() + ",优惠后价格：" + order.getRealPrice());  

        kieSession = kieContainer.newKieSession();  
        order.setOriginalPrice(BigDecimal.valueOf(300));  
        // 将order对象插入工作内存  
        kieSession.insert(order);  
        // 匹配对象  
        // 激活规则，由drools框架自动进行规则匹配。若匹配成功，则执行  
        kieSession.fireAllRules();  
        System.out.println("优惠前价格：" + order.getOriginalPrice() + ",优惠后价格：" + order.getRealPrice());  

        kieSession = kieContainer.newKieSession();  
        order.setOriginalPrice(BigDecimal.valueOf(600));  
        // 将order对象插入工作内存  
        kieSession.insert(order);  
        // 匹配对象  
        // 激活规则，由drools框架自动进行规则匹配。若匹配成功，则执行  
        kieSession.fireAllRules();  
        System.out.println("优惠前价格：" + order.getOriginalPrice() + ",优惠后价格：" + order.getRealPrice());  

        kieSession = kieContainer.newKieSession();  
        order.setOriginalPrice(BigDecimal.valueOf(1200));  
        // 将order对象插入工作内存  
        kieSession.insert(order);  
        // 匹配对象  
        // 激活规则，由drools框架自动进行规则匹配。若匹配成功，则执行  
        kieSession.fireAllRules();  
        System.out.println("优惠前价格：" + order.getOriginalPrice() + ",优惠后价格：" + order.getRealPrice());  

        kieSession = kieContainer.newKieSession();  
        order.setOriginalPrice(BigDecimal.valueOf(3000));  
        // 将order对象插入工作内存  
        kieSession.insert(order);  
        // 匹配对象  
        // 激活规则，由drools框架自动进行规则匹配。若匹配成功，则执行  
        kieSession.fireAllRules();  
        System.out.println("优惠前价格：" + order.getOriginalPrice() + ",优惠后价格：" + order.getRealPrice());  

        kieSession = kieContainer.newKieSession();  
        order.setOriginalPrice(BigDecimal.valueOf(8000));  
        // 将order对象插入工作内存  
        kieSession.insert(order);  
        // 匹配对象  
        // 激活规则，由drools框架自动进行规则匹配。若匹配成功，则执行  
        kieSession.fireAllRules();  
        System.out.println("优惠前价格：" + order.getOriginalPrice() + ",优惠后价格：" + order.getRealPrice());  

        kieSession = kieContainer.newKieSession();  
        order.setOriginalPrice(BigDecimal.valueOf(12000));  
        // 将order对象插入工作内存  
        kieSession.insert(order);  
        // 匹配对象  
        // 激活规则，由drools框架自动进行规则匹配。若匹配成功，则执行  
        kieSession.fireAllRules();  
        System.out.println("优惠前价格：" + order.getOriginalPrice() + ",优惠后价格：" + order.getRealPrice());  



        // 关闭会话  
        kieSession.dispose();  

    }  
  
}
```
执行输出
```java
[fdf-test:0.0.0.0:9002] 2023-05-03 22:25:03.823 INFO 5536 [-] [main] o.d.c.k.b.impl.ClasspathKieProject       Found kmodule: file:/Users/dongfengfan/IdeaProjects/SpringCloudLearning/fdf-test/target/classes/META-INF/kmodule.xml
[fdf-test:0.0.0.0:9002] 2023-05-03 22:25:04.019 WARN 5536 [-] [main] o.d.c.k.b.impl.ClasspathKieProject       Unable to find pom.properties in /Users/dongfengfan/IdeaProjects/SpringCloudLearning/fdf-test/target/classes
[fdf-test:0.0.0.0:9002] 2023-05-03 22:25:04.024 INFO 5536 [-] [main] o.d.c.k.b.impl.ClasspathKieProject       Recursed up folders, found and used pom.xml /Users/dongfengfan/IdeaProjects/SpringCloudLearning/fdf-test/pom.xml
[fdf-test:0.0.0.0:9002] 2023-05-03 22:25:04.025 INFO 5536 [-] [main] o.d.c.k.b.i.InternalKieModuleProvider    Creating KieModule for artifact com.fandf:fdf-test:1.0.0
[fdf-test:0.0.0.0:9002] 2023-05-03 22:25:04.035 INFO 5536 [-] [main] o.d.c.kie.builder.impl.KieContainerImpl  Start creation of KieBase: fdf
[fdf-test:0.0.0.0:9002] 2023-05-03 22:25:04.041 WARN 5536 [-] [main] o.d.c.kie.builder.impl.KieBuilderImpl    File 'rules/orderDiscount.drl' is in folder 'rules' but declares package 'com.fandf.rules'. It is advised to have a correspondance between package and folder names.
[fdf-test:0.0.0.0:9002] 2023-05-03 22:25:05.091 INFO 5536 [-] [main] o.d.c.kie.builder.impl.KieContainerImpl  End creation of KieBase: fdf
成功匹配到规则一，订单金额优惠10元
优惠前价格：180,优惠后价格：170
成功匹配到规则二，订单金额优惠20元
优惠前价格：300,优惠后价格：280
成功匹配到规则三，订单金额优惠50元
优惠前价格：600,优惠后价格：550
成功匹配到规则四，订单金额优惠100元
优惠前价格：1200,优惠后价格：1100
成功匹配到规则五，订单金额优惠300元
优惠前价格：3000,优惠后价格：2700
成功匹配到规则六，订单金额优惠500元
优惠前价格：8000,优惠后价格：7500
成功匹配到规则七，订单金额优惠1000元
优惠前价格：12000,优惠后价格：11000
```

## Drools语法

### 规则文件

在使用Drools时非常重要的一个工作就是编写规则文件，通常规则文件的后缀为.drl。

drl是Drools Rule Language的缩写。在规则文件中编写具体的规则内容。


| 关键字 | 描述 |
| --- | --- |
| package | 包名，只限于逻辑上的管理，同一个包名下的查询或者函数可以直接调用 |
| import | 用于导入类或静态方法 |
| global | 全局变量 |
| function | 自定义函数 |
| query | 查询 |
| rule end | 规则体 |

### 规则体语法

规则体是进行业务规则判断、处理业务结果的部分。

```
rule "ruleName" 
    attributes 
    when 
        LHS
    then 
        RHS
end 
```

rule：关键字，表示规则开始，参数为规则的唯一名称。

attribute：规则属性，是rule与when之间的参数，为可选项。

when：关键字，后面跟规则的条件部分。

LHS（Left Hand Side）：是规则的条件部分的通用名称。它由零个或多个条件元素组成。如果LHS为空，则它将被视为始终为true的条件元素。

then：关键字，后面跟规则的结果部分。

RHS（Right Hand Side）：是规则的后果或行动部分的通用名称。

end：关键字，表示一个规则的结束。

### Pattern模式匹配
pattern的语法结构为：绑定变量名:Object(Field约束)

其中绑定变量名可以省略，通常绑定变量名的命名一般建议以$开始。如果定义了绑定变量名，就可以在规则体的RHS部分使用此绑定变量名来操作相应的Fact对象。Field约束部分是需要返回true或者false的0个或多个表达式。

比如我们的案例
```
rule "order_discount_1"  
    when  
        $order: Order(originalPrice >= 100 && originalPrice < 200) // 匹配模式，到规则引擎中（工作内存）查找Order对象，命名为$order  
    then  
        $order.setRealPrice($order.getOriginalPrice().subtract(BigDecimal.valueOf(10)));  
        System.out.println("成功匹配到规则一，订单金额优惠10元");  
end
```


符号           | 说明                                   |
| ------------ | ------------------------------------ |
| >            | 大于                                   |
| <            | 小于                                   |
| >=           | 大于等于                                 |
| <=           | 小于等于                                 |
| ==           | 等于                                   |
| !=           | 不等于                                  |
| contains     | 检查一个Fact对象的某个属性值是否包含一个指定的对象值         |
| not contains | 检查一个Fact对象的某个属性值是否不包含一个指定的对象值        |
| memberOf     | 判断一个Fact对象的某个属性是否在一个或多个集合中           |
| not memberOf | 判断一个Fact对象的某个属性是否不在一个或多个集合中          |
| matches      | 判断一个Fact对象的属性是否与提供的标准的Java正则表达式进行匹配  |
| not matches  | 判断一个Fact对象的属性是否不与提供的标准的Java正则表达式进行匹配



