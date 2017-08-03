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

### 原理简介
假设有一个Event Bus服务接口：
```java
/**
 * Author: Jiangew
 * Date: 06/07/2017
 */
@ProxyGen // Generate the proxy and handler
@VertxGen // Generate clients in non-java languages
public interface ProcessorService {

    int NO_NAME_ERROR = 10001;
    int BAD_NAME_ERROR = 10002;

    /**
     * factory methods to create an instance
     *
     * @param vertx
     * @return
     */
    static ProcessorService create(Vertx vertx) {
        return new ProcessorServiceImpl(vertx);
    }

    /**
     * factory methods to create a proxy
     *
     * @param vertx
     * @return
     */
    static ProcessorService createProxy(Vertx vertx, String address) {
        // Alternatively, you can create the proxy directly using:
        // return new ProcessorServiceVertxEBProxy(vertx, address);
        // The name of the class to instantiate is the service interface + `VertxEBProxy`.
        // This class is generated during the compilation.

        return ProxyHelper.createProxy(ProcessorService.class, vertx, address);
    }

    /**
     * process
     *
     * @param document
     * @param resultHandler
     */
    void process(JsonObject document, Handler<AsyncResult<JsonObject>> resultHandler);

    /**
     * fluent process
     *
     * @param id
     * @param resultHandler
     * @return
     */
    @Fluent
    ProcessorService fluentProcess(String id, Handler<AsyncResult<JsonObject>> resultHandler);

}
```
这里定义了一个异步方法process，其异步调用返回的结果是AsyncResult<JsonObject>类型的。由于异步RPC底层通过Clustered Event Bus进行通信，我们需要给器指定一个通信地址SERVICE_ADDRESS。@Fluent注解代表此方法返回自身，便于进行组合。我们同时还提供了两个辅助方法：createService方法用于创建服务实例，而createProxy方法则通过ProxyHelper辅助类创建服务代理实例。<br />

假设服务提供端A注册了一个SomeService类型的服务代理，服务调用端B需要通过异步RPC调用服务的process方法，此时调用端B可以利用ProxyHelper获取服务实例并进行服务调用。B中获取的服务其实是一个服务代理类，而真正的服务实例在A处。何为服务代理？服务代理可以帮助我们向服务提供端发送调用请求，并且响应调用结果。那么如何发送调用请求呢？相信大家能想到，是调用端B将调用参数和方法名称等必要信息包装成集群消息「ClusteredMessage」，然后通过send方法将请求通过Clustered Event Bus发送至服务提供端A处「需要提供此服务的通信地址」。A在注册服务的时候会创建一个MessageConsumer监听此服务的地址来响应调用请求。当接收到调用请求的时候，A会在本地调用方法，并将结果回复至调用端。所以异步RPC本质上其实是一个基于代理模式的 Request/Response 消息模式。<br />

