---
title: "Prometheus in Action"
layout: post
date: 2018-01-28 11:45
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Prometheus
- Metrics
- Alerting
- Monitor
category: blog
author: jiangew
---

<!-- TOC -->

- [Architecture Overview](#architecture-overview)
- [Configuring Prometheus to monitor itself](#configuring-prometheus-to-monitor-itself)
- [Starting Prometheus](#starting-prometheus)
- [Gather metrics](#gather-metrics)
- [Using the expression browser](#using-the-expression-browser)
- [Expression language](#expression-language)
- [Using the graphing interface](#using-the-graphing-interface)
- [Starting up some sample targets](#starting-up-some-sample-targets)
    - [Download the Go client library for Prometheus and run three of these example processes:](#download-the-go-client-library-for-prometheus-and-run-three-of-these-example-processes)
    - [You should now have example targets listening on](#you-should-now-have-example-targets-listening-on)
- [Configuring Prometheus to monitor the sample targets](#configuring-prometheus-to-monitor-the-sample-targets)
    - [Expose such as the rpc_durations_seconds metric](#expose-such-as-the-rpc_durations_seconds-metric)
- [Configure rules for aggregating scraped data into new time series](#configure-rules-for-aggregating-scraped-data-into-new-time-series)
    - [expression](#expression)
    - [prometheus.rules.yml](#prometheusrulesyml)
- [Perfect Prometheus Config](#perfect-prometheus-config)
    - [Expose metric name](#expose-metric-name)
- [Reference](#reference)

<!-- /TOC -->

Power your metrics and alerting with a leading open-source monitoring solution.

## Architecture Overview
![](./assets/images/post/20180128/prometheus.jpg) <br />

## Configuring Prometheus to monitor itself
```sh
prometheus.yml
```

## Starting Prometheus
By default, Prometheus stores its database in ./data (flag --storage.tsdb.path).
```sh
./prometheus --config.file=prometheus.yml
```

## Gather metrics
```sh
localhost:9090/metrics
```

## Using the expression browser
```sh
localhost:9090/graph
```

## Expression language
* prometheus_target_interval_length_seconds
* prometheus_target_interval_length_seconds{quantile="0.99”}
* count(prometheus_target_interval_length_seconds)

## Using the graphing interface
* rate(prometheus_tsdb_head_chunks_created_total[1m])

## Starting up some sample targets

### Download the Go client library for Prometheus and run three of these example processes:
```sh
# Fetch the client library code and compile example.
git clone https://github.com/prometheus/client_golang.git
cd client_golang/examples/random
go get -d
go build

# Start 3 example targets in separate terminals:
./random -listen-address=:8080
./random -listen-address=:8081
./random -listen-address=:8082
```

### You should now have example targets listening on
```sh
http://localhost:8080/metrics
http://localhost:8081/metrics
http://localhost:8082/metrics
```

## Configuring Prometheus to monitor the sample targets

### Expose such as the rpc_durations_seconds metric
```sh
scrape_configs:
  - job_name: 'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

## Configure rules for aggregating scraped data into new time series

### expression
avg(rate(rpc_durations_seconds_count[5m])) by (job, service)

### prometheus.rules.yml
```sh
groups:
- name: example
  rules:
  - record: job_service:rpc_durations_seconds_count:avg_rate5m
    expr: avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

## Perfect Prometheus Config
```sh
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # Evaluate rules every 15 seconds.

  # Attach these extra labels to all timeseries collected by this Prometheus instance.
  external_labels:
    monitor: 'codelab-monitor'

rule_files:
  - 'prometheus.rules.yml'

scrape_configs:
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

### Expose metric name
```sh
job_service:rpc_durations_seconds_count:avg_rate5m
```

## Reference
* [Prometheus Getting Started](https://prometheus.io/docs/prometheus/latest/getting_started/)
* [Prometheus Expression language](https://prometheus.io/docs/prometheus/latest/querying/basics/)
