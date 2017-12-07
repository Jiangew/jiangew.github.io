---
title: "Akka Quick Start"
layout: post
date: 2017-09-07 18:55
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- akka
category: blog
author: JamesiWorks
---

Table of Contents
=================

   * [Table of Contents](#table-of-contents)
      * [Vert.x Slogon](#vertx-slogon)
      * [Akka Slogon](#akka-slogon)
      * [定义 Actors 和 Messages](#定义-actors-和-messages)
      * [创建 Actors](#创建-actors)
      * [异步通信](#异步通信)
      * [Running](#running)
      * [参考](#参考)

最近刚使用Vert.x重构了一个项目，被Vert.x高性能、全异步、Reactive编程模型震撼到了；同样是实现了Actor模型的另一个框架Akka，也很优秀，两个框架Slogon都很神似，最近准备研究对比下两个框架的实现。这篇文章只是简单的akka快速了解和上手。

## Vert.x Slogon
Vert.x is a toolkit for building reactive applications on the JVM.

## Akka Slogon
Akka is a toolkit for building highly concurrent, distributed, and resilient message-driven applications for Java and Scala.

## 定义 Actors 和 Messages
- 可以遵照领域模型

## 创建 Actors
- 通过Factory创建实例，返回的是ActorRef，指向Actor实例；这种间接级别增大了分布式系统的灵活性；
- Location transparency: ActorRef可以再保留相同语义的同时，表示正在运行的Actor在进程或远程机器上的实例；
- ActorSystem: 类似Spring BeanFactory，作为Actor的容器并管理其生命周期；
- Actor 和 ActorSystem: 命名要用意义，最好与你的领域模型一致。

## 异步通信
- Actors are reactive and message driven.
- 保证同一个Actor的消息队列是有序的；不同Actor的消息是交织的。

## Running
```sh
    mvn compile exec:exec
    mvn test
```

## 参考
- [Akka Quickstart with Java](http://developer.lightbend.com/guides/akka-quickstart-java/?_ga=2.50799274.2004120847.1504782713-1095006924.1489455612)