用时序图来描述一下上述过程：
![](http://ote3hl188.bkt.clouddn.com/vertx-async-rpc.png) <br />

### 引入
以ProcessorService接口为例，我们可以在集群中的一个节点上注册服务实例：
```java
service = ProcessorService.create(vertx);
// register the handler
ProxyHelper.registerService(ProcessorService.class, vertx, service, SERVICE_ADDRESS);
```
然后在另一个节点上获取此服务实例的代理，并进行服务调用。调用的时候看起来就像在本地调用(LPC)一样，其实是进行了RPC通信：
```java
/**
 * Author: Jiangew
 * Date: 06/07/2017
 */
public class ConsumerVerticle extends AbstractVerticle {

    public static void main(String[] args) {
        Runner.runExample(ConsumerVerticle.class);
    }

    private final String SERVICE_ADDRESS = "service.provide.processor";

    @Override
    public void start() throws Exception {
        ProcessorService proxyService = ProcessorService.createProxy(vertx, SERVICE_ADDRESS);

        JsonObject document = new JsonObject().put("name", "vertx");

        proxyService.process(document, (r) -> {
            if (r.succeeded()) {
                System.out.println(r.result().encodePrettily());
            } else {
                System.out.println(r.cause());
                Failures.handleFailure(r.cause());
            }
        });
    }

}
```
其实，这里获取到的proxyService实例的真正类型是Vert.x自动生成的服务代理类ProcessorServiceVertxEBProxy类，里面封装了通过Event Bus进行通信的逻辑。我们首先来讲一下Service Proxy生成代理类的命名规范。

### 代理类命名规范
Vert.x Service Proxy在生成代理类时遵循一定的规范。假设有一Event Bus服务接口ProcessorService，Vert.x会自动为其生成代理类以及代理处理器：
* 代理类的命名规范为 接口名 + VertxEBProxy。比如ProcessorService接口对应的代理类名称为ProcessorServiceVertxEBProxy
* 代理类会继承原始的服务接口并实现所有方法的代理逻辑
* 代理处理器的命名规范为 接口名 + VertxProxyHandler。比如ProcessorService接口对应的代理处理器名称为ProcessorServiceVertxProxyHandler
* 代理处理器会继承ProxyHandler抽象类
ProxyHelper辅助类中注册服务以及创建代理都是遵循了这个规范。

### 在Event Bus上注册服务
我们通过ProxyHelper辅助类中的registerService方法来向Event Bus上注册Event Bus服务，来看其具体实现：
```java
public static <T> MessageConsumer<JsonObject> registerService(Class<T> clazz, Vertx vertx, T service, String address,
                                                              boolean topLevel,
                                                              long timeoutSeconds) {
  String handlerClassName = clazz.getName() + "VertxProxyHandler";
  Class<?> handlerClass = loadClass(handlerClassName, clazz);
  Constructor constructor = getConstructor(handlerClass, Vertx.class, clazz, boolean.class, long.class);
  Object instance = createInstance(constructor, vertx, service, topLevel, timeoutSeconds);
  ProxyHandler handler = (ProxyHandler) instance;
  return handler.registerHandler(address);
}
```
首先根据约定生成对应的代理Handler的名称，然后通过类加载器加载对应的Handler类，再通过反射来创建代理Handler的实例，最后调用handler的registerHandler方法注册服务地址。<br />

registerHandler方法的实现在Vert.x生成的各个代理处理器中。以之前的ProcessorService为例，我们来看一下其对应的代理处理器ProcessorServiceVertxProxyHandler实现。首先是注册并订阅地址的registerHandler方法：
```java
public MessageConsumer<JsonObject> registerHandler(String address) {
  MessageConsumer<JsonObject> consumer = vertx.eventBus().<JsonObject>consumer(address).handler(this);
  this.setConsumer(consumer);
  return consumer;
}
```
registerHandler方法的实现非常简单，就是通过consumer方法在address地址上绑定了ProcessorServiceVertxProxyHandler自身。那么ProcessorServiceVertxProxyHandler是如何处理来自服务调用端的服务调用请求，并将调用结果返回到请求端呢？在回答这个问题之前，我们先来看看代理端「调用端」是如何发送服务调用请求的，这就要看对应的服务代理类的实现了。

### 服务调用
我们来看一下服务调用端是如何发出服务调用请求的消息的。之前已经介绍过，服务调用端是通过Event Bus的send方法发送调用请求的，并且会提供一个replyHandler来等待方法调用的结果。调用的方法名称会存放在消息中名为action的header中。以之前ProcessorService的代理类ProcessorServiceVertxEBProxy中process方法的请求为例：
```java
public ProcessorService process(String id, Handler<AsyncResult<JsonObject>> resultHandler) {
  if (closed) {
    resultHandler.handle(Future.failedFuture(new IllegalStateException("Proxy is closed")));
    return this;
  }
  JsonObject _json = new JsonObject();
  _json.put("id", id);
  DeliveryOptions _deliveryOptions = (_options != null) ? new DeliveryOptions(_options) : new DeliveryOptions();
  _deliveryOptions.addHeader("action", "process");
  _vertx.eventBus().<JsonObject>send(_address, _json, _deliveryOptions, res -> {
    if (res.failed()) {
      resultHandler.handle(Future.failedFuture(res.cause()));
    } else {
      resultHandler.handle(Future.succeededFuture(res.result().body()));
    }
  });
  return this;
}
```
可以看到代理类把此方法传入的参数都放到一个JsonObject中了，并将要调用的方法名称存放在消息中名为action的header中。代理方法通过send方法将包装好的消息发送至之前注册的服务地址处，并且绑定replyHandler等待调用结果，然后使用我们传入到process方法中的resultHandler对结果进行处理。是不是很简单呢？

### 服务提供端的调用逻辑
调用请求发出之后，我们的服务提供端就会收到调用请求消息，然后执行ProcessorServiceVertxProxyHandler中的处理逻辑：
```java
public void handle(Message<JsonObject> msg) {
  try {
    JsonObject json = msg.body();
    String action = msg.headers().get("action");
    if (action == null) {
      throw new IllegalStateException("action not specified");
    }
    accessed();
    switch (action) {
      case "process": {
        service.process((java.lang.String)json.getValue("id"), createHandler(msg));
        break;
      }
      default: {
        throw new IllegalStateException("Invalid action: " + action);
      }
    }
  } catch (Throwable t) {
    msg.reply(new ServiceException(500, t.getMessage()));
    throw t;
  }
}
```
handle方法首先从消息header中获取方法名称，如果获取不到则调用失败；接着handle方法会调用accessed方法记录最后调用服务的时间戳，这是为了实现超时的逻辑，后面我们会讲。接着handle方法会根据方法名称分派对应的逻辑，在“真正”的服务实例上调用方法。注意异步RPC的过程本质是 Request/Response 模式，因此这里的异步结果处理函数resultHandler应该将调用结果发送回调用端。此resultHandler是通过createHandler方法生成的，逻辑很清晰：
```
private <T> Handler<AsyncResult<T>> createHandler(Message msg) {
  return res -> {
    if (res.failed()) {
      if (res.cause() instanceof ServiceException) {
        msg.reply(res.cause());
      } else {
        msg.reply(new ServiceException(-1, res.cause().getMessage()));
      }
    } else {
      if (res.result() != null  && res.result().getClass().isEnum()) {
        msg.reply(((Enum) res.result()).name());
      } else {
        msg.reply(res.result());
      }
    }
  };
}
```
这样，一旦在服务提供端的调用过程完成时，调用结果就会被发送回调用端。这样调用端就可以调用结果执行真正的处理逻辑了。

### 超时处理
Vert.x自动生成的代理处理器内都封装了一个简单的超时处理逻辑，它是通过定时器定时检查最后的调用时间实现的。逻辑比较简单，直接放上相关逻辑：
```java
public ProcessorServiceVertxProxyHandler(Vertx vertx, ProcessorService service, boolean topLevel, long timeoutSeconds) {
    this.vertx = vertx;
    this.service = service;
    this.timeoutSeconds = timeoutSeconds;
    try {
      this.vertx.eventBus().registerDefaultCodec(ServiceException.class,
          new ServiceExceptionMessageCodec());
    } catch (IllegalStateException ex) {}
    if (timeoutSeconds != -1 && !topLevel) {
      long period = timeoutSeconds * 1000 / 2;
      if (period > 10000) {
        period = 10000;
      }
      this.timerID = vertx.setPeriodic(period, this::checkTimedOut);
    } else {
      this.timerID = -1;
    }
    accessed();
  }
  private void checkTimedOut(long id) {
    long now = System.nanoTime();
    if (now - lastAccessed > timeoutSeconds * 1000000000) {
      close();
    }
  }
  ```
  一旦超时，就自动调用close方法终止定时器，注销响应服务调用请求的consumer并关闭代理。

  ### 代码是如何生成的？
大家可能会很好奇，这些服务代理类是怎么生成出来的？其实，这都是Vert.x Codegen的功劳。Vert.x Codegen的本质是一个 注解处理器(APT)，它可以扫描源码中是否包含要处理的注解，检查规范后根据响应的模板生成对应的代码，这就是注解处理器的作用(注解处理器于JDK 1.6引入)。为了让Codegen正确地生成代码，我们需要配置编译参数来确保注解处理器能够正常的工作，具体的可以参考：
- [Vert.x Codegen的文档](https://github.com/vert-x3/vertx-codegen/blob/master/README.md)

Vert.x Codegen使用MVEL2作为生成代码的模板，扩展名为*.templ，比如代理类和代理处理器的模板就位于「vert-x3/vertx-service-proxy」中：
- [vert-x3/vertx-service-proxy](https://github.com/vert-x3/vertx-service-proxy/tree/master/src/main/resources/serviceproxy/template) <br />

配置文件类似于这样：
```java
{
  "name": "Proxy",
  "generators": [
    {
      "kind": "proxy",
      "fileName": "ifaceFQCN + 'VertxEBProxy.java'",
      "templateFileName": "serviceproxy/template/proxygen.templ"
    },{
      "kind": "proxy",
      "fileName": "ifaceFQCN + 'VertxProxyHandler.java'",
      "templateFileName": "serviceproxy/template/handlergen.templ"
    }
  ]
}
```
具体的代码生成逻辑还要涉及APT及MVEL2的知识，这里就不展开讲了，有兴趣的朋友可以研究研究Vert.x Codegen的源码。

### 优点与缺点
Vert.x提供的这种Async RPC有着许多优点：
* 通过Clustered Event Bus传输消息，不需引入其它额外的组件
* 自动生成代理类及代理处理器，可以帮助我们做消息封装、传输、编码解码以及超时处理等问题，省掉不少冗余代码，让我们可以以LPC的方式进行RPC通信
* 多语言支持「Polyglot support」。这是Vert.x的一大亮点。只要加上@VertxGen注解并在编译期依赖中加上对应语言的依赖「如vertx-lang-ruby」，Vert.x Codegen就会自动处理注解并生成对应语言的服务代理「通过调用Java版本的服务代理实现」。这样Async RPC可以真正地做到不限language

当然Vert.x要求我们的服务接口必须是 基于回调的，这样写起来可能会不优雅。还好@VertxGen注解支持生成Rx版本的服务类，因此只要加上vertx-rx-java依赖，Codegen就能生成对应的Rx风格的服务类（异步方法返回Observable），这样我们就能以更reactive的风格来构建应用了，岂不美哉？<br />

当然，为了考虑多语言支持的兼容性，Vert.x在传递消息的时候依然使用了传统的JSON，这样传输效率可能不如Protobuf高，但是不一定成为瓶颈。「看业务情况，真正的瓶颈一般还是在DB上」<br />

本文标题: Vert.x 技术内幕 | 异步RPC实现原理
文章作者: sczyh30
发布时间: 2016年09月07日
原始链接: http://www.sczyh30.com/posts/Vert-x/vertx-advanced-async-rpc/
