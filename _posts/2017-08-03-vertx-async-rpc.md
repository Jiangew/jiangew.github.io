---
title: "Vert.x 异步 RPC 实现解析"
layout: post
date: 2017-08-03 15:54
image: /assets/images/markdown.jpg
headerImage: false
tag:
- Vert.x
- Java
- RPC
category: blog
author: JamesiWorks
---

如何利用Vert.x进行RPC通信？<br />
Vert.x提供了一个组件：Vert.x Service Proxy，专门用于进行异步RPC通信「通过Event Bus」。Vert.x Service Proxy会自动生成代理类进行消息的包装与解码、发送与接收以及超时处理，可以为我们省掉不少代码。<br />

传统的RPC想必大家都不陌生，但是传统的RPC有个缺陷：传统的RPC都是阻塞型的，当调用者远程调用服务时需要阻塞着等待调用结果，这与Vert.x的异步开发模式相违背；而且，传统的RPC未对容错而设计。<br />

Vert.x提供了Service Proxy用于进行异步RPC，其底层依托Clustered Event Bus进行通信。我们只需要按照规范编写我们的服务接口「一般称为Event Bus服务」，并加上@ProxyGen注解，Vert.x就会自动为我们生成相应的代理类在底层处理RPC。有了Service Proxy，我们只需给异步方法提供一个回调函数Handler<AsyncResult<T>>，在调用结果发送过来的时候会自动调用绑定的回调函数进行相关的处理，这样就与Vert.x的异步开发模式相符了。由于AsyncResult本身就是为容错而设计的「两个状态」，因此这里的RPC也具有了容错性。<br />

