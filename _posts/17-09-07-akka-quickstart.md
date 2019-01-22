---
title: "Akka Quick Start"
layout: post
date: 2017-09-07 18:55
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- akka
category: blog
author: jiangew
---

<!-- TOC -->

- [Vert.x Slogon](#vertx-slogon)
- [Akka Slogon](#akka-slogon)
- [Actor Model](#actor-model)
- [Actor 目标](#actor-目标)
- [Actor 实现](#actor-实现)
- [Actor vs Goroutine](#actor-vs-goroutine)
- [Akka Actor Quickstart](#akka-actor-quickstart)
    - [定义 Actors 和 Messages](#定义-actors-和-messages)
    - [创建 Actors](#创建-actors)
    - [异步通信](#异步通信)
    - [Running](#running)
- [参考](#参考)

<!-- /TOC -->

最近刚使用Vert.x重构了一个项目，被Vert.x高性能、全异步、Reactive编程模型震撼到了；同样是实现了Actor模型的另一个框架Akka，也很优秀，两个框架Slogon都很神似，最近准备研究对比下两个框架的实现。这篇文章只是简单的akka快速了解和上手。

## Vert.x Slogon
Vert.x is a toolkit for building reactive applications on the JVM.

## Akka Slogon
Akka is a toolkit for building highly concurrent, distributed, and resilient message-driven applications for Java and Scala.

## Actor Model
Actor对没接触过这个概念的人可能不太好理解，Actor的概念其实和OO里的对象类似，是一种抽象。面对对象编程对现实的抽象是对象=属性+行为（method），但当使用方调用对象行为（method）的时候，其实占用的是调用方的CPU时间片，是否并发也是由调用方决定的。这个抽象其实和现实世界是有差异的。现实世界更像Actor的抽象，互相都是通过异步消息通信的。

所以Actor有以下特征：
* Processing: Actor可以做计算的，不需要占用调用方的CPU时间片，并发策略也是由自己决定；
* Storage: Actor可以保存状态；
* Communication: Actor之间可以通过发送消息通讯；

Actor遵循以下规则：
* 发送消息给其他的Actor
* 创建其他的Actor
* 接受并处理消息，修改自己的状态

## Actor 目标
* Actor可独立更新，实现热升级。因为Actor互相之间没有直接的耦合，是相对独立的实体，可能实现热升级。
* 无缝弥合本地和远程调用 因为Actor使用基于消息的通讯机制，无论是和本地的Actor，还是远程Actor交互，都是通过消息，这样就弥合了本地和远程的差异。
* 容错Actor之间的通信是异步的，发送方只管发送，不关心超时以及错误，这些都由框架层和独立的错误处理机制接管。
* 易扩展，天然分布式 因为Actor的通信机制弥合了本地和远程调用，本地Actor处理不过来的时候，可以在远程节点上启动Actor然后转发消息过去。

## Actor 实现
* Erlang/OTP Actor模型的标杆，其他的实现基本上都一定程度参照了Erlang的模式。实现了热升级以及分布式。
* Akka（Scala,Java）基于线程和异步回调模式实现。由于Java中没有Fiber，所以是基于线程的。为了避免线程被阻塞，Akka中所有的阻塞操作都需要异步化。要么是Akka提供的异步框架，要么通过Future-callback机制，转换成回调模式。实现了分布式，但还不支持热升级。
* Quasar (Java) 为了解决Akka的阻塞回调问题，Quasar通过字节码增强的方式，在Java中实现了Coroutine/Fiber。同时通过ClassLoader的机制实现了热升级。缺点是系统启动的时候要通过javaagent机制进行字节码增强。

## Actor vs Goroutine
Goroutine很大程度上降低了并发的开发成本，是不是我们所有需要并发的地方直接 *go func* 就搞定了呢？
> Go通过Goroutine的调度解决了CPU利用率的问题。但遇到其他的瓶颈资源如何处理？比如带锁的共享资源，比如数据库连接等。互联网在线应用场景下，如果每个请求都扔到一个Goroutine里，当资源出现瓶颈的时候，会导致大量的Goroutine阻塞，最后用户请求超时。这时候就需要用Goroutine池来进行控流，同时问题又来了：池子里设置多少个Goroutine合适？所以这个问题还是没有从更本上解决。

## Akka Actor Quickstart

### 定义 Actors 和 Messages
- 可以遵照领域模型

### 创建 Actors
- 通过Factory创建实例，返回的是ActorRef，指向Actor实例；这种间接级别增大了分布式系统的灵活性；
- Location Transparency: ActorRef可以在保留相同语义的同时，表示正在运行的Actor在进程或远程机器上的实例；
- ActorSystem: 类似Spring BeanFactory，作为Actor的容器并管理其生命周期；
- Actor 和 ActorSystem: 命名要有意义，最好与你的领域模型一致。

### 异步通信
- Actors are reactive and message driven.
- 保证同一个Actor的消息队列是有序的；不同Actor的消息是交织的。

### Running
```sh
    mvn compile exec:exec
    mvn test
```

## 参考
* [Akka Quickstart with Java](http://developer.lightbend.com/guides/akka-quickstart-java/?_ga=2.50799274.2004120847.1504782713-1095006924.1489455612)
