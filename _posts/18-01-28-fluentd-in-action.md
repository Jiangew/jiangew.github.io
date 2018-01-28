---
title: "Fluentd in Action"
layout: post
date: 2018-01-28 13:25
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Fluentd
- Log
category: blog
author: jiangew
---

Table of Contents
=================

   * [Fluentd](#fluentd)
      * [Fluentd Unified Logging Layer](#fluentd-unified-logging-layer)
      * [Fluentd Highly Available](#fluentd-highly-available)
      * [Install Fluentd from Source](#install-fluentd-from-source)
         * [Fetch Source Code](#fetch-source-code)
         * [Build and Install](#build-and-install)
         * [Run](#run)
         * [Stop](#stop)
      * [Install fluentd-ui](#install-fluentd-ui)
         * [Install fluentd-ui via gem command](#install-fluentd-ui-via-gem-command)
         * [Change password](#change-password)
      * [Reference](#reference)

# Fluentd
Fluentd is an open source data collector for unified logging layer.

## Fluentd Unified Logging Layer
![](/assets/images/post/20180128/fluentd.jpg) <br />

## Fluentd Highly Available
![](/assets/images/post/20180128/fluentd-ha.png) <br />

## Install Fluentd from Source

### Fetch Source Code
```sh
$ git clone https://github.com/fluent/fluentd.git
$ cd fluentd
```

### Build and Install
Build the package with rake and install it with gem.
```sh
$ bundle install
Fetching gem metadata from https://rubygems.org/.........
...
Your bundle is complete!
Use `bundle show [gemname]` to see where a bundled gem is installed.
$ bundle exec rake build
fluentd xxx built to pkg/fluentd-xxx.gem.
$ gem install pkg/fluentd-xxx.gem
```

### Run
```sh
$ fluentd --setup ./fluent
$ fluentd -c ./fluent/fluent.conf -vv &
$ echo '{"json":"message"}' | fluent-cat debug.test
```

### Stop
```sh
$ pkill -f fluentd
```

## Install fluentd-ui
fluentd-ui is a browser-based fluentd and td-agent manager that supports following operations.
* Install, uninstall, and upgrade Fluentd plugins
* start/stop/restart fluentd process
* Configure Fluentd settings such as config file content, pid file path, etc
* View Fluentd log with simple error viewer

### Install fluentd-ui via gem command
```sh
$ gem install -V fluentd-ui
$ fluentd-ui start
Puma 2.9.2 starting...
* Min threads: 0, max threads: 16
* Environment: production
* Listening on tcp://0.0.0.0:9292
```
Then, open http://localhost:9292/ by your browser.
The default account is username=“admin” and password=“changeme”

### Change password
```sh
username: admin
password: fluentdui
```

## Reference
* [Fulentd Quickstart](https://docs.fluentd.org/v1.0/articles/quickstart)
* [Fluentd-UI Quickstart](https://docs.fluentd.org/v1.0/articles/fluentd-ui)
