---
title: "Filebeat Logstash Elasticsearch"
layout: post
date: 2018-01-28 13:10
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Filebeat
- Logstash
- Elasticsearch
category: blog
author: jiangew
---

Table of Contents
=================

   * [Filebeat](#filebeat)
      * [Beats combo Logstash Highly Available](#beats-combo-logstash-highly-available)
      * [Install Filebeat](#install-filebeat)
      * [Config Filebeat](#config-filebeat)
         * [Filebeat Prospectors](#filebeat-prospectors)
         * [Filebeat modules](#filebeat-modules)
         * [Elasticsearch template setting](#elasticsearch-template-setting)
         * [Dashboards](#dashboards)
         * [Set up the Kibana dashboards by Commond](#set-up-the-kibana-dashboards-by-commond)
         * [Kibana](#kibana)
         * [Elasticsearch output](#elasticsearch-output)
         * [Logstash output](#logstash-output)
         * [Logstash conf](#logstash-conf)
         * [Elasticsearch input plugins](#elasticsearch-input-plugins)
         * [Start Filebeat](#start-filebeat)
      * [Reference](#reference)

# Filebeat
* Lightweight Shipper for Logs.
* Ship to Elasticsearch or Logstash.
* Visualize in Kibana.

## Beats combo Logstash Highly Available
![](/assets/images/post/20180128/beats-logstash-ha-01.png) <br />
![](/assets/images/post/20180128/beats-logstash-ha-02.png) <br />

## Install Filebeat
```sh
    curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-6.1.1-darwin-x86_64.tar.gz
    tar xzvf filebeat-6.1.1-darwin-x86_64.tar.gz
```
```sh
    docker pull docker.elastic.co/beats/filebeat:6.1.1
```

## Config Filebeat

### Filebeat Prospectors
```sh
filebeat.prospectors:
# Each - is a prospector. Most options can be set at the prospector level, so
# you can use different prospectors for various configurations.
# Below are the prospector specific configurations.
- type: log

  # Change to true to enable this prospector configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
  - /Users/Jiangew/elastic/elasticsearch/node0/logs/*.log
  - /Users/Jiangew/elastic/elasticsearch/node1/logs/*.log
  - /Users/Jiangew/elastic/elasticsearch/node2/logs/*.log
```

### Filebeat modules
```sh
filebeat.config.modules:
  # Glob pattern for configuration loading
  path: ${path.config}/modules.d/*.yml

  # Set to true to enable config reloading
  reload.enabled: false

  # Period on which files under path should be checked for changes
  reload.period: 10s
```

### Elasticsearch template setting
```sh
setup.template.settings:
  index.number_of_shards: 1
  #index.codec: best_compression
  #_source.enabled: false

# Configure template loading
setup.template.name: "filebeat"
setup.template.pattern: "filebeat-*"
#setup.template.fields: "path/to/fields.yml"
#setup.template.overwrite: true
#setup.template.enabled: false
```

### Dashboards
```sh
# These settings control loading the sample dashboards to the Kibana index. Loading
# the dashboards is disabled by default and can be enabled either by setting the
# options here, or by using the `-setup` CLI flag or the `setup` command.
setup.dashboards.enabled: true
setup.dashboards.index: "filebeat-*"
```

### Set up the Kibana dashboards by Commond
```sh
    ./filebeat setup --dashboards
```
```sh
    docker run docker.elastic.co/beats/filebeat:6.1.1 setup --dashboards
```

### Kibana
```sh
setup.kibana:
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  host: "localhost:5601"

  # Basic auth credentials
  #username: "elastic"
  #password: "elastic"
```

### Elasticsearch output
```sh
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200", "localhost:9201", "localhost:9202"]
  index: "filebeat-%{[beat.version]}-%{+yyyy.MM.dd}"
  # indices:
  # - index: "filebeat-failed-%{[beat.version]}-%{+yyyy.MM.dd}"
      # when.contains:
      # message: "failed"

  # Optional protocol and basic auth credentials.
  #protocol: "https"
  #username: "elastic"
  #password: "elastic"
```

### Logstash output
```sh
output.logstash:
  # The Logstash hosts
  hosts: ["localhost:5044"]
```

### Logstash conf
```sh
input {
  beats {
    port => 5044
  }
}

# The filter part of this file is commented out to indicate that it is optional.
# filter {
#
# }

output {
  elasticsearch {
    hosts => "localhost:9200"
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    document_type => "%{[@metadata][type]}"
  }
}
```

### Elasticsearch input plugins
```sh
    bin/logstash-plugin install logstash-input-beats
```

### Start Filebeat
```sh
    sudo chown -R Jiangew:admin filebeat/
    sudo chown root filebeat.yml
    sudo ./filebeat -e -c filebeat.yml -d "publish"
```
```sh 
    docker run docker.elastic.co/beats/filebeat:6.1.1
```

## Reference
* [Filebeat Getting Started](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-getting-started.html)
