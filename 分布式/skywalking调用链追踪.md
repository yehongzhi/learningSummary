# 思维导图

![](https://static.lovebilibili.com/skywalking_swdt.png)

> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 概述

**[skywalking](https://github.com/apache/skywalking)**又是一个优秀的国产开源框架，2015年由个人吴晟（华为开发者）开源 ， 2017年加入Apache孵化器。

skywalking是分布式系统的**应用程序性能监视工具**，专为微服务、云原生架构和基于容器（Docker、K8s、Mesos）架构而设计。SkyWalking 是**观察性分析平台和应用性能管理系统**。提供**分布式追踪、服务网格遥测分析、度量聚合和可视化一体化**解决方案（官网介绍）。

# 一、OpenTracing规范

**OpenTracing是一种分布式系统链路跟踪的设计原则、规范、标准。**

类似JDBC的规范，主要为了提供一套标准的JDBC API。OpenTracing也是一样，是为了统一提供一套链路追踪的标准API，所制定的一种规范。

OpenTracing通过提供平台无关、厂商无关的API，使得开发人员能够方便的添加（或更换）追踪系统的实现。

类似于JDBC的规范由各个数据库厂商实现一样，OpenTracing规范也是有很多实现的产品，下面介绍一下落地的产品。

## 1.1 实现OpenTracing的产品

**Jaeger**：Jaeger是由Uber公司开源发布的，受到Dapper和OpenZipkin启发。后端使用Go语言，前端(用户界面)使用React 。优点是上传采用的是udp传输，效率高速度快。缺点就是丢包，影响了整条调用链，而且不支持告警和JVM监控。

**Zipkin**：SpringCloud官方推荐，可以**与SpringCloud有良好集成**，实现方式是拦截请求，发送(http)数据到zipkin服务。缺点在于**不支持告警，不支持JVM监控，通信方式使用Http请求向Zipkin上报信息，比较耗性能**。

**SkyWalking**：国人(吴晟)开发，支持dubbo，SpringCloud，SpringBoot集成，**代码无侵入，通信方式采用GRPC，性能较好，实现方式是java探针，支持告警，支持JVM监控，支持全局调用统计**等等，功能较完善。缺点是**依赖较多**，需要ElasticSearch，JDK环境，Nacos注册中心等。

## 1.2 skywalking的特点

![](https://static.lovebilibili.com/skywalking_1.png)

比较重要的特点，我觉得是轻量高效，对代码无侵入性。对于微服务，支持dubbo，SpringBoot，SpringCloud集成。

# 二、安装部署

环境：CentOS 7.5，MySQL 5.7.26，Nacos 1.3.1（注册中心），JDK 1.8，**skywalking 8.1.0**。

除了skywalking之外，其他需要用到的组件我就不介绍怎么安装了，比较简单。安装skywalking其实很简单，下面一步一步来讲解。

第一步，下载。在[官网](http://skywalking.apache.org/downloads/)下载即可，选择8.1.0版本，如果要使用ES作为存储仓库，那就要选择es7的版本。

![](https://static.lovebilibili.com/skywalking_2.png)



第二步，解压。找到config目录下的application.yml文件，然后修改配置。

![](https://static.lovebilibili.com/skywalking_3.png)

需要修改的配置内容如下：

```yaml
cluster:
  selector: ${SW_CLUSTER:nacos}
  #单机模式
  standalone:
  #使用nacos作为注册中心
  nacos:
    # 注册到nacos的服务名
    serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
    #nacos服务端的地址
    hostPort: ${SW_CLUSTER_NACOS_HOST_PORT:192.168.0.105:8848}
    # Nacos Configuration namespace命名空间
    namespace: ${SW_CLUSTER_NACOS_NAMESPACE:"public"}
core:
  selector: ${SW_CORE:default}
  default:
    #skywalking服务端的REST绑定的IP
    restHost: ${SW_CORE_REST_HOST:192.168.0.107}
    #skywalking服务端的REST调用的端口
    restPort: ${SW_CORE_REST_PORT:12800}
    #skywalking服务端GRPC通信绑定的IP
    gRPCHost: ${SW_CORE_GRPC_HOST:192.168.0.107}
    #skywalking服务端GRPC通信绑定的端口
    gRPCPort: ${SW_CORE_GRPC_PORT:11800}
storage:
  #选择使用mysql
  selector: ${SW_STORAGE:mysql}
  #默认使用h2，不会持久化，重启skyWalking之前的数据会丢失
  h2:
    driver: ${SW_STORAGE_H2_DRIVER:org.h2.jdbcx.JdbcDataSource}
    url: ${SW_STORAGE_H2_URL:jdbc:h2:mem:skywalking-oap-db}
    user: ${SW_STORAGE_H2_USER:sa}
    metadataQueryMaxSize: ${SW_STORAGE_H2_QUERY_MAX_SIZE:5000}
  #使用mysql作为持久化存储的仓库
  mysql:
    properties:
      #数据库连接地址
      jdbcUrl: ${SW_JDBC_URL:"jdbc:mysql://192.168.0.107:3306/swtest"}
      #用户名
      dataSource.user: ${SW_DATA_SOURCE_USER:yehongzhi}
      #密码
      dataSource.password: ${SW_DATA_SOURCE_PASSWORD:Yehongzhi520.}
```

默认是web管理界面是8080端口，如果要修改端口号，可以修改webapp目录下的webapp.yml。

```yaml
#web管理界面的端口
server:
  port: 8080
```

第三步，添加mysql数据驱动包。因为在lib目录下是没有mysql数据驱动包的，所以修改完配置启动是会报错，启动失败的。为什么作者不提前在lib目录下放一个数据驱动包呢，还要我们手动去添加。网上貌似没有这个问题的讨论，我的理解是**因为框架不知道你用的是什么版本的mysql数据库，所以不知道放什么版本的数据库驱动包，使用者用的是什么版本的mysql，就自己放对应的数据库驱动包**。

我这里用的是5.7.26版本的mysql，所以我下载了一个8.0.17的驱动包，添加到/oap-libs目录下。

![](https://static.lovebilibili.com/skywalking_4.png)

第三步，启动。在/bin目录上一级，直接使用`./bin/startup.sh`启动即可。启动之后，可以使用`jps`命令查看进程，可以看到这两个java程序在运行状态。

![](https://static.lovebilibili.com/skywalking_5.png)

打开配置的Nacos控制台，可以看到服务列表注册了名为“SkyWalking_OAP_Cluster”的服务。

![](https://static.lovebilibili.com/skywalking_6.png)

可以看到mysql建了很多表。

![](https://static.lovebilibili.com/skywalking_7.png)

说明启动成功了，打开配置对应的地址http://192.168.0.109:8080/，可以看到skywalking的web界面。

![](https://static.lovebilibili.com/skywalking_8.png)

# 三、整合SpringCloud工程

整合其实很简单，不需要引入依赖，也不需要添加任何代码，我们只需要在启动jar包时配置参数即可。

```
-javaagent:D:\apache-skywalking-apm-bin-es7\agent\skywalking-agent.jar
-Dskywalking.agent.service_name=consumer
-Dskywalking.collector.backend_service=192.168.0.109:11800
#解释一下上面这三个参数的意思
#-javaagent:填的是skywalking-agent.jar的本地磁盘的路径
#-Dskywalking.agent.service_name：在skywalking上显示的服务名
#-Dskywalking.collector.backend_service：skywalking的collector服务的IP及端口
```

我们一般用IDEA开发就这样设置即可。

![](https://static.lovebilibili.com/skywalking_9.png)

接下来我按照这个配置，启动一个Consumer工程和Provider工程，并且注册到Nacos注册中心。

![](https://static.lovebilibili.com/skywalking_10.png)

然后使用Consumer工程的接口调用Provider工程的接口，可以看到调用链的效果。

![](https://static.lovebilibili.com/skywalking_11.png)

非常清晰地看到服务之间调用的情况，耗时等等。其他还有很多功能就不一一介绍了，读者可以自己探索一下。

# 四、谈谈架构设计

可能前面还有一些疑问，比如为什么要设置GRPC的端口号，设置存储仓库为mysql，启动之后有两个java进程等等。不妨看看架构，一切问题都明白了。首先看官网的一张架构图。

![](https://static.lovebilibili.com/skywalking_15.png)

可以看到主要有四个部分。

**上面的Agent** ：负责从应用中，收集tracing(调用链数据)和metric(指标)，发送给 SkyWalking OAP 服务器。目前支持 SkyWalking、Zikpin、Jaeger 等提供的 Tracing 数据信息。而我们目前采用的是，SkyWalking Agent 收集 SkyWalking Tracing 数据，传递给SkyWalking OAP 服务器。

**中间的SkyWalking OAP**：负责接收 Agent 发送的 Tracing 和Metric的数据信息，然后进行分析(Analysis Core) ，存储到外部存储器( Storage )，最终提供查询( Query )功能。

**左边的SkyWalking UI**：负责提供web控制台，查看链路，查看各种指标，性能等等。

**右边的Storage**：数据存储。目前支持ES、MySQL、H2等多种存储器。

# 总结

这篇文章就介绍到这里，这里仅仅只是入门，简单使用Skywalking，实际上里面还有很多功能我没有介绍，有兴趣的同学可以按照上面的教程安装部署，然后自己探索一下。

在现在微服务架构比较流行的环境下，如果没有一个调用链追踪框架，会导致很难排查线上服务调用的问题。skywalking是目前发展势头最快的技术框架的技术框架，因为对代码是无侵入性的，所以目前很多公司都采用Skywalking。

![](https://static.lovebilibili.com/skywalking_14.png)

这篇文章就讲到这里了，感谢大家的阅读。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！