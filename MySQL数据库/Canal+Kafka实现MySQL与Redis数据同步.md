# 思维导图

![](https://static.lovebilibili.com/canal_jinjie_37.png)

# 前言

在很多业务情况下，我们都会在系统中加入redis缓存做查询优化。

如果数据库数据发生更新，这时候就需要在业务代码中写一段同步更新redis的代码。

这种**数据同步的代码跟业务代码糅合在一起会不太优雅**，能不能把这些数据同步的代码抽出来形成一个独立的模块呢，答案是可以的。

# 架构图

canal是一个伪装成slave订阅mysql的binlog，实现数据同步的中间件。上一篇文章[《canal入门》](https://juejin.im/post/6859278867635388430)

我已经介绍了最简单的使用方法，也就是tcp模式。

实际上canal是支持直接发送到MQ的，**目前最新版是支持主流的三种MQ：Kafka、RocketMQ、RabbitMQ**。而canal的RabbitMQ模式目前是有一定的bug，所以一般使用Kafka或者RocketMQ。

![](https://static.lovebilibili.com/canal_jinjie_25.png)

本文使用Kafka，实现Redis与MySQL的数据同步。架构图如下：

![](https://static.lovebilibili.com/canal_jinjie_26.png)

通过架构图，我们很清晰就知道要用到的组件：MySQL、Canal、Kafka、ZooKeeper、Redis。

下面演示Kafka的搭建，MySQL搭建大家应该都会，ZooKeeper、Redis这些网上也有很多资料参考。

# 搭建Kafka

首先在[官网](https://github.com/apache/kafka)下载安装包：

![](https://static.lovebilibili.com/canal_jinjie_21.png)

解压，打开/config/server.properties配置文件，修改日志目录：

```properties
log.dirs=./logs
```

首先启动ZooKeeper，我用的是3.6.1版本：

![](https://static.lovebilibili.com/canal_jinjie_27.png)

接着再启动Kafka，在Kafka的bin目录下打开cmd，输入命令：

```java
kafka-server-start.bat ../../config/server.properties
```

我们可以看到ZooKeeper上注册了Kafka相关的配置信息：

![](https://static.lovebilibili.com/canal_jinjie_28.png)

然后需要创建一个队列，用于接收canal传送过来的数据，使用命令：

```java
kafka-topics.bat --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic canaltopic
```

创建的队列名是`canaltopic`。

![](https://static.lovebilibili.com/canal_jinjie_29.png)

# 配置Cannal Server

canal[官网](https://github.com/alibaba/canal/releases)下载相关安装包：

![](https://static.lovebilibili.com/pic/canal_download.png)

找到canal.deployer-1.1.4/conf目录下的canal.properties配置文件：

```properties
# tcp, kafka, RocketMQ 这里选择kafka模式
canal.serverMode = kafka
# 解析器的线程数，打开此配置，不打开则会出现阻塞或者不进行解析的情况
canal.instance.parser.parallelThreadSize = 16
# 配置MQ的服务地址，这里配置的是kafka对应的地址和端口
canal.mq.servers = 127.0.0.1:9092
# 配置instance，在conf目录下要有example同名的目录，可以配置多个
canal.destinations = example
```

然后配置instance，找到/conf/example/instance.properties配置文件：

```properties
## mysql serverId , v1.0.26+ will autoGen(自动生成，不需配置)
# canal.instance.mysql.slaveId=0

# position info
canal.instance.master.address=127.0.0.1:3306
# 在Mysql执行 SHOW MASTER STATUS;查看当前数据库的binlog
canal.instance.master.journal.name=mysql-bin.000006
canal.instance.master.position=4596
# 账号密码
canal.instance.dbUsername=canal
canal.instance.dbPassword=Canal@****
canal.instance.connectionCharset = UTF-8
#MQ队列名称
canal.mq.topic=canaltopic
#单队列模式的分区下标
canal.mq.partition=0
```

配置完成后，就可以启动canal了。

# 测试

这时可以打开kafka的消费者窗口，测试一下kafka是否收到消息。

使用命令进行监听消费：

```java
kafka-console-consumer.bat --bootstrap-server 127.0.0.1:9092 --from-beginning --topic canaltopic
```

**有个小坑**。我这里使用的是win10系统的cmd命令行，win10系统默认的编码是GBK，而Canal Server是UTF-8的编码，所以控制台会出现乱码：

![](https://static.lovebilibili.com/canal_jinjie_30.png)

怎么解决呢？

在cmd命令行执行前切换到UTF-8编码即可，使用命令行：chcp 65001

然后再执行打开kafka消费端的命令，就不乱码了：

![](https://static.lovebilibili.com/canal_jinjie_31.png)

接下来就是启动Redis，把数据同步到Redis就完事了。

# 封装Redis客户端

环境搭建完成后，我们可以写代码了。

首先引入Kafka和Redis的maven依赖：

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

在application.yml文件增加以下配置：

```yaml
spring:  
  redis:
    host: 127.0.0.1
    port: 6379
    database: 0
    password: 123456
```

封装一个操作Redis的工具类：

```java
@Component
public class RedisClient {

    /**
     * 获取redis模版
     */
    @Resource
    private StringRedisTemplate stringRedisTemplate;

    /**
     * 设置redis的key-value
     */
    public void setString(String key, String value) {
        setString(key, value, null);
    }

    /**
     * 设置redis的key-value，带过期时间
     */
    public void setString(String key, String value, Long timeOut) {
        stringRedisTemplate.opsForValue().set(key, value);
        if (timeOut != null) {
            stringRedisTemplate.expire(key, timeOut, TimeUnit.SECONDS);
        }
    }

    /**
     * 获取redis中key对应的值
     */
    public String getString(String key) {
        return stringRedisTemplate.opsForValue().get(key);
    }

    /**
     * 删除redis中key对应的值
     */
    public Boolean deleteKey(String key) {
        return stringRedisTemplate.delete(key) == null;
    }
}
```

# 创建MQ消费者进行同步

在application.yml配置文件加上kafka的配置信息：

```yaml
spring:
  kafka:
  	# Kafka服务地址
    bootstrap-servers: 127.0.0.1:9092
    consumer:
      # 指定一个默认的组名
      group-id: consumer-group1
      #序列化反序列化
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringDeserializer
      value-serializer: org.apache.kafka.common.serialization.StringDeserializer
      # 批量抓取
      batch-size: 65536
      # 缓存容量
      buffer-memory: 524288
```

根据上面Kafka消费命令那里，我们知道了json数据的结构，可以创建一个CanalBean对象进行接收：

```java
public class CanalBean {
    //数据
    private List<TbCommodityInfo> data;
    //数据库名称
    private String database;
    private long es;
    //递增，从1开始
    private int id;
    //是否是DDL语句
    private boolean isDdl;
    //表结构的字段类型
    private MysqlType mysqlType;
    //UPDATE语句，旧数据
    private String old;
    //主键名称
    private List<String> pkNames;
    //sql语句
    private String sql;
    private SqlType sqlType;
    //表名
    private String table;
    private long ts;
    //(新增)INSERT、(更新)UPDATE、(删除)DELETE、(删除表)ERASE等等
    private String type;
    //getter、setter方法
}
```

```java
public class MysqlType {
    private String id;
    private String commodity_name;
    private String commodity_price;
    private String number;
    private String description;
    //getter、setter方法
}
```

```java
public class SqlType {
    private int id;
    private int commodity_name;
    private int commodity_price;
    private int number;
    private int description;
}
```

最后就可以创建一个消费者CanalConsumer进行消费：

```java
@Component
public class CanalConsumer {
	//日志记录
    private static Logger log = LoggerFactory.getLogger(CanalConsumer.class);
	//redis操作工具类
    @Resource
    private RedisClient redisClient;
	//监听的队列名称为：canaltopic
    @KafkaListener(topics = "canaltopic")
    public void receive(ConsumerRecord<?, ?> consumer) {
        String value = (String) consumer.value();
        log.info("topic名称:{},key:{},分区位置:{},下标:{},value:{}", consumer.topic(), consumer.key(),consumer.partition(), consumer.offset(), value);
        //转换为javaBean
        CanalBean canalBean = JSONObject.parseObject(value, CanalBean.class);
        //获取是否是DDL语句
        boolean isDdl = canalBean.getIsDdl();
        //获取类型
        String type = canalBean.getType();
        //不是DDL语句
        if (!isDdl) {
            List<TbCommodityInfo> tbCommodityInfos = canalBean.getData();
            //过期时间
            long TIME_OUT = 600L;
            if ("INSERT".equals(type)) {
                //新增语句
                for (TbCommodityInfo tbCommodityInfo : tbCommodityInfos) {
                    String id = tbCommodityInfo.getId();
                    //新增到redis中,过期时间是10分钟
                    redisClient.setString(id, JSONObject.toJSONString(tbCommodityInfo), TIME_OUT);
                }
            } else if ("UPDATE".equals(type)) {
                //更新语句
                for (TbCommodityInfo tbCommodityInfo : tbCommodityInfos) {
                    String id = tbCommodityInfo.getId();
                    //更新到redis中,过期时间是10分钟
                    redisClient.setString(id, JSONObject.toJSONString(tbCommodityInfo), TIME_OUT);
                }
            } else {
                //删除语句
                for (TbCommodityInfo tbCommodityInfo : tbCommodityInfos) {
                    String id = tbCommodityInfo.getId();
                    //从redis中删除
                    redisClient.deleteKey(id);
                }
            }
        }
    }
}
```

# 测试MySQL与Redis同步

mysql对应的表结构如下：

```sql
CREATE TABLE `tb_commodity_info` (
  `id` varchar(32) NOT NULL,
  `commodity_name` varchar(512) DEFAULT NULL COMMENT '商品名称',
  `commodity_price` varchar(36) DEFAULT '0' COMMENT '商品价格',
  `number` int(10) DEFAULT '0' COMMENT '商品数量',
  `description` varchar(2048) DEFAULT '' COMMENT '商品描述',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品信息表';
```

首先在MySQL创建表。然后启动项目，接着新增一条数据：

```sql
INSERT INTO `canaldb`.`tb_commodity_info` (`id`, `commodity_name`, `commodity_price`, `number`, `description`) VALUES ('3e71a81fd80711eaaed600163e046cc3', '叉烧包', '3.99', '3', '又大又香的叉烧包，老人小孩都喜欢');
```

tb_commodity_info表查到新增的数据：

![](https://static.lovebilibili.com/canal_jinjie_32.png)

Redis也查到了对应的数据，证明同步成功！

![](https://static.lovebilibili.com/canal_jinjie_33.png)

如果更新呢？试一下Update语句：

```sql
UPDATE `canaldb`.`tb_commodity_info` SET `commodity_name`='青菜包',`description`='很便宜的青菜包呀，不买也开看看了喂' WHERE `id`='3e71a81fd80711eaaed600163e046cc3';
```

![](https://static.lovebilibili.com/canal_jinjie_34.png)

![](https://static.lovebilibili.com/canal_jinjie_35.png)

没有问题！

# 总结

那么你会说，canal就没有什么缺点吗？

肯定是有的：

1. canal只能同步增量数据。
2. 不是实时同步，是准实时同步。
3. 存在一些bug，不过社区活跃度较高，对于提出的bug能及时修复。
4. MQ顺序性问题。我这里把官网的回答列出来，大家参考一下。

![](https://static.lovebilibili.com/canal_jinjie_36.png)

尽管有一些缺点，毕竟没有一样技术或者产品是完美的，最重要是合适。

我们公司在同步MySQL数据到Elastic Search也是采用Canal+RocketMQ的方式。

参考资料：[canal官网](https://github.com/alibaba/canal/)

## 絮叨

上面所有例子的代码都上传Github了：

> https://github.com/yehongzhi/mall

**如果你觉得这篇文章对你有用，点个赞吧**~

**你的点赞是我创作的最大动力**~

想第一时间看到我更新的文章，可以微信搜索公众号「`java技术爱好者`」，**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**