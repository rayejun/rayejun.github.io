---
title: Spring Boot集成Elastic-Job实现分布式任务调度系统
date: 2019-07-26 15:01:02
tags:
---
## Elastic-Job
Elastic-Job是是由当当网基于quartz二次开发之后的分布式调度解决方案 ， 由两个相对独立的子项目Elastic-Job-Lite和Elastic-Job-Cloud组成 。
Elastic-Job主要的设计理念是无中心化的分布式定时调度框架，思路来源于Quartz的基于数据库的高可用方案。但数据库没有分布式协调功能，所以在高可用方案的基础上增加了弹性扩容和数据分片的思路，以便于更大限度的利用分布式服务器的资源。
Elastic-Job-Lite定位为轻量级无中心化解决方案，使用jar包的形式提供分布式任务的协调服务。
Elastic-Job-Cloud采用自研Mesos Framework的解决方案，额外提供资源治理、应用分发以及进程隔离等功能。
开源地址：https://github.com/elasticjob
### 功能列表
* 分布式调度协调
* 弹性扩容缩容
* 失效转移
* 错过执行作业重触发
* 作业分片一致性，保证同一分片在分布式环境中仅一个执行实例
* 自诊断并修复分布式不稳定造成的问题
* 支持并行调度
* 支持作业生命周期操作
* 丰富的作业类型
* Spring整合以及命名空间提供
* 运维平台

## 简介
lastic-Job-Lite定位为轻量级无中心化解决方案，使用jar包的形式提供最轻量级的分布式任务的协调服务，外部依赖仅Zookeeper。
## 基本概念
### 分片概念
任务的分布式执行，需要将一个任务拆分为多个独立的任务项，然后由分布式的服务器分别执行某一个或几个分片项。

例如：有一个遍历数据库某张表的作业，现有2台服务器。为了快速的执行作业，那么每台服务器应执行作业的50%。 为满足此需求，可将作业分成2片，每台服务器执行1片。作业遍历数据的逻辑应为：服务器A遍历ID以奇数结尾的数据；服务器B遍历ID以偶数结尾的数据。 如果分成10片，则作业遍历数据的逻辑应为：每片分到的分片项应为ID%10，而服务器A被分配到分片项0,1,2,3,4；服务器B被分配到分片项5,6,7,8,9，直接的结果就是服务器A遍历ID以0-4结尾的数据；服务器B遍历ID以5-9结尾的数据。
### 分片项与业务处理解耦
Elastic-Job并不直接提供数据处理的功能，框架只会将分片项分配至各个运行中的作业服务器，开发者需要自行处理分片项与真实数据的对应关系。
### 个性化参数的适用场景
个性化参数即shardingItemParameter，可以和分片项匹配对应关系，用于将分片项的数字转换为更加可读的业务代码。

例如：按照地区水平拆分数据库，数据库A是北京的数据；数据库B是上海的数据；数据库C是广州的数据。 如果仅按照分片项配置，开发者需要了解0表示北京；1表示上海；2表示广州。 合理使用个性化参数可以让代码更可读，如果配置为0=北京,1=上海,2=广州，那么代码中直接使用北京，上海，广州的枚举值即可完成分片项和业务逻辑的对应关系。
## 核心理念
### 分布式调度
Elastic-Job-Lite并无作业调度中心节点，而是基于部署作业框架的程序在到达相应时间点时各自触发调度。

注册中心仅用于作业注册和监控信息存储。而主作业节点仅用于处理分片和清理等功能。
### 作业高可用
Elastic-Job-Lite提供最安全的方式执行作业。将分片总数设置为1，并使用多于1台的服务器执行作业，作业将会以1主n从的方式执行。

一旦执行作业的服务器崩溃，等待执行的服务器将会在下次作业启动时替补执行。开启失效转移功能效果更好，可以保证在本次作业执行时崩溃，备机立即启动替补执行。
### 最大限度利用资源
Elastic-Job-Lite也提供最灵活的方式，最大限度的提高执行作业的吞吐量。将分片项设置为大于服务器的数量，最好是大于服务器倍数的数量，作业将会合理的利用分布式资源，动态的分配分片项。

例如：3台服务器，分成10片，则分片项分配结果为服务器A=0,1,2;服务器B=3,4,5;服务器C=6,7,8,9。 如果服务器C崩溃，则分片项分配结果为服务器A=0,1,2,3,4;服务器B=5,6,7,8,9。在不丢失分片项的情况下，最大限度的利用现有资源提高吞吐量。

