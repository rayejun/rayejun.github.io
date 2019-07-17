---
title: Spring Boot集成Dubbo+Zookeeper实现分布式
date: 2019-07-17 20:19:38
tags: 
 - Spring Boot 
 - Dubbo
categories: 
 - [Spring Boot] 
 - [Dubbo]
---
本文将介绍基于XML方式集成

## 引用dubbo和zkclient包，这里展示maven方式的引用
dubbo:
```
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>dubbo</artifactId>
	<version>2.6.0</version>
</dependency>
```
zkclient:
```
<dependency>
	<groupId>com.101tec</groupId>
	<artifactId>zkclient</artifactId>
	<version>0.10</version>
</dependency>
```

## 提供者dubbo-provider.xml文件配置
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="${dubbo.application.name}"/>
    <!-- 使用 zookeeper 注册中心暴露服务 -->
    <dubbo:registry address="${dubbo.registry.address}"/>
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="${dubbo.protocol.port}"/>
    <!-- 调用超时 -->
    <dubbo:provider timeout="5000"/>
    <!-- 监控中心 -->

    <!--<dubbo:monitor protocol="registry"/>-->

    <!-- 要暴露的服务接口 -->
    <dubbo:service interface="com.test.api.service.DemoService" ref="demoServiceImpl"/>
</beans>
```

## 消费者dubbo-consumer.xml文件配置
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <!-- 消费方应用信息，用于计算依赖关系 -->
    <dubbo:application name="${dubbo.application.name}"/>
    <!-- 使用 zookeeper 注册中心发现服务 -->
    <dubbo:registry address="${dubbo.registry.address}"/>
    <!-- 调用超时 -->
    <dubbo:provider timeout="5000"/>
    <!-- 监控中心 -->

    <!--<dubbo:monitor protocol="registry"/>-->

    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
    <dubbo:reference id="demoService" interface="com.test.api.service.demoService"/>
</beans>
```

## 配置application.properties
提供者:
```
dubbo.application.name=${spring.application.name}
dubbo.config.location=classpath:dubbo/dubbo-provider.xml
dubbo.registry.address=zookeeper://127.0.0.1:2181
dubbo.protocol.port=20880
```
消费者:
```
dubbo.application.name=${spring.application.name}
dubbo.config.location=classpath:dubbo/dubbo-consumer.xml
dubbo.registry.address=zookeeper://127.0.0.1:2181
```

用注解方式分别加载dubbo-provider.xml和dubbo-consumer.xml配置
```
@SpringBootApplication
@ImportResource("${dubbo.config.location}")
public class TestApplication {
   public static void main(String[] args) {
      SpringApplication.run(TestApplication.class, args);
   }
}
```

## 启动zookeeper注册中心
在zookeeper conf目录下新建zoo.cfg配置文件，内容如下:
```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
clientPort=2181
```
启动zookeeper

到此结束，先启动服务提供者，再启动服务消费者，消费者就可以调用提供者的服务了。
