---
title: Spring Boot集成Apollo配置中心
tags: 
 - Spring Boot 
 - Apollo
categories: 
 - [Spring Boot] 
 - [Apollo]
---
## Apollo简介
Apollo（阿波罗）是携程框架部门研发的开源配置管理中心，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性。

Apollo支持4个维度管理Key-Value格式的配置：

application (应用)
environment (环境)
cluster (集群)
namespace (命名空间)
同时，Apollo基于开源模式开发，开源地址：https://github.com/ctripcorp/apollo

### 特性:
* 统一管理不同环境、不同集群的配置
* 配置修改实时生效（热发布）
* 版本发布管理
* 灰度发布
* 权限管理、发布审核、操作审计
* 客户端配置信息监控

## 部署Apollo配置中心
[参考文档](https://github.com/ctripcorp/apollo/wiki/%E5%88%86%E5%B8%83%E5%BC%8F%E9%83%A8%E7%BD%B2%E6%8C%87%E5%8D%97)

Apollo配置管理平台如下:
![配置管理平台图](apollo-manage.png)
可以在管理平台中管理自己的配置

## 新建spring boot项目添加apollo依赖
以maven为例：
```
<dependency>
	<groupId>com.ctrip.framework.apollo</groupId>
	<artifactId>apollo-client</artifactId>
	<version>1.4.0</version>
</dependency>
```

## 添加@EnableApolloConfig注解，就可以通过@Value获取配置信息
```
@SpringBootApplication
@EnableApolloConfig
public class ApolloDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApolloDemoApplication.class, args);
    }

}

@Component
public class SomeBean {
    //timeout的值会自动更新
    @Value("${request.timeout:200}")
    private int timeout;
}
```

apollo支持配置修改后实时生效，灰度发布，多环境、分集群管理配置

关于更多Apollo信息请参考[Apollo配置中心介绍](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E4%BB%8B%E7%BB%8D)
