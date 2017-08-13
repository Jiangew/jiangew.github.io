---
title: "Vert.x 实战"
layout: post
date: 2017-07-14 15:45
image: /assets/images/markdown.jpg
headerImage: false
tag:
- Vert.x
- Java
category: blog
author: JamesiWorks
---

### What is Vert.x ?
Vert.x is a tool-kit for building reactive applications on the JVM.
- [Vert.x Official Wiki](http://vertx.io/docs/)
- [Vert.x Official Examples](https://github.com/vert-x3/vertx-examples)

### Maven 构建 Verticle 并部署
#### 使用 maven-shade-plugin 插件构建 fat-jar 包，包含了所有依赖，可以独立运行的包
```maven
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>2.3</version
        <executions>
            <execution>
               <phase>package</phase>
               <goals>
                   <goal>shade</goal>
               </goals>
               <configuration>
                   <transformers>
                       <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                           <manifestEntries>
                               <Main-Class>io.vertx.core.Launcher</Main-Class>
                               <Main-Verticle>${main.verticle}</Main-Verticle>
                           </manifestEntries>
                       </transformer>
                       <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                           <resource>META-INF/services/io.vertx.core.spi.VerticleFactory</resource>
                       </transformer>
                   </transformers>
                   <artifactSet>
                   </artifactSet>
                   <outputFile>${project.build.directory}/${project.artifactId}-${project.version}-fat.jar</outputFile>
               </configuration>
            </execution>
        </executions>
    </plugin>
```
```sh
    mvn clean package
```
#### 运行 fat-jar
```sh
    java -jar target/maven-verticle-3.4.2-fat.jar
```
#### 通过 -instances 参数部署多个 Verticle 实例，释放多核的能力
```sh
    java -jar target/maven-verticle-3.4.2-fat.jar -instances 8
```
#### 通过 -cluster && -ha 参数部署 Verticle 实例，开启集群和高可用模式
```sh
    java -jar target/maven-verticle-3.4.2-fat.jar -cluster
    java -jar target/maven-verticle-3.4.2-fat.jar -ha
```
#### 通过 -conf 参数部署 Verticle 实例，动态加载配置文件
```sh
    java -jar target/maven-verticle-3.4.2-fat.jar -conf src/conf/conf.json
```

### BlockingHandler | ExecuteBlocking vs WorkerVerticle
每一个阻塞的耗时操作单独 deploy 一个 worker verticle 处理，一个 worker verticle 一直被线程池中的一个线程执行。
#### ExecuteBlocking 示例
```java
    vertx.createHttpServer().requestHandler(request -> vertx.<String>executeBlocking(future -> {
            // do blocking operation
            try {
                Thread.sleep(500);
            } catch (Exception e) {
                // ignore
            }

            String result = "jamesiworks";
            future.complete(result);
        }, res -> {
            if (res.succeeded()) {
                request.response().putHeader("content-type", "text/plain").end(res.result());
            } else {
                res.cause().printStackTrace();
            }
        })).listen(8080);
```
#### BlockingHandler 示例
```java
        // blocking handler && ordered false && future
        router.get("/chaptersAsync").blockingHandler(this::handleGetChaptersAsync, false).failureHandler(this::handleWorkerTimeout);
```

### 动态部署 Verticle 实例
通过动态部署 Verticle 实例，可以指定 DeploymentOptions 的各种属性，可以对比 VertxOptions，包括 workerPoolSize, isWorker, isHA 等。
```java
        // different ways of deploying verticles
        // 01 deploy a verticle and do not wait for it to start
        vertx.deployVerticle("com.qq.reader.ts.verticle.DownloadVerticle");

        // 02 deploy a verticle and wait for it to start
        vertx.deployVerticle("com.qq.reader.ts.verticle.DownloadVerticle", res -> {
            if (res.succeeded()) {
                String deployId = res.result();
                System.out.println("DeployVerticle deployed ok, deployId = " + deployId);
            }
        });

        // 03 deploy a verticle with options
        int core = Runtime.getRuntime().availableProcessors();
        vertx.deployVerticle("com.qq.reader.ts.verticle.DownloadVerticle",
                new DeploymentOptions()
                        .setInstances(core)
                        .setHa(true)
                        .setWorkerPoolName("vertx-work-pool-ts")
                        .setWorkerPoolSize(core * 50)
                        .setMaxWorkerExecuteTime(VertxOptions.DEFAULT_MAX_WORKER_EXECUTE_TIME)
        );    
```

### 日志
可以通过启动 Vert.x 实例时，动态设置环境变量加载日志实例。
```java
/**
 * Author: Jiangew
 * Date: 20/07/2017
 */
public class Launcher extends io.vertx.core.Launcher {

    public static void main(String[] args) {
        // Force to use slf4j
        System.setProperty("vertx.logger-delegate-factory-class-name", "io.vertx.core.logging.SLF4JLogDelegateFactory");

        new Launcher().dispatch(args);
    }

    @Override
    public void beforeStartingVertx(VertxOptions options) {
        // Start dropwizard monitor
        options.setMetricsOptions(
                new DropwizardMetricsOptions()
                        .setEnabled(true)
                        .setJmxEnabled(true)
                        .setJmxDomain("vertx-metrics-minerva")
        );
    }

}
```

### Vert.x 运行时性能指标监控「Dropwizard && Jolokia && Hawtio」
关于 Vert.x 运行态的性能监控，官方提供了 Dropwizard 和 Hawkular 两种开箱即用的工具。本人实践了使用 Dropwizard Metrics 实现 Vert.x 性能统计的过程「当然踩了很多坑」。<br />
开启 Vert.x 的 Metrics 监控有两种方式，如下：
#### 通过运行 fat-jar 时指定参数
```sh
    java -jar xxx-fat.jar
    -Dvertx.metrics.options.enabled=true
    -Dvertx.metrics.options.jmxEnabled=true
    -Dvertx.metrics.options.jmxDomain=vertx-metrics-jew
```
#### 通过设置 VertxOptions 属性
要想通过 VertxOptions 属性赋值方式，需要想办法自定义 Vertx 实例，因为运行时 Vertx 实例已经初始化完成，无法修改 VertxOptions 各属性值。由于项目框架搭建时，我是使用 Main Verticle 去动态 deploy 业务 Verticle，这样可以给业务 Verticle 通过 DeploymentOptions 指定 instances、workPoolSize、ha、cluster 等指标，所以最终选择了扩展 Launcher 启动类来实现。方式如下：
```java
public class Launcher extends io.vertx.core.Launcher {

    public static void main(String[] args) {
        new Launcher().dispatch(args);
    }

    @Override
    public void beforeStartingVertx(VertxOptions options) {
        // Start dropwizard monitor
        options.setMetricsOptions(
                new DropwizardMetricsOptions()
                        .setEnabled(true)
                        .setJmxEnabled(true)
                        .setJmxDomain("vertx-metrics-jew")
        );
    }

}
```
动态部署 Verticle 实例，并指定 instances、workPoolSize、ha、cluster 等指标。
```java
public class DeployVerticle extends AbstractVerticle {

    public static void main(String[] args) {
        Runner.runExample(DeployVerticle.class);
    }

    @Override
    public void start() {
        System.out.println("Main verticle has started, let's deploy some others ...");

        // 03 deploy a verticle with options
        int core = Runtime.getRuntime().availableProcessors();
        vertx.deployVerticle("com.qq.reader.ts.verticle.DownloadVerticle",
                new DeploymentOptions()
                        .setInstances(core)
                        .setHa(true)
                        .setWorkerPoolName("vertx-work-pool-ts")
                        .setWorkerPoolSize(core * 50)
                        .setMaxWorkerExecuteTime(VertxOptions.DEFAULT_MAX_WORKER_EXECUTE_TIME)
        );

    }
}
```
别忘了 maven 依赖
```maven
    <dependency>
            <groupId>io.vertx</groupId>
            <artifactId>vertx-dropwizard-metrics</artifactId>
            <version>${vertx.version}</version>
    </dependency>
```
#### 下载 Jolokia，用来获取运行时 Vert.x 实例的性能指标
去官网下载 jolokia agent 包，通过「java -javaagent:」方式运行 fat-jar Vert.x 项目，并启动 Jolokia 监控，执行 jolokia 端口和 Vert.x 实例的 host，jolokia-agent.jar 务必是绝对路径。
```sh
    java -javaagent:/.../jolokia-jvm-1.3.7-agent.jar=port=8888,host=localhost -jar xxx-fat.jar
```
或
```sh
    java -javaagent:/.../jolokia-jvm-1.3.7-agent.jar=port=8888,host=localhost -jar xxx-fat.jar
    -Dvertx.metrics.options.enabled=true
    -Dvertx.metrics.options.jmxEnabled=true
    -Dvertx.metrics.options.jmxDomain=vertx-metrics-jew
```
参数设置：
    Dropwizard Metrics 用于指标收集 <br />
    Jmx 暴露桥接接口 <br />
    Jolokia 提供指标数据 Rest 接口 <br >
备注：部署到服务器环境时，不要忘记把 Host「local」换成「0.0.0.0」，否则内部端口不同，不能外部监听端口。
#### 下载 Hawtio，Hawtio 可以远程连接 Jolokia，以图形化形式监控 Vert.x 运行时状态
去官网 hawtio app 包，下载有两种方式启动 Hawtio，一种是以 jar 包的形式运行，一种是以 war 包的形式运行。<br />
jar 包方式启动，指定一个未被占用的端口。
```sh
    java -jar hawtio-app-1.5.2.jar -p 9999
```
#### 基于 Tomcat 部署监控图形化
war 包方式启动，将下载的 hawtio-default-1.5.2.war 重命名为 hawtio.war，放入 tomcat 的 webapps 目录，重新启动 tomcat 即可通过 http://localhost:8080/hawtio/ 来访问。<br />
修改 tomcat 监听和转发端口配置，「tomcat/conf/server.xml」：
```xml
    <Connector port="9999" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
```
#### 开启 Tomcat 登录鉴权
修改 tomcat 鉴权用户配置，「tomcat/conf/tomcat-users.xml」：
```xml
    <role rolename="manager"/>
    <user username="jiangew" password="123456" roles="manager"/>
```
开启 tomcat 鉴权有两种方式，一种是通过在 tomcat 启动脚本中加入环境变量设置，一种是修改 hawtio 的配置文件。<br />
方式一：修改 tomcat 启动脚本，在「tomcat/bin/catalina.sh」中加入：
```sh
    export CATALINA_OPTS='-Dhawtio.authenticationEnabled=true -Dhawtio.role=manager'
```
方式二：修改部署在 tomcat 中的 hawtio war 的配置文件，在「tomcat/webapps/hawtio/WEB-INF/web.xml」中修改：
```xml
    <env-entry>
        <description>Enable/disable hawtio's authentication filter, value is really a boolean</description>
        <env-entry-name>hawtio/authenticationEnabled</env-entry-name>
        <env-entry-type>java.lang.String</env-entry-type>
        <env-entry-value>true</env-entry-value>
    </env-entry>

    <env-entry>
        <description>Authorized user role, empty string disables authorization</description>
        <env-entry-name>hawtio/role</env-entry-name>
        <env-entry-type>java.lang.String</env-entry-type>
        <env-entry-value>manager</env-entry-value>
    </env-entry>
```
还可以设置 jolokia agent 代理服务器 ip 白名单：
```xml
    <servlet>
        <servlet-name>jolokia-proxy</servlet-name>
        <servlet-class>io.hawt.web.ProxyServlet</servlet-class>
        <init-param>
            <param-name>proxyWhitelist</param-name>
            <param-value>
                localhost,
                127.0.0.1,
                10.211.0.199,
                10.211.0.211
            </param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>jolokia-proxy</servlet-name>
        <url-pattern>/proxy/*</url-pattern>
    </servlet-mapping>
```
#### 图形化监控面板示例
图形化监控界面启动后，设置监控信息，并保存，可以看到各个维度的监控数据，很赞。。。 <br />
Portal Menu <br />
![](http://ote3hl188.bkt.clouddn.com/hawtio-01.jpg) <br />
配置 Vert.x 监控实例 <br />
![](http://ote3hl188.bkt.clouddn.com/hawtio-02.jpg) <br />
配置指定 Port 和 Host <br />
![](http://ote3hl188.bkt.clouddn.com/hawtio-03.jpg) <br />
JMX Metrics 监控 <br />
![](http://ote3hl188.bkt.clouddn.com/hawtio-04.jpg) <br />
Dashboard <br />
![](http://ote3hl188.bkt.clouddn.com/hawtio-05.jpg) <br />
#### Using Jolokia and JMX4Perl to expose metrics to Nagios
```sh
    java -javaagent:/.../jolokia-jvm-1.3.7-agent.jar=port=8888,host=localhost -jar xxx-fat.jar ...
    check_jmx4perl --url http://127.0.0.1:8888/jolokia --name eventloops --mbean vertx:name=vertx.event-loop-size --attribute Value --warning 4
```
#### Jolokia && Hawtio 监控参考
- [Vert.x Dropwizard Metrics Use Jolokia and Hawtio](http://vertx.io/docs/vertx-dropwizard-metrics/java/)
- [Jolokia](https://jolokia.org/)
- [Hawtio](http://hawt.io/getstarted/index.html)
- [Hawtio Configuration](http://hawt.io/configuration/index.html)
- [使用 Dropwizard Metrics 对 Vert.x 性能指标进行监控](http://www.w2bc.com/article/228854)

### 健康检查「Health Checks」
- [Vert.x Health Check](http://vertx.io/docs/vertx-health-check/java/)

### 数据打点统计「Hawkular && Cassandra && Grafana」
- [Vert.x Hawkular Metrics](http://vertx.io/docs/vertx-hawkular-metrics/java/)
- [Getting started with Hawkular and Grafana Part 1](http://www.hawkular.org/hawkular-services/docs/quickstart-guide/)
- [Getting started with Hawkular and Grafana Part 2](http://www.hawkular.org/hawkular-services/docs/installation-guide/)
- [Getting started with Hawkular and Grafana Part 3](http://www.hawkular.org/hawkular-clients/grafana/docs/quickstart-guide/)
- [Github: hawkular-metrcis](https://github.com/hawkular/hawkular-metrics)
- [Github: Cassandra](https://github.com/apache/cassandra)
- [Github: Grafana](https://github.com/grafana/grafana)