## 整体架构图
{% asset_img elastic_job_lite.png %}

## 搭建运维平台
解压缩elastic-job-lite-console-${version}.tar.gz并执行bin\start.sh。打开浏览器访问http://localhost:8899/即可访问控制台。8899为默认端口号，可通过启动脚本输入-p自定义端口号。

elastic-job-lite-console-${version}.tar.gz可通过mvn install编译获取。
### 登录
提供两种账户，管理员及访客，管理员拥有全部操作权限，访客仅拥有察看权限。默认管理员用户名和密码是root/root，访客用户名和密码是guest/guest，可通过conf\auth.properties修改管理员及访客用户名及密码。
### 功能列表
* 登录安全控制
* 注册中心、事件追踪数据源管理
* 快捷修改作业设置
* 作业和服务器维度状态查看
* 操作作业禁用\启用、停止和删除等生命周期
* 事件追踪查询

### 设计理念
运维平台和elastic-job-lite并无直接关系，是通过读取作业注册中心数据展现作业状态，或更新注册中心数据修改全局配置。

控制台只能控制作业本身是否运行，但不能控制作业进程的启动，因为控制台和作业本身服务器是完全分离的，控制台并不能控制作业服务器。
### 不支持项
加作业 作业在首次运行时将自动添加。Elastic-Job-Lite以jar方式启动，并无作业分发功能。如需完全通过运维平台发布作业，请使用Elastic-Job-Cloud。
注意：依赖注册中心，要先启动zookeeper。
{% asset_img elastic-job-lite-console.png %}

## 快速开始

### 引入Maven依赖
```
<dependency>
	<groupId>com.dangdang</groupId>
	<artifactId>elastic-job-lite-core</artifactId>
	<version>2.1.5</version>
</dependency>

<dependency>
	<groupId>com.dangdang</groupId>
	<artifactId>elastic-job-lite-spring</artifactId>
	<version>2.1.5</version>
</dependency>

<!-- Elastic-Job提供了事件追踪功能，可通过事件订阅的方式处理调度过程的重要事件，用于查询、统计和监控。Elastic-Job目前提供了基于关系型数据库两种事件订阅方式记录事件。添加如下依赖-->
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<version>5.1.38</version>
</dependency>

<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid-spring-boot-starter</artifactId>
	<version>1.1.10</version>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```
### 配置文件application.properties添加如下配置：
#作业注册中心地址
regCenter.serverList=127.0.0.1:2181
regCenter.namespace=elasticjob-demo

simpleJob.cron=0/5 * * * * ?
simpleJob.shardingTotalCount=1
simpleJob.shardingItemParameters=0=Shanghai

#数据源配置
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/elastic_job_log?autoReconnect=true&useSSL=false&allowMultiQueries=true&zeroDateTimeBehavior=convertToNull&failOverReadOnly=false
spring.datasource.username=root
spring.datasource.password=111111

### 配置注册中心
新建RegistryCenterConfig类
```
import com.dangdang.ddframe.job.reg.zookeeper.ZookeeperConfiguration;
import com.dangdang.ddframe.job.reg.zookeeper.ZookeeperRegistryCenter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnExpression;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnExpression("'${regCenter.serverList}'.length() > 0")
public class RegistryCenterConfig {

    @Bean(initMethod = "init")
    public ZookeeperRegistryCenter regCenter(@Value("${regCenter.serverList}") final String serverList, @Value("${regCenter.namespace}") final String namespace) {
        return new ZookeeperRegistryCenter(new ZookeeperConfiguration(serverList, namespace));
    }
}
```

### Spring xml方式配置作业
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:reg="http://www.dangdang.com/schema/ddframe/reg"
    xmlns:job="http://www.dangdang.com/schema/ddframe/job"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://www.dangdang.com/schema/ddframe/reg
                        http://www.dangdang.com/schema/ddframe/reg/reg.xsd
                        http://www.dangdang.com/schema/ddframe/job
                        http://www.dangdang.com/schema/ddframe/job/job.xsd
                        ">
    <!--配置作业注册中心 -->
    <reg:zookeeper id="regCenter" server-lists="yourhost:2181" namespace="dd-job" base-sleep-time-milliseconds="1000" max-sleep-time-milliseconds="3000" max-retries="3" />
    
    <!-- 配置作业-->
    <job:simple id="demoSimpleSpringJob" class="xxx.MyElasticJob" registry-center-ref="regCenter" cron="0/10 * * * * ?" sharding-total-count="3" sharding-item-parameters="0=A,1=B,2=C" />
