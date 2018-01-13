---
title: "Elasticsearch 集群初探：部署、管理、监控、代理、压测、调优"
layout: post
date: 2017-12-06 11:45
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Elasticsearch
- Proxy
- TAF 3.0 HTTP
- Vert.x Web
- Gatling
category: blog
author: jiangew
---

Table of Contents
=================

  * [Table of Contents](#table-of-contents)
      * [背景](#背景)
      * [内容](#内容)
      * [1 集群部署](#1-集群部署)
         * [1.1 集群节点配置](#11-集群节点配置)
         * [1.2 集群健康、节点、索引、分片、统计](#12-集群健康节点索引分片统计)
         * [1.3 IK 中文分词](#13-ik-中文分词)
         * [1.4 小结](#14-小结)
      * [2 集群管理与监控](#2-集群管理与监控)
         * [2.1 ElasticHD](#21-elastichd)
         * [2.2 Elasticsearch-head](#22-elasticsearch-head)
         * [2.3 Kibana](#23-kibana)
      * [3 代理层 Server &amp; Client](#3-代理层-server--client)
      * [4 性能压测](#4-性能压测)
         * [4.1 压测基础设施](#41-压测基础设施)
         * [4.2 压测结果汇总](#42-压测结果汇总)
         * [4.3 压测报告分析 单条查询](#43-压测报告分析-单条查询)
         * [4.4 压测报告分析 单条写入](#44-压测报告分析-单条写入)
         * [4.5 压测报告分析 Elasticsearch 集群单节点多场景混合](#45-压测报告分析-elasticsearch-集群单节点多场景混合)
         * [4.6 压测报告分析 读写全索引搜索多场景混合](#46-压测报告分析-读写全索引搜索多场景混合)
         * [4.7 压测问题汇总及解决方案](#47-压测问题汇总及解决方案)
      * [参考资料](#参考资料)

## 背景
目前评论数据存储用户和书籍两个维度的数据，共计8亿条记录左右，还在快速持续增长中；数据链路采用 DCache & Redis & MySQL 存储模式，DCache 存储了一份评论全量数据，Redis 存储评论数据各种维度检索的索引的各种数据结构，MySQL 集群采用分库分表[10*10]策略存储评论原子结构数据。随着评论业务线需求逐渐迭代和复杂化，评论数据的各种维度查询服务逐渐复杂化，Redis 存储的索引数据结构比较隐晦，越来越多，越来越复杂，不易维护；由于评论更新流程是3写，没有数据一致性校验阶段，导致存储侧各种数据不一致。所以，考虑接入Elasticsearch，代替Redis存储侧，简化 MySQL 存储侧各维度评论原子数据为一份；减少依赖链路长度，保证数据一致性。

## 内容
本文是本人 Elasticsearch 集群从部署，到管理与监控，到代理层 Server 和 Client 封装，到测试集群压测，到压测暴露问题的调优，到简单集群运维等，整个链路的初探。记录了整个流程中各阶段的具体操作，遇到问题的解决方案等具体细节。

## 1 集群部署
Elasticsearch 集群的部署，为了保证集群节点配置和插件的统一，可以采用官网的推荐的部署模块 [Elasticsearch Puppet](https://github.com/elastic/puppet-elasticsearch)，也可以采用Docker部署的方式；当然，如果节点较少，也可以采用手动逐个节点部署的方式。

### 1.1 集群节点配置
以下是集群节点的配置，未提及的属性使用默认值，后期如果由于性能需求可以调整；如果个别属性不清楚请问 Google。

#### 集群名称，默认是elasticsearch，不同的集群用名字来区分，es会通过ZenDiscovery服务自动发现在同一网段下的es，配置成相同集群名字的各个节点形成一个集群。如果在同一网段下有多个集群，就可以用这个属性来区分不同的集群。
```sh
    cluster.name: elasticsearch-jew
```

#### 集群节点名称
```sh
    node.name: node-9200
    node.attr.rack: r1
```

#### Data & Log
```sh
    path.data: /path/to/data
    path.logs: /path/to/logs
```

#### Node Host
```sh
    network.host: 192.168.0.1
```

#### HTTP & TCP，设置对外服务的http端口 默认为9200；内部tcp端口 默认9300；不能相同，否则会冲突。
```sh
    http.port: 19200
    http.pipelining.max_events: 30000
    transport.tcp.port: 9302
```

#### cors 跨域支持
```sh
    http.cors.enabled: true
    http.cors.allow-origin: "*"
```

#### 集群节点类型：master | data | client
```sh
    node.master: false
    node.data: true
    node.ingest: false
```

#### 集群节点发现机制
```sh
    discovery.zen.ping.unicast.hosts: ["10.211.0.165:9300", "10.211.0.199:9300"]
```

#### 防止脑裂 [total number of master-eligible nodes / 2 + 1]
```sh
    discovery.zen.minimum_master_nodes: 2
```

#### Elastic X-Pack Basic License Disable Security
```sh
    xpack.security.enabled: false
```

#### Fixed Err: unable to intall syscall filter, CONFIG_SECCOMP not compiled into kernel
```sh
    bootstrap.system_call_filter: false
```

#### Fixed Err: max virtual memory areas vm.max*map*count [65530] is too low
```sh
    sudo sysctl -w vm.max_map_count=262144
```

### 1.2 集群健康、节点、索引、分片、统计
Elasticsearch 提供丰富的 cat APIs，以下会列出一些常用的，如果有不清楚或不能满足需求的，请问 Google。

#### Cluster Health
```sh
curl -XGET -u elastic:elastic 'http://localhost:9200/_cat/health?v&pretty'
curl -XGET -u elastic:elastic 'http://localhost:9200/_cluster/health?pretty'
```

#### 集群索引清单：每个索引细节「状态、分片数、未分配分片数」
```sh
curl -XGET -u elastic:elastic 'http://localhost:9200/_cluster/health?level=indices&pretty'
```

#### 集群索引清单：每个索引分片细节「状态、分片数、未分配分片数」
```sh
curl -XGET -u elastic:elastic 'http://localhost:9200/_cluster/health?level=shards&pretty'
```

#### Cluster Nodes
```sh
curl -XGET -u elastic:elastic 'http://localhost:9200/_cat/nodes?v&pretty'
```

#### 单个节点监控统计：「索引、操作系统、进程、JVM、线程池、文件系统、网络、断路器」
```sh
curl -XGET -u elastic:elastic 'http://localhost:9200/_nodes/stats?pretty'
```

#### 集群统计：「分片数、文档数、存储空间、缓存信息、内存作用率、插件内容、文件系统内容、JVM作用状况、系统CPU和OS信息、段信息」
```sh
curl -XGET -u elastic:elastic 'http://localhost:9200/_cluster/stats?human&pretty'
```

#### 索引列表
```sh
curl -XGET -u elastic:elastic 'http://localhost:9200/_cat/indices?v&pretty'
```

#### 索引统计
```sh
curl -XGET -u elastic:elastic 'http://localhost:9200/_all/_stats?pretty'
curl -XGET -u elastic:elastic 'http://localhost:9200/weibo,elastic/_stats?pretty'
```

#### 每个Index所包含的Type，6.0版本开始，计划每个索引只能有一个Type
```sh
curl -XGET -u elastic:elastic 'http://localhost:9200/_mapping?pretty=true'
```

#### 对索引级别进行完全控制：每个索引默认5个分片和1个副本，更改为1个分片；大拇指法则表明最佳的分片数量取决于节点数量
```sh
curl -XPUT -u elastic:elastic 'http://localhost:9200/twitter?pretty' -H 'Content-Type: application/json' -d'                                     
{
    "index" : {          
        "number_of_shards" : 1,        
        "number_of_replicas" : 1
    }                                              
}'
```

#### 修改副本个数
```sh
curl -XPUT "http://localhost:9200/notes/_settings" -H 'Content-Type: application/json' -d'
{
    "number_of_replicas": 2
}'
```

#### 索引分片状态
```sh
curl -XGET "http://localhost:9200/_cat/shards?v"
```

#### 索引分片的 segment.count 统计
```sh
curl -XGET "http://localhost:9200/_cat/shards/notes?v&h=n,iic,sc"
```

#### 恢复状态：集群出现波动时，查看数据迁移和恢复速度
```sh
curl -XGET "http://localhost:9200/_cat/recovery?v&active_only&h=i,s,shost,thost,fp,bp,tr,trp,trt"
```

#### 查看线程池状态
```sh
curl -XGET "http://localhost:9200/_cat/thread_pool?v"
```

#### Elastic X-Pack Updating Your License
```sh
curl -XPUT -u elastic 'http://localhost:9200/_xpack/license' -H "Content-Type: application/json" -d @license.json
```

#### 内存使用 & GC指标
* Elasticsearch 和 Lucene 以两种方式利用节点上的所有可用RAM：JVM heap 和 文件系统缓存
* Elasticsearch 运行在JVM中，这意味着JVM垃圾回收的持续时间和频率将成为其他重要的监控领域

#### 集群统计 重点监控指标
* nodes.successful
* nodes.failed
* nodes.total
* nodes.mem.used_percent
* nodes.process.cpu.percent
* nodes.jvm.mem.heap_used

### 1.3 IK 中文分词
* 采用特有的”正向迭代最细粒度切分算法”，支持细粒度和最大词长两种切分模式；具有83W字/秒[1600KB/S]的高速处理能力。
* 采用多子处理器分析模式，支持：英文字母、数字、中文词汇等分词处理，兼容韩文、日文字符。
* 优化的词典存储，更小的内存的占用；支持用户词典扩展定义。
* 针对Lucene全文检索优化的查询分析器IKQueryParser；引入简单搜索表达式，采用歧义分析算法优化查询关键字的搜索排列组合，能极大的提高Lucene检索的命中率。

关于IK中文分词具体部署细节，不在这里赘述，感兴趣的话，请问 Google。

### 1.4 小结
关于更多 Elasticsearch 内部实现、部署、运维等细节，请问 Google；包括但不限于如下：
* ElasticSearch 发现机制
* ElasticSearch 存储机制
* ElasticSearch 恢复机制
* ElasticSearch 配置文件
* ElasticSearch 搜索类型

## 2 集群管理与监控
目前 Elasticsearch 集群的监控和管理是通过如下2个开源工具和Kibana来进行的，官方提供的一些插件是收费的，关于更多管理和监控的细节，请问 Google。

* [ElasticHD](https://github.com/farmerx/ElasticHD)
* [elasticsearch-head](https://github.com/mobz/elasticsearch-head)
* [Kibana](https://www.elastic.co/start)

### 2.1 ElasticHD
* start: ./elasticsearch-hd -p 127.0.0.1:9800
* connect: http://elastic:elastic@127.0.0.1:9200
* 域名 *.**.qq.com 映射 10.62.21.163:9800

### 2.2 Elasticsearch-head
* npm install -g grunt grunt-cli
* npm install
* npm run start
* grunt start
* 域名 *.**.qq.com 映射 10.62.21.163:9100

### 2.3 Kibana
Kibana 部署并连接ES集群，最好连接一个非主节点和非数据节点，一个ES路由节点，能够构建搜索负载均衡
* elasticsearch.url: "http://10.211.0.165:19200”
* start: nohup bin/kibana -H 0.0.0.0 > logs/kibana.log 2>&1 &
* 域名 *.**.qq.com 映射 10.211.0.165:5601

## 3 代理层 Server & Client
Elasticsearch集群作为基础服务组件，一定要做好与业务隔离性，做好接口收敛，屏蔽掉业务无关性，封装一些通用的ES接入接口；官方提供了各种语言的Client，可以根据需求自行封装服务，然后与自家的中间件层集成，这样就可以接入自家平台的发布部署、监控、CI相融合。<br/>
我这里封装了 TAF Proxy 接入集团 TAF 服务治理和链路监控，同时封装Client给业务接入方使用，可以做到细粒度到接口层面的ES接入控制，统一版本迭代。<br/>
关于代理层 Server 和 Client 的更多细节就不再这里赘述。

## 4 性能压测
本次压测使用 Gatling 服务端性能压测框架，优秀的 DSL 设计，关于更多细节，请问 Google。<br/>

### 4.1 压测基础设施
压测的吞吐量大小主要依赖于搭建的基础设施，我先罗列下目前压测的基础设施：
* 3台服务器：服务器配置 24 Core、64G RAM、HDD
* 3个ES节点：2个Master 3个Data、5个分片 1个副本；后续压测过程中 扩容到5个ES节点，2个Master 5个Data、5个分片 2个副本
* ES集群目前 75个索引，压测的单个索引 9300W 条记录，磁盘占用 210G，压测的Case都是基于索引全量搜索
* 压测链路：Benchmark Web APIs -> Proxy Server -> Elasticsearch Cluster
* Web APIs 和 Proxy 层都是单点部署，由于测试 nginx 压测TPS到300多就域名解析失败了，所以没有对 Web APIs 和 Proxy 做集群压测

### 4.2 压测结果汇总
目前压测比较稳定的报告汇总(Benchmark Web APIs 和 Proxy 单节点部署):
```sh
   Gatling      R/W    Scene           TPS                 Benchmark          Proxy        Async
10.62.21.158     W       1      5500/120/140/5300      10.188.1.23:10002    10.188.1.23   Chained

10.62.21.158     R       1      12000/120/120/11600    10.188.1.23:10002    10.188.1.23   Chained

10.62.21.158     F       6      350/300/320/2350       10.188.1.23:10002    10.188.1.23   Chained
10.62.21.158     Q       6      350/180/190/2300       10.188.1.23:10002    10.188.1.23   Parallel

10.62.21.158    F/W      8      300/300/315/2450       10.188.1.23:10002    10.188.1.23   Chained
10.62.21.158    Q/W      8      300/180/185/2400       10.188.1.23:10002    10.188.1.23   Parallel

ES Node         F/W      8      450/180/185/3550                                          Chained
ES Node         Q/W      8      450/180/185/3500                                          Parallel
```

针对上述压测报告汇总中的名词解释：
* R/W:      R 单条记录查询；W 单条记录写入；Q Query搜索；F Filter搜索；Q/W Query搜索和写入混合场景；F/W Filter搜索和写入混合场景；
* Scene:    压测场景个数
* TPS:      每秒注入的虚拟用户数 / 注入虚拟用户数的时间段 / 压测场景运行时长 / TPS
* Async:    Gatling 提供2种异步方式，Chained 链式异步，Parallel 并行异步

### 4.3 压测报告分析 单条查询

#### 4.3.1 压测结果
```sh
   Gatling      R/W    Scene           TPS                 Benchmark          Proxy        Async
10.62.21.158     R       1      12000/120/120/11600    10.188.1.23:10002    10.188.1.23   Chained
```

#### 4.3.2 压测报告

##### 压测报告信息汇总
![](/assets/images/post/20171206/get-01.jpg) <br />
##### 虚拟用户注入时间分布
![](/assets/images/post/20171206/get-02.jpg) <br />
##### 请求响应时间分布
![](/assets/images/post/20171206/get-03.jpg) <br />
##### 请求和响应吞吐量
![](/assets/images/post/20171206/get-04.jpg) <br />

### 4.4 压测报告分析 单条写入

#### 4.4.1 压测结果
```sh
   Gatling      R/W    Scene           TPS                 Benchmark          Proxy        Async
10.62.21.158     W       1      5500/120/140/5300      10.188.1.23:10002    10.188.1.23   Chained
```

#### 4.4.2 压测报告

##### 压测报告信息汇总
![](/assets/images/post/20171206/put-01.jpg) <br />
##### 虚拟用户注入时间分布
![](/assets/images/post/20171206/put-02.jpg) <br />
##### 请求响应时间分布
![](/assets/images/post/20171206/put-03.jpg) <br />
##### 请求和响应吞吐量
![](/assets/images/post/20171206/put-04.jpg) <br />

### 4.5 压测报告分析 Elasticsearch 集群单节点多场景混合

#### 4.5.1 压测结果
```sh
   Gatling      R/W    Scene           TPS                 Benchmark          Proxy        Async
ES Node         F/W      8      450/180/185/3550                                          Chained
ES Node         Q/W      8      450/180/185/3500                                          Parallel
```

#### 4.5.2 压测报告

##### 压测报告信息汇总
![](/assets/images/post/20171206/node-01.jpg) <br />
##### 虚拟用户注入时间分布
![](/assets/images/post/20171206/node-02.jpg) <br />
##### 请求响应时间分布
![](/assets/images/post/20171206/node-03.jpg) <br />
##### 请求和响应吞吐量
![](/assets/images/post/20171206/node-04.jpg) <br />

### 4.6 压测报告分析 读写全索引搜索多场景混合

#### 4.6.1 压测结果
```sh
   Gatling      R/W    Scene           TPS                 Benchmark          Proxy        Async
10.62.21.158    F/W      8      300/300/315/2450       10.188.1.23:10002    10.188.1.23   Chained
10.62.21.158    Q/W      8      300/180/185/2400       10.188.1.23:10002    10.188.1.23   Parallel
```

#### 4.6.2 压测报告

##### 压测报告信息汇总
![](/assets/images/post/20171206/mix-01.jpg) <br />
##### 虚拟用户注入时间分布
![](/assets/images/post/20171206/mix-02.jpg) <br />
##### 请求响应时间分布
![](/assets/images/post/20171206/mix-03.jpg) <br />
##### 请求和响应吞吐量
![](/assets/images/post/20171206/mix-04.jpg) <br />

### 4.7 压测问题汇总及解决方案

#### 01.Elastic X-Pack License Expired，集群读写正常，健康检查、索引管理、集群监控和统计 功能无法使用 ？
fixed：重新申请 Free Basic License

#### 02.Basic License Disable Security ？
fixed：配置 xpack.security.enabled: false

#### 03.部署测试ES集群 5个节点 集群间无法通信 ？
调整配置文件；跟运维@张京 沟通服务器外网和端口开放问题

#### 04.Fixed err: unable to intall syscall filter, CONFIG_SECCOMP not compiled into kernel
bootstrap.system_call_filter: false

#### 05.Fixed err: max virtual memory areas vm.max*map*count [65530] is too low
```sh
sudo sysctl -w vm.max_map_count=262144
```

#### 06.多播模式：配置不配置 network.host & transport.host 节点之间不能互相发现

#### 07.单播模式：必须配置 network.host，hosts 列表必须配置端口或端口范围

#### 08.更新 License，从trial到basic，Elastic X-Pack Updating Your License
```sh
curl -XPUT -u elastic 'http://localhost:9200/_xpack/license’ -H "Content-Type: application/json" -d @license.json
```

#### 09.Elastic X-Pack Basic License Disable Security
xpack.security.enabled: false

#### 10.集群控制台ElasticHD配置域名[*.**.qq.com]连接后，集群可以连接，Rest API报错，502: Bad Gateway ?
fixed: nginx层反向代理的问题，域名连接加80端口[*.**.qq.com:80]

#### 11.并发量上来 restClient close 问题
fixed bug: httpClient async request timeout「设置0」

#### 12.并发量上来大量「502 504」，分析发现是域名解析超时，临时干掉nginx域名解析反向代理层

#### 13.fixed bug: Cannot assign requested address「无可用端口，短链接关闭后，链接处于 TIME_WAIT 状态，本地端口仍被占用中」
```sh
    # 端口复用内核参数设置
    net.ipv4.tcp_tw_reuse=1
    net.ipv4.tcp_tw_recycle=1
    net.ipv4.tcp_timestamps=1
    net.ipv4.tcp_max_tw_buckets=50000
    net.ipv4.ip_local_port_range=1024 65000
    net.ipv4.tcp_keepalive_time=6000
```
#### 14.开启端口复用后，请求一直连接超时；并且一直触发GC「Minor GC，不是 Full GC」，新生代中没有合适区域存放分配的数据结构
fixed: JVM调大堆内存，调高新生代和老生代比例

#### 15.调整 ArslanElasticBenchmark & ArslanElasticServer 线程数、队列长度、超时时间、Gatling配置 多轮压测

#### 16.master被压挂了，索引分片丢失，30mins恢复索引分片同步

#### 17.ES单节点堆内存不超过32G，为了充分利用服务器[64G内存]资源，节点由3个扩容到5个

#### 18.ES集群节点从3个扩容到5个，集群分片动态迁移达到均衡状态，每个节点2个分片

#### 19.副本从1调整到2，提升搜索性能；压测脚本重构，从链式异步改成并行异步；压测脚本区分场景

#### 20.Vert.x web 重构 TAF 3.0 HTTP 压测基准 Web APIs ArslanElasticBenchmarkV2

#### 21.跟@帆总一起排查 "TAF主控连接关闭问题”
bug: 同时创建多个communicator时，并发高时会出现窜包问题，即连接和回包的对应关系在特定场景下出现匹配错乱；
     深层次原因，初步认为与连接异步创建和回包超时淘汰有关，等后续 base-taf 版本迭代fix

## 参考资料
* [Elastic Start](https://www.elastic.co/start)
* [Elastic Stack](https://www.elastic.co/guide/index.html)
* [ES 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)
* [ES 英文指南](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
* [Rest Client & High Level Rest Client](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/index.html)
* [ES 学习笔记](https://coyotey.gitbooks.io/elasticsearch/content)
* [ES 中文手册](https://doc.yonyoucloud.com/doc/mastering-elasticsearch/index.html)
* [ES 集群监控](http://www.54tianzhisheng.cn/2017/10/15/ElasticSearch-cluster-health-metrics)
* [ES 中分分词](https://juejin.im/post/59b2b2ea6fb9a00a3b3bd011)
* [ES 搜索入门](http://mp.weixin.qq.com/s/npXpXgiLZxTV93YgykInwg)
