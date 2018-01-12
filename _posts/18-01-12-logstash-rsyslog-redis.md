---
title: "Logstash Rsyslog Redis"
layout: post
date: 2018-01-12 19:35
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Logstash
- Rsyslog
- Redis
category: blog
author: JamesiWorks
---

Table of Contents
=================

   * [Table of Contents](#table-of-contents)
      * [Unified Logging](#unified-logging)
      * [Run Logstash Shipper &amp; Broker &amp; Indexer](#run-logstash-shipper--broker--indexer)
      * [Enrich &amp; Transport](#enrich--transport)
      * [Collect](#collect)
      * [Env Test](#env-test)
      * [Install](#install)
      * [Reference](#reference)

## Unified Logging
日志统一输出，Logstash Agent 把 Linux 服务器各节点的 Rsyslog 日志文件收集，然后统一归并集中。

## Run Logstash Shipper & Broker & Indexer
```sh
./logstash agent -f indexer.conf & >/dev/null &
./logstash agent -f shipper.conf & >/dev/null &
```

## Enrich & Transport
```sh
input {
    redis { 
        host      => "120.24.229.84"                 # redis主机地址
        port      => 6379                            # redis端口号
        db        => 10                              # redis数据库编号
        password  => “cTdATrbWXrLpJZCFxrb6X2AV"
        data_type => "channel"                       # 使用发布/订阅模式
        key       => "logstash_list_0"               # 发布通道名称
    }
}

output {
    file { 
        path           => "/home/work/jiangew/logstash/pikachu.log"     # 指定写入文件路径
        message_format => "%{host} %{message}"                          # 指定写入格式
        flush_interval => 0                                             # 指定刷新间隔，0代表实时写入
    }
}
```

## Collect
```sh
input {
    file {
        path => [
            # 监控日志
            "~/services/online-api-http-chihiro-1.0.0_prod/logs/*.stderrout.log"
        ]
    }
}

filter {
    mutate {
        # 替换元数据host的值
        replace => ["host", “ditto1"]
    }
}

output {
    # 输出到控制台
    # stdout { }

    # 输出到redis
    redis {
        host      => "120.24.229.84"               # redis主机地址
        port      => 6379                          # redis端口号
        db        => 10                            # redis数据库编号
        password  => “cTdATrbWXrLpJZCFxrb6X2AV"
        data_type => "channel"                     # 使用发布/订阅模式
        key       => "logstash_list_0"             # 发布通道名称
    }
}
```

## Env Test
```sh
cd /opt/logstash/
bin/logstash -e 'input { stdin { } } output { stdout {} }'
```

## Install
```sh
rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch

cd /etc/yum.repos.d/
vi logstash.repo
[logstash-2.1]
name=Logstash repository for 2.1.x packages
baseurl=http://packages.elastic.co/logstash/2.1/centos
gpgcheck=1
gpgkey=http://packages.elastic.co/GPG-KEY-elasticsearch
enabled=1

yum install logstash
```

## Reference
[Logstash实践 分布式系统的日志监控](http://www.cnblogs.com/yiwenshengmei/p/4956033.html)