</beans>
```

将配置Spring命名空间的xml通过Spring启动，作业将自动加载。

### 基于注解的方式配置作业

新建名为ElasticSimpleJob注解类
```
import org.springframework.core.annotation.AliasFor;
import org.springframework.stereotype.Component;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Component
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ElasticSimpleJob {

    /**
     * 作业名称
     */
    String jobName() default "";

    /**
     * cron表达式，用于控制作业触发时间
     */
    @AliasFor("cron")
    String value() default "";

    @AliasFor("value")
    String cron() default "";

    /**
     * 作业分片总数
     */
    int shardingTotalCount() default 1;

    /**
     * 分片序列号和参数用等号分隔，多个键值对用逗号分隔
     * 分片序列号从0开始，不可大于或等于作业分片总数
     * 如：
     * 0=a,1=b,2=c
     */
    String shardingItemParameters() default "";

    /**
     * 作业自定义参数
     * 作业自定义参数，可通过传递该参数为作业调度的业务方法传参，用于实现带参数的作业
     * 例：每次获取的数据量、作业实例从数据库读取的主键等
     */
    String jobParameter() default "";

    /**
     * 是否开启任务执行失效转移，开启表示如果作业在一次任务执行中途宕机，允许将该次未完成的任务在另一作业节点上补偿执行
     */
    boolean failover() default true;

    /**
     * 是否开启错过任务重新执行
     */
    boolean misfire() default true;

    /**
     * 作业描述信息
     */
    String description() default "";

    /**
     * 监控作业运行时状态
     * 每次作业执行时间和间隔时间均非常短的情况，建议不监控作业运行时状态以提升效率。因为是瞬时状态，所以无必要监控。请用户自行增加数据堆积监控。并且不能保证数据重复选取，应在作业中实现幂等性。
     * 每次作业执行时间和间隔时间均较长的情况，建议监控作业运行时状态，可保证数据不会重复选取。
     */
    boolean monitorExecution() default true;

    /**
     * 最大允许的本机与注册中心的时间误差秒数
     * 如果时间误差超过配置秒数则作业启动时将抛异常
     * 配置为-1表示不校验时间误差
     */
    int maxTimeDiffSeconds() default -1;

    /**
     * 作业监控端口
     * 建议配置作业监控端口, 方便开发者dump作业信息。
     * 使用方法: echo “dump” | nc 127.0.0.1 9888
     */
    int monitorPort() default -1;

    /**
     * 作业分片策略实现类全路径
     * 默认使用平均分配策略
     * 如果分片不能整除，则不能整除的多余分片将依次追加到序号小的服务器。如：
     * 如果有3台服务器，分成9片，则每台服务器分到的分片是：1=[0,1,2], 2=[3,4,5], 3=[6,7,8]
     * 如果有3台服务器，分成8片，则每台服务器分到的分片是：1=[0,1,6], 2=[2,3,7], 3=[4,5]
     * 如果有3台服务器，分成10片，则每台服务器分到的分片是：1=[0,1,2,9], 2=[3,4,5], 3=[6,7,8]
     */
    String jobShardingStrategyClass() default "";

    /**
     * 修复作业服务器不一致状态服务调度间隔时间，配置为小于1的任意值表示不执行修复
     * 单位：分钟
     */
    int reconcileIntervalMinutes() default 10;

    /**
     * 作业是否禁止启动
     * 可用于部署作业时，先禁止启动，部署结束后统一启动
     */
    boolean disabled() default false;

    /**
     * 本地配置是否可覆盖注册中心配置
     * 如果可覆盖，每次启动作业都以本地配置为准
     */
    boolean overwrite() default false;

    /**
     * 作业事件追踪的数据源Bean引用
     */
    String dataSource() default "";
}
```
新建JobEventConfig类
```
import com.dangdang.ddframe.job.event.JobEventConfiguration;
import com.dangdang.ddframe.job.event.rdb.JobEventRdbConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.annotation.Resource;
import javax.sql.DataSource;

@Configuration
public class JobEventConfig {

    @Resource
    private DataSource dataSource;

