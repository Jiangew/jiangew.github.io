---
title: "Hawkular Metrics in Action"
layout: post
date: 2018-01-28 11:22
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Hawkular Metrics
- Cassandra
- CCM
- Grafana
- Monitor
category: blog
author: jiangew
---

Table of Contents
=================

  * [Table of Contents](#table-of-contents)
      * [Cassandra](#cassandra)
         * [Reference](#reference)
      * [CCM「Cassandra Cluster Manager」](#ccmcassandra-cluster-manager)
         * [Reference](#reference-1)
         * [Create Cassandra Cluster &amp; Start &amp; Stop](#create-cassandra-cluster--start--stop)
      * [Hawkular Metrcis](#hawkular-metrcis)
         * [Reference](#reference-2)
         * [Run Hawkular Services](#run-hawkular-services)
         * [Dashboard](#dashboard)
         * [Metric Types](#metric-types)
         * [Creating Tenants](#creating-tenants)
         * [Creating Metrics](#creating-metrics)
         * [Query Metrics](#query-metrics)
      * [Grafana](#grafana)
         * [Reference](#reference-3)
         * [Install &amp; Start &amp; Stop](#install--start--stop)
         * [Foreground Run](#foreground-run)
         * [Hawkular Datasource Plugin](#hawkular-datasource-plugin)
         * [Dashboard](#dashboard-1)

## Cassandra
Apache Cassandra is a highly-scalable partitioned row store. Rows are organized into tables with a required primary key.

### Reference
* [Cassandra Github](https://github.com/apache/cassandra)

## CCM「Cassandra Cluster Manager」
A script/library to create, launch and remove an Apache Cassandra cluster on localhost.

### Reference
* [CCM Github](https://github.com/pcmanus/ccm)

### Create Cassandra Cluster & Start & Stop
```sh
ccm create -v 3.0.12 -n 1 -s hawkular
ccm updateconf "start_rpc: true"
ccm start
ccm stop
ccm remove

ccm node1 ring
ccm node1 show
ccm node1 cqlsh

desc cluster
desc keyspaces
desc keyspace hawkular_metrics
desc tables
desc table xxx
```

## Hawkular Metrcis
Hawkular Metrics, a storage engine for metric data.

### Reference
* [Hawkular Metrics Quickstart Guide](http://www.hawkular.org/hawkular-services/docs/quickstart-guide)
* [Hawkular Metrics Installation Guide](http://www.hawkular.org/hawkular-services/docs/installation-guide)
* [Hawkular Metrics User Guide](http://www.hawkular.org/hawkular-metrics/docs/user-guide/#_introduction)
* [Hawkular Metrics Github](https://github.com/hawkular/hawkular-metrics)

### Run Hawkular Services
```sh
cd hawkular
sh bin/add-user.sh -a -u jiangew -p 123456 -g read-write,read-only
sh bin/standalone.sh -Dhawkular.rest.user=jiangew -Dhawkular.rest.password=123456 -Dhawkular.agent.enabled=true
```

### Dashboard
```sh
localhost:8080
```

### Metric Types
```sh
Availability
Gauge
Counter
String
```

### Creating Tenants
```sh
curl -X POST http://localhost:8080/hawkular/metrics/gauges/raw -d @request.json \
-H "Content-Type: application/json” \
-H "Hawkular-Tenant: jiangew.me"
```
```sh
curl -X POST http://localhost:8080/hawkular/metrics/tenants -d '{"id": “jiangew.me”}' \
-H "Content-Type: application/json"
```

### Creating Metrics
```sh
curl -u jiangew:123456 \
-X POST http://localhost:8080/hawkular/metrics/gauges/temperature/raw \
-d @metrics_day_1.json \
-H "Content-Type: application/json” \
-H "Hawkular-Tenant: jiangew.me"
```
```sh
curl -u jiangew:123456 \
-X POST http://localhost:8080/hawkular/alters/import/all \
-d @trigger_definition.json \
-H "Content-Type: application/json” \
-H "Hawkular-Tenant: jiangew.me"
```

### Query Metrics
```sh
 curl -u jiangew:123456 \
-X GET "http://localhost:8080/hawkular/metrics/gauges/temperature/raw?start=1468578600000&end=1468594800001&order=ASC” \
-H "Content-Type: application/json” \
-H "Hawkular-Tenant: jiangew.me" 
```
```sh
curl -u jiangew:123456 \
-X GET "http://localhost:8080/hawkular/metrics/gauges/temperature/stats?bucketDuration=2h&start=1468533600000&end=1468618200001” \
-H "Content-Type: application/json” \
-H "Hawkular-Tenant: jiangew.me"
```
```sh
curl -u jiangew:123456 \
-X GET http://0.0.0.0:8080/hawkular/metrics/gauges/temperature/raw \
-H "Content-Type: application/json” \
-H "Hawkular-Tenant: hawkular"
```

## Grafana
Grafana is an open source, feature rich metrics dashboard and graph editor for Graphite, Elasticsearch, OpenTSDB, Prometheus and InfluxDB.

### Reference
* [Hawkular Client Grafana Quickstart Guide](http://www.hawkular.org/hawkular-clients/grafana/docs/quickstart-guide)
* [Hawkular Datasource For Grafana](https://grafana.com/plugins/hawkular-datasource)
* [Grafana Installation Config](http://docs.grafana.org/installation/configuration)
* [Grafana Github](https://github.com/grafana/grafana)

### Install & Start & Stop
```sh
brew update
brew install grafana
brew services start grafana
```

### Foreground Run
```sh
grafana-server --config=/usr/local/etc/grafana/grafana.ini \
--homepath /usr/local/share/grafana \
cfg:default.paths.logs=/usr/local/var/log/grafana \
cfg:default.paths.data=/usr/local/var/lib/grafana \
cfg:default.paths.plugins=/usr/local/var/lib/grafana/plugins
```

### Hawkular Datasource Plugin
```sh
grafana-cli plugins install hawkular-datasource
```

### Dashboard
```sh
localhost:3000
```
