---
title: "Actor 编程模型浅谈"
layout: post
date: 2019-01-16 15:30
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Actor
- Akka
- Vert.x
- Erlang
category: blog
author: jiangew
---

<!-- TOC -->

- [Actor 模型背景](#actor-模型背景)
- [Actor 模型特点](#actor-模型特点)
- [Akka vs Netty](#akka-vs-netty)
- [Actor Model](#actor-model)
    - [What: Actor Model](#what-actor-model)
    - [How: Actor Model](#how-actor-model)
    - [Who: Actor Model](#who-actor-model)
    - [What: Akka](#what-akka)
        - [Akka 实现了独特的混合模型](#akka-实现了独特的混合模型)
    - [What: Vert.x](#what-vertx)
        - [Actor Arch](#actor-arch)
        - [EventBus](#eventbus)
        - [并发编程框架](#并发编程框架)
        - [异步IO编程框架](#异步io编程框架)
        - [响应式编程](#响应式编程)
        - [集群](#集群)
    - [How: Vert.x Actor Model](#how-vertx-actor-model)
    - [What: Vert.x EventBus](#what-vertx-eventbus)
    - [Resources](#resources)

<!-- /TOC -->

## Actor 模型背景

Actor 模型(Actor model)首先是由Carl Hewitt在1973定义， 由Erlang OTP(Open Telecom Platform)推广，其消息传递更加符合面向对象的原始意图。Actors属于并发组件模型，通过组件方式定义并发编程范式的高级阶段，避免使用者直接接触多线程并发或线程池等基础概念。

流行语言并发是基于多线程之间的共享内存，使用同步方法防止写争夺，Actors使用消息模型，每个Actors在同一时间处理最多一个消息，可以发送消息给其他Actors，保证了[单独写原则](https://www.jdon.com/performance/singlewriter.html)。从而巧妙避免了多线程写争夺。

## Actor 模型特点

* 隔离计算实体
* Share Nothing
* 没有任何地方同步
* 异步消息传递
* 不可变的消息 消息模型类似 MailBox / Queue

 AKKA框架是一个实现Actors模型的平台，灵感来自Erlang，能更轻松地开发可扩展，实现多线程安全应用。支持Java和Scala。

 Actors是一个轻量级的对象，通过发送消息实现交互。每个Actors在同一时间处理最多一个消息，可以发送消息给其他Actors。在同一时间可以于一个Java虚拟机存在数以百万计的参与者，架构是一个父子层结构，其中父层监控子层的行为。还可以很容易地扩展Actor运行在集群中各个节点之间，无需修改代码。每个演员都可以有内部状态（字段/变量），但通信只能通过消息传递，不会有共享数据结构（计数器，队列）。

## Akka vs Netty

Akka是一个建立在Actors概念和可组合Futures之上的并发框架，Akka设计灵感来源于Erlang，Erlang是基于Actor模型构建的。它通常被用来取代阻塞锁如同步、读写锁及类似的更高级别的异步抽象。

Netty是一个异步网络库，使JAVA NIO的功能更好用。

Akka针对IO操作有一个抽象，这和Netty类似。使用Akka可以用来部署计算集群，Actor在不同的机器之间传递消息。从这个角度来看，Akka相对于Netty来说，是一个更高层次的抽象。

## Actor Model

### What: Actor Model

每个Actor可认为一个EventLoop可简单理解为线程（实际可能多个Actor共享线程）。它有以下组件：
* 事件信箱
* 内部状态

Actor EventLoop 的状态有：
* 等待信箱事件
* 处理事件中

研发会使用异步IO编程来实现具体的Actor。即，具体Actor关心的IO读写事件也会放入Actor自己的信箱。处于【处理事件中】的事件消费程序，不会有IO引起的Block等待。

### How: Actor Model

* 简化多线程编程：单线程的Actor程序，可以保证内部状态不并发访问。不再需要考虑多线程共享状态下的各种问题。
* 天然分布式设计：Actor间基于消息的通讯，天然地支持分布式的部署，如果使用合适的Actor容器，多Actor实例的管理等，天然地实现了“微服务”架构。
* HA：Actor容器可以用统一的健康探测消息，来测试Actor的健康情况。需要时可以重启新的Actor。
* 其它 Event Driven Arch（EDA）的好处。

### Who: Actor Model

* Erlang
* Akka
* Vert.x
* ...

### What: Akka

Akka支持可扩展的实时事务处理。我们相信编写出正确的、具有容错性和可扩展性的并发程序太困难了。这多数是因为使用了错误的工具和错误的抽象级别。Akka就是为了改变这种状况而生的。通过使用Actor模型提升了抽象级别，为构建可扩展的、有弹性的响应式并发应用提供了一个更好的平台，详见[响应式宣言](https://www.reactivemanifesto.org/)。在容错性方面Akka采用了“let 
it crash”（让它崩溃）模型，该模型已经在电信行业构建出“自愈合”的应用和永不停机的系统，取得了巨大成功。Actor还为透明的分布式系统以及真正的可扩展高容错应用的基础进行了抽象。

一个Actor是一个容器，它包含了状态，行为，一个邮箱，子Actor和一个监管策略。所有这些封装在一个Actor引用里。最终在Actor终止时，会有这些发生。

Actor使你能够进行服务失败管理（监控），负载管理（缓和策略、超时和处理隔离），以及水平和垂直方向上的可扩展性（增加cpu核数和/或增加更多的机器）管理。

#### Akka 实现了独特的混合模型

Actors
* 对并发/并行程序的简单的、高级别的抽象；
* 异步、非阻塞、高性能的事件驱动编程模型；
* 非常轻量的事件驱动处理（1G内存可容纳数百万个actors）；

容错性
* 使用“let-it-crash”语义的监控层次体系；
* 监控层次体系可以跨越多个JVM，从而提供真正的容错系统；
* 非常适合编写永不停机、自愈合的高容错系统；

位置透明性
* Akka的所有元素都为分布式环境而设计：所有actor只通过发送消息进行交互，所有操作都是异步的。

持久性
* Actor接收到的消息可以选择性的被持久化，并在actor启动或重启的时候重放。这使得actor能够恢复其状态，即使是在JVM崩溃或正在迁移到另外节点的情况下。


### What: Vert.x

有人说它是java界的Node.js。但它自称性能上，多语言上超越Node.js。他是个异步框架，Web框架，有EventBus，支持Actor模式，支持微服务管理和发现等，听起来十分牛逼的样子。

#### Actor Arch

Actor 在 Vert.x 中叫 [Verticle](http://vertx.io/docs/vertx-core/java/#_verticles)

#### EventBus

事件总线，支持事件广播、点到点发送、点对点发送与回复 [EventBus](http://vertx.io/docs/vertx-core/java/#event_bus)

#### 并发编程框架

* inline异步任务 [Verticle中异步调用阻塞代码的类似ForkJoinPool]
* Future的串并联使用，异步任务与结果的封装
* 好莱坞原则的异步Callback编程 [Hollywood principle:"Don't call me; I'll call you.]

#### 异步IO编程框架

![AIO 01](/assets/images/post/20190116/aio-01.png) <br>
![AIO 02](/assets/images/post/20190116/aio-02.png) <br>

#### 响应式编程

如果你认为OOP或好莱坞原则的异步Callback编程已经OUT了，或者已经被Callback Hell坑过。那么可以试试Reactive。Vert.x支持 [RxJava](https://vertx.io/docs/vertx-rx/java/)

#### 集群

跨JVM的集群EventBus，基于hazelcast来实现集群的发现和消费订阅的注册功能。[Vert.x Cluster Manager](http://vertx.io/docs/vertx-core/java/#_cluster_managers)
![Vert.x Cluster](/assets/images/post/20190116/cluster-01.png) <br>

### How: Vert.x Actor Model

Actor模式在Vert.x中是这样体现的：
* Actor --> Verticle
* 信箱 --> 一般的事件，如EventBus消息到达池。还有其它事件，如Http/Websocket/异步JDBC请求或事件池、inline异步任务（Verticle中异步调用阻塞代码的类似ForkJoinPool）的结果池。
* EventLoop --> Actor的信箱消费线程

![Vert.x Actor Model 01](/assets/images/post/20190116/vertx-model-01.png) <br>
![Vert.x Actor Model 02](/assets/images/post/20190116/vertx-model-02.png) <br>

### What: Vert.x EventBus

Vert.x EventBus是个普通的无中心消息框架:
* 事件广播；
* 点到点发送；
* 点对点发送与回复[ACK]；
* 事件的发生源模块，事件的监听模块完全解藕；
* 事件可以支持广播、路由、点对点等方法传递。生产者不关心订阅者的数量、实现逻辑；
* 生产者与订阅者可以关注自己的Subject。生产者与订阅者间没有相互的依赖关系；
* SLA: 消息没有持久化，可能丢失。消息的投递以最大努力为原则，即不保证成功；
* 支持机器间EventBus的集群共享（可选基于hazelcast），消息和Subject可以在集群中传播；
* 在分布式部署架构下支持功能和服务热插拔，HA要求的情况下可以灵活支撑；
* 集群自治，同时支持集群成员的注册和发现、异常容错；

Vert.x EventBus是个打通到客户端的消息框架，是个All in one工具集。由于它包含一个 EventBus SockJS Bridge 模块，浏览器可以通过它提供的JS库，使用sockjs协议（Server-Sent Events/Websocket/HTTP poll）来接入EventBus。后端的事件可以实时通知到浏览器（提供event bus javascript库），实时推浏览器的广播变得相对容易实现。想想一个长后台任务完成，或监控事件触发时，一切都变得实时响应到用户端。浏览器也可以向后端发送异步指令和请求。[EventBus Bridge](http://vertx.io/docs/vertx-web/java/#_sockjs_event_bus_bridge)

### Resources

* [PayPal 如何使用8个VM每天支撑数十亿个事务](https://www.jdon.com/48257)
* [PayPal 如何使用8个VM每天支撑数十亿个事务 原文](http://highscalability.com/blog/2016/8/15/how-paypal-scaled-to-billions-of-transactions-daily-using-ju.html)
* [PayPal Akka Streams & Akka HTTP for Large-Scale Production Deployments](http://paypal.github.io/squbs/)
* [Lightbend Platform Tech Hub](https://developer.lightbend.com/guides/?_ga=2.3489849.888347207.1547346978-1017804577.1547100744)
