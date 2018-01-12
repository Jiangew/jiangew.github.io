---
title: "Logstash JDBC Elasticsearch"
layout: post
date: 2018-01-12 19:50
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Logstash
- Elasticsearch
category: blog
author: JamesiWorks
---

Table of Contents
=================

  * [Table of Contents](#table-of-contents)
      * [安装 GEM](#安装-gem)
         * [Linux](#linux)
         * [OS X](#os-x)
      * [替换 Ruby 仓库镜像「Amazon to Taobao」](#替换-ruby-仓库镜像amazon-to-taobao)
         * [Output](#output)
         * [请确保只有 ruby.taobao.org](#请确保只有-rubytaobaoorg)
      * [修改 Gemfile 的数据源地址](#修改-gemfile-的数据源地址)
      * [直接替换源，不用修改 Gemfile 地址](#直接替换源不用修改-gemfile-地址)
      * [Install Logstash Plugins from RubyGems](#install-logstash-plugins-from-rubygems)
      * [Install Logstash Plugins Offline](#install-logstash-plugins-offline)
      * [Logstash Plugins List](#logstash-plugins-list)
      * [Logstash-input-jdbc conf](#logstash-input-jdbc-conf)
      * [Logstash input source filter conf](#logstash-input-source-filter-conf)
      * [Logstash-output-elasticsearch conf](#logstash-output-elasticsearch-conf)
      * [Logstash Run](#logstash-run)
      * [Problems Encountered and Solved](#problems-encountered-and-solved)
         * [Elasticsearch 数据重复和增量同步](#elasticsearch-数据重复和增量同步)
         * [数据同步频繁，影响MySQL集群性能](#数据同步频繁影响mysql集群性能)
         * [Elasticsearch 存储容量上升](#elasticsearch-存储容量上升)
         * [增量同步和MySQL范围查询，导致MySQL数据库有修改时无法同步](#增量同步和mysql范围查询导致mysql数据库有修改时无法同步)
         * [大数据量的同步方案](#大数据量的同步方案)

## 安装 GEM
### Linux
```sh
sudo yum install gem
```

### OS X
```sh
sudo brew install gem
```

## 替换 Ruby 仓库镜像「Amazon to Taobao」
```sh
gem sources --add https://ruby.taobao.org/ --remove https://rubygems.org/
gem sources -l
```
### Output
*** CURRENT SOURCES ***
https://ruby.taobao.org

### 请确保只有 ruby.taobao.org
如果还是显示 https://rubygems.org/ 进入 home 的 .gemrc 文件 <br/>
sudo vim ~/.gemrc <br/>
删除 https://rubygems.org/

## 修改 Gemfile 的数据源地址
```sh
whereis logstash

sudo vi Gemfile
修改 source 的值为："https://ruby.taobao.org"

sudo vi Gemfile.jruby-1.9.lock
修改 remote 的值为： https://ruby.taobao.org
```

## 直接替换源，不用修改 Gemfile 地址
```sh
sudo gem install bundler
bundle config mirror.https://rubygems.org https://ruby.taobao.org
```

## Install Logstash Plugins from RubyGems
```sh
bin/logstash-plugin install logstash-output-elasticsearch
bin/logstash-plugin install logstash-input-jdbc
```

## Install Logstash Plugins Offline
Download Gems from RubyGems
```sh
bin/logstash-plugin install /path/to/logstash-output-elasticsearch.gem
bin/logstash-plugin install /path/to/logstash-input-jdbc.gem
```

## Logstash Plugins List
```sh
bin/logstash-plugin list
bin/logstash-plugin list --verbose
bin/logstash-plugin list --group output
```

## Logstash-input-jdbc conf
```sh
input {
    stdin {
    }
    jdbc {
        # mysql jdbc connection string to our backup databse
        jdbc_connection_string => "jdbc:mysql://127.0.0.1:3306/qqreader_comment"
        # the user we wish to excute our statement as
        jdbc_user => "root"
        jdbc_password => "root"
        # the path to downloaded jdbc driver
        jdbc_driver_library => "/Users/Jiangew/logstash/mysql-connector/mysql-connector-java-5.1.45-bin.jar"
        # the name of the driver class for mysql
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        jdbc_paging_enabled => "true"
        jdbc_page_size => "50000"
        # statement_filepath => "/Users/Jiangew/logstash/sql/statement.sql"
        statement => "SELECT * FROM comment where createtime > :sql_last_value"
        # minute hour day month year
        schedule => "*/2 * * * *"
        type => "doc"

        # whether to record_last_run result, save to last_run_metadata_path specified file
        record_last_run => "true"
        # whether to record_last_run result by track column, default track field is timestamp
        use_column_value => "true"
        # customize track column name we need, the value must be incremental, ex: mysql primary key
        tracking_column => "createtime"
        # track column value saved path
        last_run_metadata_path => "/Users/Jiangew/logstash/config/last_id"
        # whether to clear last_run_metadata_path record, if true then each time is equivalent to all database records from scratch
        clear_run => "false"
        # whether to lower the name of the column
        lowercase_column_names => "false"
    }
}
```

## Logstash input source filter conf
```sh
filter {
    # json {
        # source => "message"
        # remove_field => ["message"]
    # }
    # comment table remove fields
    mutate {
        remove_field => ["csid", "title", "content", "supportcount", "weight", "picurls", "videourls", "@version"]
    }
}
```

## Logstash-output-elasticsearch conf
```sh
output {
    elasticsearch {
        hosts => ["localhost:9200"]
        index => "comments"
        document_id => "%{cid}"
    }
    stdout {
        codec => json_lines
    }
    # stdout {
        # codec => rubydebug
    # }
}
```

## Logstash Run
```sh
    bin/logstash -f logstash-jdbc-es.conf
```

## Problems Encountered and Solved

### Elasticsearch 数据重复和增量同步
在默认配置下，tracking_column的值是@timestamp，存在elasticsearch就是_id值，是logstash存入elasticsearch的时间，这个值的主要作用类似mysql的主键，是唯一的；但是我们的时间戳其实是一直在变的，所以我们每次使用select语句查询的数据都会存入elasticsearch中，导致数据重复。<br/>
Fixed：在要查询的表中，找主键或者自增值的字段，将它设置为_id的值，因为_id值是唯一的，所以，当有重复的_id的时候，数据就不会重复写入，只会更新。

### 数据同步频繁，影响MySQL集群性能
由于写入sql语句是固定的，所以每次查询的数据库有很多是已经不需要去查询的，尤其是每次[select * from table;]的时候，对mysql数据库造成了非常大的压力。<br/>
Fixed：根据业务对实时性的要求，可以设定适当的同步时间。ex: 30分钟
```sh
    # crontab [分 时 天 月 年]
    schedule =>  “*/30 * * * *"
```
Fixed：设置MySQL查询范围，防止大量的查询拖死数据库 [xx > :sql_last_value 一定放在WHERE后，然后接AND语句]
```sh
    statement => "SELECT * FROM comment where @timestamp > :sql_last_value"
```

### Elasticsearch 存储容量上升
原因：在[elasticsearch/nodes/0/indices/jdbc/{0,1,2,3,4}/]下有个[translog]，这个是elasticsearch的事务日志，类似mysql的binlog。elasticsearch为了数据安全，接收到数据后，先将数据写入内存和translog，然后再建立索引写入到磁盘，这样即使突然断电，重启后，还可以通过translog恢复，不过这里由于我们每次查询都有很多重复的数据，而这些重复的数据又没有写入到elasticsearch的索引中，所以就囤积了下来。<br/>
解决：elasticsearch会定期refresh，会自动清理掉老的日志，因此可不做处理。

### 增量同步和MySQL范围查询，导致MySQL数据库有修改时无法同步
* 1.设置全量更新索引的频率
* 2.业务侧可以接入消息队列，有数据变更时准实时更新ES索引

### 大数据量的同步方案
问题：logstash使用[enable_paging]时，本质是将原sql语句作为子查询，拼接offset和limit来实现；当查询的结果集较大时存在深度分页瓶颈，数据的抽取效率变低。<br/>
解决：可以使用阿里开源的DataX以及kafka connect来进行数据的全量同步，以及基于时间戳的增量同步。如果对实时性要求较高，最好使用cancal来进行增量订阅和消费。