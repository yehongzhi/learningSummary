> 文章已收录到我的[Github](https://github.com/yehongzhi/learningSummary)精选，欢迎Star！

# 简单介绍

Pulsar 是一个用于服务器到服务器的消息系统，具有多租户、高性能等优势。最初是由 Yahoo 开发，目前由 Apache 软件基金会 管理。是 Apache 软件基金会顶级项目，是下一代云原生分布式消息流平台，集消息、存储、轻量化函数式计算为一体，采用计算与存储分离架构设计，支持多租户、持久化存储、多机房跨区域数据复制，具有强一致性、高吞吐、低延时及高可扩展性等流数据存储特性，被看作是云原生时代实时消息流传输、存储和计算最佳解决方案。

# 特性

- Pulsar 的单个实例原生支持多个集群，可跨机房在集群间无缝地完成消息复制。

- 极低的发布和端到端延迟。

- 可无缝扩展到超过一百万个 topic。

- 简单易用的客户端API，支持Java、Go、Python和C++。

- 支持多种 topic 订阅模式（独占订阅、共享订阅、故障转移订阅）。

- 通过 Apache BookKeeper 提供的持久化消息存储机制保证消息传递。

- 基于 Pulsar Functions 的 serverless connector 框架 Pulsar IO 使得数据更易移入、移出 Apache Pulsar。

- 由轻量级的 serverless 计算框架 Pulsar Functions 实现流原生的数据处理。

- 分层式存储可在数据陈旧时，将数据从热存储卸载到冷/长期存储（如S3、GCS）中。

# 架构

![](https://static.lovebilibili.com/pulsar_01.png)

这是在官网的架构图，涉及到几个组件，这里简单说明一下：

Broker：Broker负责消息的传输，Topic的管理以及负载均衡，Broker**不负责消息的存储**，是个**无状态**组件。

Bookie：负责**消息的的持久化**，采用Apache BookKeeper组件，BookKeeper是一个分布式的WAL系统。

Producer：生产者，封装消息并将消息以同步或者异步的方式发送到Broker。

Consumer：消费者，以订阅Topic的方式消费消息，并确认。Pulsar中还定义了Reader角色，也是一种消费者，区别在于，它可以从指定置位获取消息，且不需要确认。

Zookeeper：元数据存储，负责集群的配置管理，包括租户，命名空间等，并进行一致性协调。

# 四种订阅模式

在介绍Pulsar特性时，讲过支持多种订阅模式，总共有四种，分别是独占(exclusive)订阅、共享(shared)订阅、故障转移(failover)订阅、键(key_shared)共享。

![](https://static.lovebilibili.com/pulsar_02.png)

## 独占（Exclusive）

独占模式：同时只有一个消费者可以启动并消费数据；通过 `SubscriptionName` 标明是同一个消费者），适用范围较小。

![](https://static.lovebilibili.com/pulsar_03.png)

## 共享（Shared）

可以有 N 个消费者同时运行，消息按照 `round-robin` 轮询投递到每个 `consumer` 中；当某个 `consumer` 宕机没有 `ack` 时，该消息将会被投递给其他消费者。这种消费模式可以提高消费能力，但消息无法做到有序。

![](https://static.lovebilibili.com/pulsar_05.png)

## 故障转移（Failover）

故障转移模式：在独占模式基础之上可以同时启动多个 `consumer`，一旦一个 `consumer` 挂掉之后其余的可以快速顶上，但也只有一个 `consumer` 可以消费；部分场景可用。

![](https://static.lovebilibili.com/pulsar_04.png)

## 键共享（KeyShared）

基于共享模式；相当于对同一个`topic`中的消息进行分组，同一分组内的消息只能被同一个消费者有序消费。

![](https://static.lovebilibili.com/pulsar_06.png)

# 下载安装

我这里安装的是2.9.1版本的pulsar，链接地址如下：

> https://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=pulsar/pulsar-2.9.1/apache-pulsar-2.9.1-bin.tar.gz

下载完成之后，上传到Linux服务器，然后使用命令解压：

> tar -zxvf apache-pulsar-2.9.0-bin.tar.gz

单机版的话，使用命令后台启动：

> ./bin/pulsar-daemon start standalone

终止后台运行的命令：

> ./bin/pulsar-daemon stop standalone

# SpringBoot整合

在Linux服务器上启动完成之后，就到了使用Java客户端进行操作的步骤，首先引入Maven依赖：

```xml
<dependency>
    <groupId>io.github.majusko</groupId>
    <artifactId>pulsar-java-spring-boot-starter</artifactId>
    <version>1.1.0</version>
</dependency>
```

application.yml配置文件加上配置：

```yml
#pulsar的服务地址
pulsar:
  service-url: pulsar://192.168.0.105:6650
```

增加个配置类`PulsarConfig`：

```java
@Configuration
public class PulsarConfig {

    @Bean
    public ProducerFactory producerFactory() {
        return new ProducerFactory().addProducer("testTopic", String.class);
    }
}
```

增加个记录Topic名称的常量类：

```java
/**
 * Pulsar中间件的topic名称
 *
 * @author yehongzhi
 * @date 2022/4/9 5:57 PM
 */
public class TopicName {

    private TopicName(){}
    /**
     * 测试用的topic
     */
    public static final String TEST_TOPIC = "testTopic";
}
```

增加消息生产者`PulsarProducer`类：

```java
/**
 * Pulsar生产者
 *
 * @author yehongzhi
 * @date 2022/4/9 5:23 PM
 */
@Component
public class PulsarProducer<T> {

    @Resource
    private PulsarTemplate<T> template;

    /**
     * 发送消息
     */
    public void send(String topic, T message) {
        try {
            template.send(topic, message);
        } catch (PulsarClientException e) {
            e.printStackTrace();
        }
    }
}
```

增加Topic名称为"testTopic"的消费者：

```java
/**
 * topic名称为"testTopic"对应的消费者
 *
 * @author yehongzhi
 * @date 2022/4/9 6:00 PM
 */
@Component
public class TestTopicPulsarConsumer {

    private static final Logger log = LoggerFactory.getLogger(TestTopicPulsarConsumer.class);

    //SubscriptionType.Shared，表示共享模式
    @PulsarConsumer(topic = TopicName.TEST_TOPIC,
                    subscriptionType = SubscriptionType.Shared,
                    clazz = String.class)
    public void consume(String message) {
        log.info("PulsarRealConsumer content:{}", message);
    }

}
```

最后增加一个`PulsarController`测试发送消息：

```java
@RestController
@RequestMapping("/pulsar")
public class PulsarController {

    @Resource
    private PulsarProducer<String> pulsarProducer;

    @PostMapping(value = "/sendMessage")
    public CommonResponse<String> sendMessage(@RequestParam(name = "message") String message) {
        pulsarProducer.send(TopicName.TEST_TOPIC, message);
        return CommonResponse.success("done");
    }
}
```

`CommonResponse`公共响应体：

```java
public class CommonResponse<T> {

    private String code;

    private Boolean success;

    private T data;

    public static <T>CommonResponse<T> success(T t){
        return new CommonResponse<>("200",true,t);
    }

    public CommonResponse(String code, Boolean success, T data) {
        this.code = code;
        this.success = success;
        this.data = data;
    }
    //getter、setter方法
}
```

启动项目，然后使用postman测试：
![](https://static.lovebilibili.com/pulsar_07.png)

![](https://static.lovebilibili.com/pulsar_08.png)

# 总结

以上就是pulsar中间件的简单入门，分别介绍了pulsar的特性，架构，订阅模式，还有个整合SpringBoot的小例子。最后跟其他主流的中间件做个对比，供大家参考一下：

![](https://static.lovebilibili.com/pulsar_09.jpeg)

感谢大家的阅读，希望这篇文章对你有所帮助。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！
