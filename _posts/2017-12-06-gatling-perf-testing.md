---
title: "服务端性能压测利器 Gatling"
layout: post
date: 2017-12-06 10:15
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- Gatling
- Scala
category: blog
author: jiangew
---

<!-- TOC -->

- [Quickstart](#quickstart)
- [Installing](#installing)
    - [Gatling 执行测试脚本](#gatling-执行测试脚本)
- [Test Case](#test-case)
    - [Gatling Scenario Explained](#gatling-scenario-explained)
    - [Running Gatling](#running-gatling)
    - [Execute Case](#execute-case)
    - [Scala crashed issue](#scala-crashed-issue)
    - [Config connection timed out](#config-connection-timed-out)
    - [Failed to open socket issue](#failed-to-open-socket-issue)
    - [Cannot assign requested address issue](#cannot-assign-requested-address-issue)
- [Advanced Tutorial](#advanced-tutorial)
    - [Isolate processes](#isolate-processes)
    - [Configure virtual users](#configure-virtual-users)
    - [Loop statements](#loop-statements)
    - [Check and failure management](#check-and-failure-management)
- [General](#general)
    - [Concepts](#concepts)
    - [Operations](#operations)
    - [Configuration](#configuration)
    - [Simulation setup](#simulation-setup)
    - [Reports](#reports)
        - [Use log file to generate report](#use-log-file-to-generate-report)
- [Reference](#reference)

<!-- /TOC -->

Back to [Performance Test](http://wiki.li3huo.com/Performance_Test)
See Also
[Scala](http://wiki.li3huo.com/Scala)、
[JMeter](http://wiki.li3huo.com/JMeter)、
[ApacheBench](http://wiki.li3huo.com/ApacheBench)、
[wrk](http://wiki.li3huo.com/wrk)

## Quickstart
[Offical Quickstart](http://gatling.io/docs/current/quickstart/)

## Installing
```sh
➜  gatling wget https://repo1.maven.org/maven2/io/gatling/highcharts/gatling-charts-highcharts-bundle/2.3.0/gatling-charts-highcharts-bundle-2.3.0-bundle.zip
➜  gatling-charts-highcharts-bundle-2.3.0 bin/gatling.sh -h
➜  gatling-charts-highcharts-bundle-2.3.0 tree . -d
├── bin                                         //执行脚本
│   ├── gatling.sh                                  //运行测试
│   └── recorder.sh                                 //启动录制脚本的UI
├── conf                                        //应用自身的配置
│   ├── gatling-akka.conf                       
│   ├── gatling.conf                                
│   ├── logback.xml                                 
│   └── recorder.conf                               
├── lib                                        //应用自身依赖库
│   └── zinc                                            
├── results                                    //存放测试报告
├── target                                              
│   └── test-classes                                    
└── user-files                                 //存放测试脚本
    ├── bodies                                          
    ├── data                                                
    └── simulations                                     
        └── computerdatabase                                
            ├── BasicSimulation.scala
            └── advanced
```
### Gatling 执行测试脚本
```sh
    gatling-charts-highcharts-bundle-2.3.0 st user-files/simulations/computerdatabase/BasicSimulation.scala
```

## Test Case
[Quickstart Test Case](http://gatling.io/docs/current/quickstart/#test-case)

Test Case中的关键概念，详见[#Concepts](http://wiki.li3huo.com/Gatling#Concepts)
* simulations
* scenarios A scenario is a chain of requests and pauses
* feeders
* recorder
* loops

### Gatling Scenario Explained
[Gatling Scenario](http://gatling.io/docs/current/quickstart/#gatling-scenario-explained)

* 首先定义一个URL： val httpConf
* 然后定义Case中的场景： val scn = scenario("Scenario Name")
* 最后启动模拟器： setUp(scn.inject(atOnceUsers(1)).protocols(httpConf))

### Running Gatling
[Quickstart Running](http://gatling.io/docs/current/quickstart/#running-gatling)

### Execute Case
Launch $GATLING_HOME/bin/gatling.sh
```sh
➜  gatling-charts-highcharts-bundle-2.3.0 bin/gatling.sh 
GATLING_HOME is set to /Users/jiangew/gatling/gatling-charts-highcharts-bundle-2.3.0
18:43:51.565 [WARN ] i.g.c.ZincCompiler$ - Pruning sources from previous analysis, due to incompatible CompileSetup.
Choose a simulation number:
     [0] computerdatabase.BasicSimulation
     [1] computerdatabase.advanced.AdvancedSimulationStep01
     [2] computerdatabase.advanced.AdvancedSimulationStep02
     [3] computerdatabase.advanced.AdvancedSimulationStep03
     [4] computerdatabase.advanced.AdvancedSimulationStep04
     [5] computerdatabase.advanced.AdvancedSimulationStep05
0
Select simulation id (default is 'basicsimulation'). Accepted characters are a-z, A-Z, 0-9, - and _

Select run description (optional)

Simulation computerdatabase.BasicSimulation started...

Reports generated in 0s.
Please open the following file: results/basicsimulation-1505126722282/index.html
```

### Scala crashed issue
[Scala build crashed](https://stackoverflow.com/questions/40628310/scala-build-crashed)

Scala 2.12 require's newer version of JDK then 1.8.0_111

### Config connection timed out
```sh
    export JAVA_OPTS="-Dgatling.http.connectionTimeout=5000"
```

### Failed to open socket issue
gatling j.n.ConnectException: Failed to open a socket.

* update limits.conf on Linux [Kernel_Parameters#limits.conf](http://wiki.li3huo.com/Kernel_Parameters#limits.conf)
* action on MacOS [Kernel_Parameters#Change_ulimit_settings](http://wiki.li3huo.com/Kernel_Parameters#Change_ulimit_settings)

测试机系统调优参考 [#Operations](http://wiki.li3huo.com/Gatling#Operations)

### Cannot assign requested address issue

解决 Gatling 端口占用问题
```sh
sudo sysctl -w net.ipv4.ip_local_port_range="1025 65535"
sudo sysctl -w net.ipv4.tcp_tw_recycle=1
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
sudo sysctl -w net.ipv4.tcp_timestamps=1
sudo sysctl -w net.ipv4.tcp_max_tw_buckets=50000
```

查看端口状态
```sh
$ ss -ant | awk '{++s[$1]} END {for(k in s) print k,s[k]}'
ESTAB 44853
State 1
FIN-WAIT-1 1
FIN-WAIT-2 2
TIME-WAIT 1338
LISTEN 5
```

## Advanced Tutorial
[Offical Advanced Tutorial](http://gatling.io/docs/current/advanced_tutorial/)

### Isolate processes
可以把Case定义为object， 在exec的时候传入多个
```scala
    val scn = scenario("Scenario Name").exec(Search.search, Browse.browse, Edit.edit)
```

### Configure virtual users
```scala
setUp(users.inject(atOnceUsers(10)).protocols(httpConf))

setUp(
  users.inject(rampUsers(10) over (10 seconds)),
  admins.inject(rampUsers(2) over (10 seconds))
).protocols(httpConf)
```

### Loop statements
[Scenario Loops](http://gatling.io/docs/current/general/scenario/#scenario-loops)

### Check and failure management
[Http Check](http://gatling.io/docs/current/http/http_check/#http-check)

```scala
class BenchmarkPost extends Simulation {
  val httpConf = http
    .baseURL("http://10.62.21.164:28080")
    .acceptHeader("text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8")
    .doNotTrackHeader("1")
    .acceptLanguageHeader("en-US,en;q=0.5")
    .acceptEncodingHeader("gzip, deflate")

  val headers_common = Map("Content-Type" -> "application/json;charset=utf-8")

  val scn = scenario("Elasticsearch Proxy Post")
    .exec(
      http("request_post_1")
        .post("/put")
        .headers(headers_common)
        .queryParam("index", "notes")
        .queryParam("type", "doc")
        .queryParam("id", "0")
        .body(RawFileBody("post.json")).asJSON
        .check(
            status.not(500),
            status.not(502),
            status.not(504),
            jsonPath("$.code").ofType[Int].is(440).saveAs("code"),
            jsonPath("$.msg").ofType[String].is("session timeout")
        )
    )

  setUp(scn.inject(constantUsersPerSec(5000) during (180 seconds)).protocols(httpConf))
}
```

## General
[Offical General](http://gatling.io/docs/current/general/)

### Concepts
[Concepts](http://gatling.io/docs/current/general/concepts/)

* Virtual User Gatling 用消息来实现，比用 Thread 效率上要高很多
* Scenario 某个用户的具体行为
* Simulation 一次压力测试，可以同时包含多个用户的不同行为
* Session
* Feeders
* Checks
* Assertions
* Reports

### Operations
[OS Tuning: Open Files Limit & Kernel and Network Tuning](http://gatling.io/docs/current/general/operations/)

包含一些测试机需要调整的系统参数，例如: 
```shell
ulimit -n
ulimit -n 65535
```

### Configuration
[配置文件、命令行参数](http://gatling.io/docs/current/general/configuration/)

### Simulation Setup
[Simulation Setup](http://gatling.io/docs/current/general/simulation_setup/)
[Simulation Setup 中文翻译](https://testerhome.com/topics/4094)

Injection 定义虚拟用户的操作：
* atOnceUsers(nbUsers) ：立即注入一定数量的虚拟用户
* rampUsers(nbUsers) over(duration) ：将一定数量的虚拟用户，在指定时间内逐步注入进来
* constantUsersPerSec(rate) during(duration) ：定义一个在每秒钟恒定的并发用户数，持续指定的时间；
* constantUsersPerSec(rate) during(duration) randomized ：定义一个在每秒钟围绕指定并发数随机增减的并发，持续指定时间
* rampUsersPerSec(rate1) to (rate2) during(duration) ：定义一个并发数区间，运行指定时间，并发增长的周期是一个规律的值
* rampUsersPerSec(rate1) to(rate2) during(duration) randomized ：定义一个并发数区间，运行指定时间，并发增长的周期是一个随机的值；
* splitUsers(nbUsers) into(injectionStep) separatedBy(duration) ：定义一个周期，执行injectionStep里面的注入，将nbUsers的请求平均分配
* splitUsers(nbUsers) into(injectionStep1) separatedBy(injectionStep2) ：使用injectionStep2的注入作为周期，分隔injectionStep1的注入，直到用户数达到nbUsers
* heavisideUsers(nbUsers) over(duration) ：定义一个持续的并发，围绕和海维赛德函数平滑逼近的增长量，持续指定时间（译者解释海维赛德函数` H(x)当x>0时返回1，x< * 0时返回0，x=0时返回0.5。实际操作时，并发数是一个成平滑抛物线形的曲线）

### Reports
[怎么看测试报告](http://gatling.io/docs/current/general/reports/)

#### Use log file to generate report
Generates the reports for the simulation log file located in *<gatling_home>/results/<folderName>*

```sh
➜  gatling-charts-highcharts-bundle-2.3.0 ll results/test
total 648
-rw-r--r--  1 jiangew  staff   322K Oct 17 18:29 simulation.log

➜  gatling-charts-highcharts-bundle-2.3.0 bin/gatling.sh -ro test
...
Generating reports...
...
Reports generated in 0s.
Please open the following file: /gatling/gatling-charts-highcharts-bundle-2.3.0/results/test/index.html
```

## Reference
* [Gatling](http://gatling.io/)
* [Gatling Web 压力测试](https://segmentfault.com/a/1190000008254640)
* [Gatling Quickstart 问题汇总](http://wiki.li3huo.com/Gatling)