    @Bean
    public JobEventConfiguration jobEventConfiguration() {
        return new JobEventRdbConfiguration(dataSource);
    }
}
```
新建SimpleJobConfig类
```
import com.dangdang.ddframe.job.api.simple.SimpleJob;
import com.dangdang.ddframe.job.config.JobCoreConfiguration;
import com.dangdang.ddframe.job.config.simple.SimpleJobConfiguration;
import com.dangdang.ddframe.job.event.JobEventConfiguration;
import com.dangdang.ddframe.job.event.rdb.JobEventRdbConfiguration;
import com.dangdang.ddframe.job.lite.api.JobScheduler;
import com.dangdang.ddframe.job.lite.config.LiteJobConfiguration;
import com.dangdang.ddframe.job.lite.spring.api.SpringJobScheduler;
import com.dangdang.ddframe.job.reg.zookeeper.ZookeeperRegistryCenter;
import com.example.elasticjobdemo.job.MySimpleJob;
import org.apache.commons.lang3.StringUtils;
import org.springframework.aop.framework.Advised;
import org.springframework.aop.support.AopUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;
import javax.annotation.Resource;
import javax.sql.DataSource;
import java.util.Map;

@Configuration
public class SimpleJobConfig {

    @Resource
    private ZookeeperRegistryCenter regCenter;

    @Resource
    private JobEventConfiguration jobEventConfiguration;

//    @Bean
//    public SimpleJob simpleJob() {
//        return new MySimpleJob();
//    }
//
//    @Bean(initMethod = "init")
//    public JobScheduler simpleJobScheduler(final SimpleJob simpleJob, @Value("${simpleJob.cron}") final String cron, @Value("${simpleJob.shardingTotalCount}") final int shardingTotalCount, @Value("${simpleJob.shardingItemParameters}") final String shardingItemParameters) {
//        return new SpringJobScheduler(simpleJob, regCenter, getLiteJobConfiguration(simpleJob.getClass(), cron, shardingTotalCount, shardingItemParameters), jobEventConfiguration);
//    }
//
//    private LiteJobConfiguration getLiteJobConfiguration(final Class<? extends SimpleJob> jobClass, final String cron, final int shardingTotalCount, final String shardingItemParameters) {
//        return LiteJobConfiguration.newBuilder(new SimpleJobConfiguration(JobCoreConfiguration.newBuilder(jobClass.getName(), cron, shardingTotalCount).shardingItemParameters(shardingItemParameters).build(), jobClass.getCanonicalName())).overwrite(true).disabled(true).build();
//    }

    @Autowired
    private ApplicationContext applicationContext;

