---
title: "Actor vs CSP 并发模型随笔"
layout: post
date: 2019-01-17 15:30
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Actor
- Akka
- Golong
- CSP
category: blog
author: jiangew
---


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Akka/Erlang 的 Actor 模型与 Go 语言的协程 Goroutine 与通道 Channel 代表的 CSP模型（Communicating Sequential Processes）区别](#akkaerlang-的-actor-模型与-go-语言的协程-goroutine-与通道-channel-代表的-csp模型communicating-sequential-processes区别)
  - [Actor 模型描述了一组为了避免并发编程常见问题的公理](#actor-模型描述了一组为了避免并发编程常见问题的公理)
  - [Channel 模型](#channel-模型)
  - [Actor vs Channel](#actor-vs-channel)
- [Actor 模型](#actor-模型)
  - [Actor 特征](#actor-特征)
  - [Actor 遵循规则](#actor-遵循规则)
  - [Actor 的目标](#actor-的目标)
  - [Actor 的实现](#actor-的实现)
- [Goroutine 调度器](#goroutine-调度器)
- [Akka](#akka)
  - [定义 Actors 和 Messages](#定义-actors-和-messages)
  - [创建 Actors](#创建-actors)
  - [异步通信](#异步通信)
  - [Running](#running)
- [事件溯源 Event Sourcing](#事件溯源-event-sourcing)
- [理解 CQRS](#理解-cqrs)

<!-- /code_chunk_output -->

## Akka/Erlang 的 Actor 模型与 Go 语言的协程 Goroutine 与通道 Channel 代表的 CSP模型（Communicating Sequential Processes）区别

### Actor 模型描述了一组为了避免并发编程常见问题的公理
* 所有Actor状态都是本地的，外部无法访问；
* Actor之间只有通过消息传递进行通信；
* 一个Actor可以响应消息，推出新的Actor并改变其内部状态，或将消息发送到一个或多个Actor参与者；
* Actor可能会阻塞自己，但不应该阻塞它运行的线程；

### Channel 模型
* Worker 之间不直接彼此通信，而是通过不同Channel进行消息发布和侦听，消息的发送者和接受者之间通过Channel松耦合。
* Golang CSP 模型是由协程 Goroutine 与通道 Channel 实现。
* Goroutine 是一种轻量级线程，他不是操作系统的线程，而是将操作系统线程分段使用，通过调度器实现协作式调度。是一种绿色线程，微线程，与 Coroutine 协程也有区别，能够在发现阻塞后启动新的微线程。
* 通道 Channel: 类似 Unix Pipe，用于协程之间通信和同步；协程之间解耦，但是跟Channel耦合。

### Actor vs Channel
* 都是描述独立的流程通过消息传递进行通信；区别在于CSP模型消息交换是同步的（两个流程的执行接触点），而Actor模型是完全解耦的，可以在任意时间将消息发送给任何接受者；由于Actor享有更大的独立性，他可以根据自己的状态选择性处理传入的消息，自主性更大。
* Golang中为了不阻塞流程，程序必须检查不同的传入消息，以便预见确保正确的顺序。
* CSP模型好处是Channel不需要缓冲消息，而Actor模型理论上需要一个无限大小的邮箱作为消息缓冲队列。


## Actor 模型
Actor对没接触过这个概念的人可能不太好理解，Actor的概念其实和OO里的对象类似，是一种抽象。面对对象编程对现实的抽象是对象=属性+行为（method），但当使用方调用对象行为（method）的时候，其实占用的是调用方的CPU时间片，是否并发也是由调用方决定的。这个抽象其实和现实世界是有差异的。现实世界更像Actor的抽象，互相都是通过异步消息通信的。

### Actor 特征
* Processing – actor可以做计算的，不需要占用调用方的CPU时间片，并发策略也是由自己决定。
* Storage – actor可以保存状态
* Communication – actor之间可以通过发送消息通讯

### Actor 遵循规则
* 发送消息给其他的Actor
* 创建其他的Actor
* 接受并处理消息，修改自己的状态

### Actor 的目标
* Actor可独立更新，实现热升级。因为Actor互相之间没有直接的耦合，是相对独立的实体，可能实现热升级。
* 无缝弥合本地和远程调用 因为Actor使用基于消息的通讯机制，无论是和本地的Actor，还是远程Actor交互，都是通过消息，这样就弥合了本地和远程的差异。
* 容错Actor之间的通信是异步的，发送方只管发送，不关心超时以及错误，这些都由框架层和独立的错误处理机制接管。
* 易扩展，天然分布式 因为Actor的通信机制弥合了本地和远程调用，本地Actor处理不过来的时候，可以在远程节点上启动Actor然后转发消息过去。

### Actor 的实现
* Erlang/OTP Actor模型的标杆，其他的实现基本上都一定程度参照了Erlang的模式。实现了热升级以及分布式。
* Akka（Scala,Java）基于线程和异步回调模式实现。由于Java中没有Fiber，所以是基于线程的。为了避免线程被阻塞，Akka中所有的阻塞操作都需要异步化。要么是Akka提供的异步框架，要么通过Future-callback机制，转换成回调模式。实现了分布式，但还不支持热升级。
* Quasar (Java) 为了解决Akka的阻塞回调问题，Quasar通过字节码增强的方式，在Java中实现了Coroutine/Fiber。同时通过ClassLoader的机制实现了热升级。缺点是系统启动的时候要通过javaagent机制进行字节码增强。


## Goroutine 调度器
Goroutine很大程度上降低了并发的开发成本，是不是我们所有需要并发的地方直接 go func 就搞定了呢？
Go通过Goroutine的调度解决了CPU利用率的问题。但遇到其他的瓶颈资源如何处理？比如带锁的共享资源，比如数据库连接等。互联网在线应用场景下，如果每个请求都扔到一个Goroutine里，当资源出现瓶颈的时候，会导致大量的Goroutine阻塞，最后用户请求超时。这时候就需要用Goroutine池来进行控流，同时问题又来了：池子里设置多少个Goroutine合适？

所以这个问题还是没有从更本上解决。


## Akka

### 定义 Actors 和 Messages
- 可以遵照领域模型

### 创建 Actors
- 通过Factory创建实例，返回的是ActorRef，指向Actor实例；这种间接级别增大了分布式系统的灵活性；
- Location transparency: ActorRef可以再保留相同语义的同时，表示正在运行的Actor在进程或远程机器上的实例；
- ActorSystem: 类似Spring BeanFactory，作为Actor的容器并管理其生命周期；
- Actor 和 ActorSystem: 命名要用意义，最好与你的领域模型一致；

### 异步通信
- Actors are reactive and message driven.
- 保证同一个Actor的消息队列是有序的；不同Actor的消息是交织的。

### Running
- mvn compile exec:exec
- mvn test


## 事件溯源 Event Sourcing
「事件溯源」保证了数据状态的变换都以一系列的事件的形式存储在数据库中。所以，我们不仅可以获取每个变换的事件，而且可以通过过去的事件来组合出过去任意时刻的数据状态！

注意，有一点很重要，我们不能更改已经保存的事件以及它们的序列 —— 也就是说，事件存储是只能添加而不能删除的，并且需要不可变。是不是感觉和数据库事务日志的原理差不多呢？

在微服务架构中，事件溯源模式可以带来以下的好处：
* 我们可以从过去的事件序列中组建出任意时刻的数据状态
* 每个过去的事件都得以保存，因此这使得补偿事务成为可能
* 我们可以从事件存储中获取事件流，并且以异步、响应式风格对其进行变换和处理
* 事件存储同样可以当作为数据日志


## 理解 CQRS
CQRS 目前在 DDD 领域使用广泛，甚至可以称为一种架构风格，可以取得与 MapReduce、Rest 同等的地位。

CQRS: Command Query Responsibility Seperation（命令查询职责分离），命令与查询的分离，可以更好的控制请求者的操作。查询操作不会造成数据变更，属于一种幂等操作，可以提供缓存，提高查询性能。

从请求响应的角度看，查询操作需要同步请求，实时返回结果；命令操作则不然，我们并不期待命令操作必须实时返回结果，可以采用 free-and-forget 方式，这种方式是运用异步操作的前提。

CQRS 模式风格就是基于事件的异步状态机模型，核心概念是 Command 和 Event。
