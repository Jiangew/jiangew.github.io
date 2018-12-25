---
title: "分布式 Schedule Job 业务接入开发指南"
layout: post
date: 2018-06-27 18:45
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Zookeeper
- Job
category: blog
author: jiangew
---

<!-- TOC -->

- [0 环境要求](#0-环境要求)
- [1 作业开发](#1-作业开发)
    - [1.1 Simple 类型作业](#11-simple-类型作业)
    - [1.2 Dataflow 类型作业](#12-dataflow-类型作业)
    - [1.3 流式处理](#13-流式处理)
- [2 作业配置](#2-作业配置)
- [3 基于 SpringBoot 业务接入开发指南](#3-基于-springboot-业务接入开发指南)
    - [3.1 项目pom文件中新增maven依赖](#31-项目pom文件中新增maven依赖)
    - [3.2 项目配置文件中新增 RegistryCenter & Job Config](#32-项目配置文件中新增-registrycenter--job-config)
    - [3.3 注册中心 Configuration Bean 注入](#33-注册中心-configuration-bean-注入)
    - [3.4 事件追踪数据源 Configuration Bean 注入](#34-事件追踪数据源-configuration-bean-注入)
    - [3.5 新增 Job Service](#35-新增-job-service)
    - [3.6 新增 Job Configuration](#36-新增-job-configuration)
    - [3.7 启动 SpringBoot](#37-启动-springboot)
    - [3.8 登录 Schedule Job 控制台，分别配置 注册中心和事件追踪数据源](#38-登录-schedule-job-控制台分别配置-注册中心和事件追踪数据源)
    - [3.9 新增了注册中心配置后，可以看到作业操作，包含Job维度和Server维度](#39-新增了注册中心配置后可以看到作业操作包含job维度和server维度)
    - [4.0 新增了数据追踪数据源后，可以查看作业历史，包括历史轨迹和历史状态。](#40-新增了数据追踪数据源后可以查看作业历史包括历史轨迹和历史状态)

<!-- /TOC -->

## 0 环境要求
采用当当开源的 Elastic-Job 框架，要求JDK 1.7及以上版本，Zookeeper 3.4.6及以上版本，Maven 3.0.4及以上版本。

## 1 作业开发
Elastic-Job-Lite 提供统一作业接口，开发者仅需对业务作业进行一次开发，之后可根据不同的配置以及部署至不同的Lite环境。

Elastic-Job提供Simple、Dataflow和Script 3种作业类型。 方法参数shardingContext包含作业配置、片和运行时信息。可通过 getShardingTotalCount()，getShardingItem() 等方法分别获取分片总数，运行在本作业服务器的分片序列号等。

### 1.1 Simple 类型作业
意为简单实现，未经任何封装的类型。需实现SimpleJob接口。该接口仅提供单一方法用于覆盖，此方法将定时执行。与Quartz原生接口相似，但提供了弹性扩缩容和分片等功能。
```java
public class MyElasticJob implements SimpleJob {
    @Override
    public void execute(ShardingContext context) {
        switch (context.getShardingItem()) {
            case 0:
                // do something by sharding item 0
                break;
            case 1:
                // do something by sharding item 1
                break;
            case 2:
                // do something by sharding item 2
                break;
            // case n: ...
        }
    }
}
```

### 1.2 Dataflow 类型作业
Dataflow类型用于处理数据流，需实现DataflowJob接口。该接口提供2个方法可供覆盖，分别用于抓取(fetchData)和处理(processData)数据。
```java
public class MyElasticJob implements DataflowJob<Foo> {
    @Override
    public List<Foo> fetchData(ShardingContext context) {
        switch (context.getShardingItem()) {
            case 0:
                List<Foo> data = // get data from database by sharding item 0
                return data;
            case 1:
                List<Foo> data = // get data from database by sharding item 1
                return data;
            case 2:
                List<Foo> data = // get data from database by sharding item 2
                return data;
            // case n: ...
        }
    }

    @Override
    public void processData(ShardingContext shardingContext, List<Foo> data) {
        // process data
        // ...
    }
}
```

### 1.3 流式处理
可通过DataflowJobConfiguration配置是否流式处理。

流式处理数据只有fetchData方法的返回值为null或集合长度为空时，作业才停止抓取，否则作业将一直运行下去； 非流式处理数据则只会在每次作业执行过程中执行一次fetchData方法和processData方法，随即完成本次作业。

如果采用流式作业处理方式，建议processData处理数据后更新其状态，避免fetchData再次抓取到，从而使得作业永不停止。 流式数据处理参照TbSchedule设计，适用于不间歇的数据处理。

## 2 作业配置
Elastic-Job配置分为3个层级，分别是Core, Type和Root。每个层级使用相似于装饰者模式的方式装配。

Core对应JobCoreConfiguration，用于提供作业核心配置信息，如：作业名称、分片总数、CRON表达式等。

Type对应JobTypeConfiguration，有3个子类分别对应SIMPLE, DATAFLOW和SCRIPT类型作业，提供3种作业需要的不同配置，如：DATAFLOW类型是否流式处理或SCRIPT类型的命令行等。

Root对应JobRootConfiguration，有2个子类分别对应Lite和Cloud部署类型，提供不同部署类型所需的配置，如：Lite类型的是否需要覆盖本地配置或Cloud占用CPU或Memory数量等。

## 3 基于 SpringBoot 业务接入开发指南

### 3.1 项目pom文件中新增maven依赖
```java
<dependency>
    <artifactId>elastic-job-common-core</artifactId>
    <groupId>com.dangdang</groupId>
    <version>2.1.5</version>
</dependency>
<dependency>
    <artifactId>elastic-job-lite-core</artifactId>
    <groupId>com.dangdang</groupId>
    <version>2.1.5</version>
</dependency>
<dependency>
    <artifactId>elastic-job-lite-spring</artifactId>
    <groupId>com.dangdang</groupId>
    <version>2.1.5</version>
</dependency>
```

### 3.2 项目配置文件中新增 RegistryCenter & Job Config
```sh
# elastic job
elasticjob:
    registryCenter:
      serverList: 172.25.205.40:2181
      namespace: marketing-elastic-job
    exchangeMatchResultJob:
      enable: true
      cron: 0 */1 * * * ?
      shardingTotalCount: 1
      shardingItemParameters: 0=A,1=B
```

### 3.3 注册中心 Configuration Bean 注入
```java
@Configuration
@ConditionalOnExpression("'${elasticjob.registryCenter.serverList}'.length() > 0")
public class RegistryCenterConfig {

    @Bean(initMethod = "init")
    public ZookeeperRegistryCenter registryCenter(
            @Value("${elasticjob.registryCenter.serverList}") final String serverList,
            @Value("${elasticjob.registryCenter.namespace}") final String
            namespace) {
        return new ZookeeperRegistryCenter(new ZookeeperConfiguration(serverList, namespace));
    }
}
```

### 3.4 事件追踪数据源 Configuration Bean 注入
```java
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

### 3.5 新增 Job Service
```java
@Slf4j
public class ExchangeMatchResultJob implements SimpleJob {

    @Override
    public void execute(ShardingContext shardingContext) {
       switch (shardingContext.getShardingItem()) {
    		case 0:
        	// do something by sharding item 0
        		break;
    		case 1:
        	// do something by sharding item 1
        		break;
    		// case n: ...
		}
    }
}
```

### 3.6 新增 Job Configuration
```java
@Configuration
public class ExchangeMatchResultJobConfig {

    @Resource
    private ZookeeperRegistryCenter registryCenter;

    @Resource
    private JobEventConfiguration jobEventConfiguration;

    @Bean
    public SimpleJob exchangeMatchResultJob() {
        return new ExchangeMatchResultJob();
    }

    @Bean(initMethod = "init")
    public JobScheduler exchangeMatchResultJobScheduler(
            final SimpleJob exchangeMatchResultJob,
            @Value("${elasticjob.exchangeMatchResultJob.cron}") final String cron,
            @Value("${elasticjob.exchangeMatchResultJob.shardingTotalCount}") final int shardingTotalCount,
            @Value("${elasticjob.exchangeMatchResultJob.shardingItemParameters}") final String shardingItemParameters) {
        return new SpringJobScheduler(
                exchangeMatchResultJob,
                registryCenter,
                getLiteJobConfiguration(exchangeMatchResultJob.getClass(), cron, shardingTotalCount, shardingItemParameters),
                jobEventConfiguration
        );
    }

	private LiteJobConfiguration getLiteJobConfiguration(
        final Class<? extends SimpleJob> jobClass, final String cron, final int shardingTotalCount, final String shardingItemParameters) {
    return LiteJobConfiguration.newBuilder(new SimpleJobConfiguration(JobCoreConfiguration.newBuilder(jobClass.getName(),
            cron,
            shardingTotalCount
    ).shardingItemParameters(shardingItemParameters).build(), jobClass.getCanonicalName())).overwrite(true).build();
}

}
```

### 3.7 启动 SpringBoot

### 3.8 登录 Schedule Job 控制台，分别配置 注册中心和事件追踪数据源

![](/assets/images/post/20181027/elasticjob-01.png)<br>
![](/assets/images/post/20181027/elasticjob-02.png)<br>

### 3.9 新增了注册中心配置后，可以看到作业操作，包含Job维度和Server维度

Job维度，可以进行Job配置的修改，触发，失效，终止操作。<br>
![](/assets/images/post/20181027/elasticjob-03.png)<br>

Server维度，可以对服务实例进行失效，终止操作。<br>
![](/assets/images/post/20181027/elasticjob-04.png)<br>

### 4.0 新增了数据追踪数据源后，可以查看作业历史，包括历史轨迹和历史状态。

![](/assets/images/post/20181027/elasticjob-05.png)<br>
![](/assets/images/post/20181027/elasticjob-06.png)<br>

~ 今天就到这里吧 ！！！~