    @PostConstruct
    public void initElasticJob() {
        Map<String, SimpleJob> map = applicationContext.getBeansOfType(SimpleJob.class);
        for (Map.Entry<String, SimpleJob> entry : map.entrySet()) {
            SimpleJob simpleJob = entry.getValue();
            if (AopUtils.isAopProxy(simpleJob)) {
                try {
                    simpleJob = (SimpleJob) ((Advised) simpleJob).getTargetSource().getTarget();
                } catch (Exception var1) {
                    throw new RuntimeException(var1);
                }
            }

            ElasticSimpleJob elasticSimpleJobAnnotation = simpleJob.getClass().getAnnotation(ElasticSimpleJob.class);
            if (null != elasticSimpleJobAnnotation) {
                String cron = StringUtils.defaultIfBlank(elasticSimpleJobAnnotation.cron(), elasticSimpleJobAnnotation.value());
                String jobName = StringUtils.isBlank(elasticSimpleJobAnnotation.jobName()) ? simpleJob.getClass().getName() : elasticSimpleJobAnnotation.jobName();
                boolean overwrite = elasticSimpleJobAnnotation.overwrite();
                boolean monitorExecution = elasticSimpleJobAnnotation.monitorExecution();

                //SimpleJob任务配置
                SimpleJobConfiguration simpleJobConfiguration = new SimpleJobConfiguration(
                        JobCoreConfiguration.newBuilder(jobName, cron, elasticSimpleJobAnnotation.shardingTotalCount())
                                .shardingItemParameters(elasticSimpleJobAnnotation.shardingItemParameters())
                                .description(elasticSimpleJobAnnotation.description())
                                .failover(elasticSimpleJobAnnotation.failover())
                                .jobParameter(elasticSimpleJobAnnotation.jobParameter()).build(), simpleJob.getClass().getCanonicalName());
                LiteJobConfiguration liteJobConfiguration = LiteJobConfiguration.newBuilder(simpleJobConfiguration).overwrite(overwrite).monitorExecution(monitorExecution).build();

                JobListener jobListener = new JobListener();

                //配置数据源
                String dataSourceRef = elasticSimpleJobAnnotation.dataSource();
                if (StringUtils.isNotBlank(dataSourceRef)) {

                    if (!applicationContext.containsBean(dataSourceRef)) {
                        throw new RuntimeException("not exist datasource [" + dataSourceRef + "]");
                    }

                    DataSource dataSource = (DataSource) applicationContext.getBean(dataSourceRef);
                    JobEventRdbConfiguration jobEventRdbConfiguration = new JobEventRdbConfiguration(dataSource);
                    SpringJobScheduler jobScheduler = new SpringJobScheduler(simpleJob, regCenter, liteJobConfiguration, jobEventRdbConfiguration, jobListener);
                    jobScheduler.init();
                } else {
                    SpringJobScheduler jobScheduler = new SpringJobScheduler(simpleJob, regCenter, liteJobConfiguration, jobEventConfiguration, jobListener);
                    jobScheduler.init();
                }

                SpringJobScheduler jobScheduler = new SpringJobScheduler(simpleJob, regCenter, liteJobConfiguration, jobListener);
                jobScheduler.init();
            }
        }
    }
}
```
新建作业监听器JobListener类
```
import com.dangdang.ddframe.job.executor.ShardingContexts;
import com.dangdang.ddframe.job.lite.api.listener.ElasticJobListener;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class JobListener implements ElasticJobListener {

    private static final Logger logger = LoggerFactory.getLogger(JobListener.class);

    private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    private long beginTime = 0;

    @Override
    public void beforeJobExecuted(ShardingContexts shardingContexts) {
        beginTime = System.currentTimeMillis();
        logger.info("开始执行 ==> 线程ID：{}，作业任务ID：{}，作业名称：{}，分片总数：{}，作业自定义参数：{}，分片项和参数：{}，作业事件采样统计数：{}，当前作业事件采样统计数：{}，时间：{}",
                Thread.currentThread().getId(),
                shardingContexts.getTaskId(),
                shardingContexts.getJobName(),
                shardingContexts.getShardingTotalCount(),
                shardingContexts.getJobParameter(),
                shardingContexts.getShardingItemParameters(),
                shardingContexts.getJobEventSamplingCount(),
                shardingContexts.getCurrentJobEventSamplingCount(),
                LocalDateTime.now().format(DATE_TIME_FORMATTER));
    }

    @Override
    public void afterJobExecuted(ShardingContexts shardingContexts) {
        long endTime = System.currentTimeMillis();
        logger.info("执行结束 ==> 线程ID：{}，作业任务ID：{}，作业名称：{}，分片总数：{}，作业自定义参数：{}，分片项和参数：{}，作业事件采样统计数：{}，当前作业事件采样统计数：{}，时间：{}，总耗时：{} 毫秒",
                Thread.currentThread().getId(),
                shardingContexts.getTaskId(),
                shardingContexts.getJobName(),
                shardingContexts.getShardingTotalCount(),
                shardingContexts.getJobParameter(),
                shardingContexts.getShardingItemParameters(),
                shardingContexts.getJobEventSamplingCount(),
                shardingContexts.getCurrentJobEventSamplingCount(),
                LocalDateTime.now().format(DATE_TIME_FORMATTER),
                endTime - beginTime);
    }
}
```
新建名为TestSimpleJob任务类
```
import com.dangdang.ddframe.job.api.ShardingContext;
import com.dangdang.ddframe.job.api.simple.SimpleJob;
import com.example.elasticjobdemo.config.ElasticSimpleJob;

@ElasticSimpleJob(jobName = "TestSimpleJob", cron = "0/20 * * * * ?")
public class TestSimpleJob implements SimpleJob {

    @Override
    public void execute(ShardingContext shardingContext) {
        System.out.println("TestSimpleJob 进来了...");
    }
}
```
到此为止，启动程序，在运维控制台就可以看到TestSimpleJob这个任务了，平台上可以对任务进行一些操作。
{% asset_img test-simple-job.png %}

更多关于Elastic-Job的内容，请前往官网：http://elasticjob.io
