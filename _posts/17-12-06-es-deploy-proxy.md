---
title: "Elasticsearch 集群初探：部署、管理、监控、代理层、压测、调优"
layout: post
date: 2017-12-06 11:45
image: /assets/images/markdown.jpg
headerImage: false
tag:
- Elasticsearch
- Proxy
- TAF 3.0 HTTP
- Vert.x Web
- Gatling
category: blog
author: JamesiWorks
---

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
目前 Elasticsearch 集群的监控和管理是通过如下2个开源工具和Kibana来进行的，官方的是收费的，关于更多细节，请问 Google。

* [ElasticHD](https://github.com/farmerx/ElasticHD)
* [elasticsearch-head](https://github.com/mobz/elasticsearch-head)
* [Kibana](https://www.elastic.co/start)

### 1.1 ElasticHD
* start: ./elasticsearch-hd -p 127.0.0.1:9800
* connect: http://elastic:elastic@127.0.0.1:9200
* 域名 arslan-cluster.bookcs.3g.qq.com 映射 10.62.21.163:9800

### 1.2 Elasticsearch-head
* npm install -g grunt grunt-cli
* npm install
* npm run start
* grunt start
* 域名 arslan-head.bookcs.3g.qq.com 映射 10.62.21.163:9100

### 1.3 Kibana「部署并连接ES集群，最好连接一个非主节点和非数据节点，一个ES路由节点，能够构建搜索负载均衡」
* elasticsearch.url: "http://10.211.0.165:19200”
* start: nohup bin/kibana -H 0.0.0.0 > logs/kibana.log 2>&1 & 「1:屏幕输出；2:错误输出；合并」
* 域名 arslan-kibana.bookcs.3g.qq.com 映射 10.211.0.165:5601

## 3 代理层 Server & Client

## 4 性能压测

## 参考资料
* [Elastic Start](https://www.elastic.co/start)
* [Elastic Stack](https://www.elastic.co/guide/index.html)
* [ES 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)
* [ES 英文指南](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
* [Rest Client & High Level Rest Client](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/index.html)
* [ES 学习笔记](https://coyotey.gitbooks.io/elasticsearch/content)
* [ES 中文手册](https://doc.yonyoucloud.com/doc/mastering-elasticsearch/index.html)
* [ES 集群监控](http://www.54tianzhisheng.cn/2017/10/15/ElasticSearch-cluster-health-metrics)
* [ES IK分词器](https://juejin.im/post/59b2b2ea6fb9a00a3b3bd011)
* [ES 搜索入门](http://mp.weixin.qq.com/s/npXpXgiLZxTV93YgykInwg)
