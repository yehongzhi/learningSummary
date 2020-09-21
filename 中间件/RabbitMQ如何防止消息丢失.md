---
title: RabbitMQ如何防止消息丢失
date: 2020-08-08 14:47:49
index_img: https://static.lovebilibili.com/rabbitmq_jinjie_index.jpg
tags:
	- 消息队列
	- RabbitMQ
	- 中间件
---

# 思维导图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200802232920896.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3llaG9uZ3poaTE5OTQ=,size_16,color_FFFFFF,t_70)
# 一、分析数据丢失的原因
分析RabbitMQ消息丢失的情况，不妨先看看一条消息从生产者发送到消费者消费的过程：

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC83LzI4LzE3Mzk1Yzc2OTVkMDE5MjE?x-oss-process=image/format,png)

可以看出，一条消息整个过程要经历两次的网络传输：**从生产者发送到RabbitMQ服务器，从RabbitMQ服务器发送到消费者**。

<!--more-->

**在消费者未消费前存储在队列(Queue)中**。

所以可以知道，有三个场景下是会发生消息丢失的：

- 存储在队列中，如果队列没有对消息持久化，RabbitMQ服务器宕机重启会丢失数据。
- 生产者发送消息到RabbitMQ服务器过程中，RabbitMQ服务器如果宕机停止服务，消息会丢失。
- 消费者从RabbitMQ服务器获取队列中存储的数据消费，但是消费者程序出错或者宕机而没有正确消费，导致数据丢失。

针对以上三种场景，RabbitMQ提供了三种解决的方式，分别是消息持久化，confirm机制，ACK事务机制。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC83LzI4LzE3Mzk1ZDVhNjQ1YzNkNDc?x-oss-process=image/format,png)

# 二、消息持久化

RabbitMQ是支持消息持久化的，消息持久化需要设置：Exchange为持久化和Queue持久化，这样当消息发送到RabbitMQ服务器时，消息就会持久化。

首先看Exchange交换机的类图：

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC83LzI4LzE3Mzk1ZjY3MTUyNmVkMmQ?x-oss-process=image/format,png)

看这个类图其实是要说明上一篇文章介绍的四种交换机都是AbstractExchange抽象类的子类，所以根据java的特性，**创建子类的实例会先调用父类的构造器**，父类也就是AbstractExchange的构造器是怎么样的呢？

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC83LzI4LzE3Mzk1ZjhjMGQ1YzhlNTE?x-oss-process=image/format,png)

从上面的注释可以看到**durable参数表示是否持久化。默认是持久化(true)**。创建持久化的Exchange可以这样写：
```java
	@Bean
    public DirectExchange rabbitmqDemoDirectExchange() {
        //Direct交换机
        return new DirectExchange(RabbitMQConfig.RABBITMQ_DEMO_DIRECT_EXCHANGE, true, false);
    }
```

接着是Queue队列，我们先看看Queue的构造器是怎么样的：

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMS1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC9lNTA3ZmU0OTExNWE0MTk5YTFhNjg2ZDIwNWM5OWRlY350cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)

也是通过durable参数设置是否持久化，默认是true。所以创建时可以不指定：

```java
	@Bean
    public Queue fanoutExchangeQueueA() {
    	//只需要指定名称，默认是持久化的
        return new Queue(RabbitMQConfig.FANOUT_EXCHANGE_QUEUE_TOPIC_A);
    }
```

这就完成了消息持久化的设置，接下来启动项目，发送几条消息，我们可以看到：

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wNi1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC8zNWFmZjJkNmE0YzU0YjFjYmQ4ODk1NTM2NTExOTAwYn50cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
怎么证明是已经持久化了呢，实际上可以找到对应的文件：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200801205656332.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3llaG9uZ3poaTE5OTQ=,size_16,color_FFFFFF,t_70)
找到对应磁盘中的目录：
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMS1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC80NmU1NDIxNTYwM2U0MmIyODI4ODlhYTQxYTVhNjFiOH50cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
**消息持久化可以防止消息在RabbitMQ Server中不会因为宕机重启而丢失**。

# 三、消息确认机制

## 3.1 confirm机制
**在生产者发送到RabbitMQ Server时有可能因为网络问题导致投递失败，从而丢失数据**。我们可以使用confirm模式防止数据丢失。工作流程是怎么样的呢，看以下图解：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200801210244345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3llaG9uZ3poaTE5OTQ=,size_16,color_FFFFFF,t_70)
从上图中可以看到是通过两个回调函数**confirm()、returnedMessage()**进行通知。

一条消息从生产者发送到RabbitMQ，首先会发送到Exchange，对应回调函数**confirm()**。第二步从Exchange路由分配到Queue中，对应回调函数则是**returnedMessage()**。

代码怎么实现呢，请看演示：

首先在**application.yml**配置文件中加上如下配置：

```yml
spring:
  rabbitmq:
    publisher-confirms: true
#    publisher-returns: true
    template:
      mandatory: true
# publisher-confirms：设置为true时。当消息投递到Exchange后，会回调confirm()方法进行通知生产者
# publisher-returns：设置为true时。当消息匹配到Queue并且失败时，会通过回调returnedMessage()方法返回消息
# spring.rabbitmq.template.mandatory: 设置为true时。指定消息在没有被队列接收时会通过回调returnedMessage()方法退回。
```
有个小细节，**publisher-returns和mandatory如果都设置的话，优先级是以mandatory优先**。可以看源码：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200801212504531.png)
接着我们需要定义回调方法：
```java
@Component
public class RabbitmqConfirmCallback implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback {
    private Logger logger = LoggerFactory.getLogger(RabbitmqConfirmCallback.class);

    /**
     * 监听消息是否到达Exchange
     *
     * @param correlationData 包含消息的唯一标识的对象
     * @param ack             true 标识 ack，false 标识 nack
     * @param cause           nack 投递失败的原因
     */
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        if (ack) {
            logger.info("消息投递成功~消息Id：{}", correlationData.getId());
        } else {
            logger.error("消息投递失败，Id：{}，错误提示：{}", correlationData.getId(), cause);
        }
    }

    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        logger.info("消息没有路由到队列，获得返回的消息");
        Map map = byteToObject(message.getBody(), Map.class);
        logger.info("message body: {}", map == null ? "" : map.toString());
        logger.info("replyCode: {}", replyCode);
        logger.info("replyText: {}", replyText);
        logger.info("exchange: {}", exchange);
        logger.info("routingKey: {}", exchange);
        logger.info("------------> end <------------");
    }

    @SuppressWarnings("unchecked")
    private <T> T byteToObject(byte[] bytes, Class<T> clazz) {
        T t;
        try (ByteArrayInputStream bis = new ByteArrayInputStream(bytes);
             ObjectInputStream ois = new ObjectInputStream(bis)) {
            t = (T) ois.readObject();
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
        return t;
    }
}
```

我这里就简单地打印回调方法返回的消息，在实际项目中，可以把返回的消息存储到日志表中，使用定时任务进行进一步的处理。

我这里是使用**RabbitTemplate**进行发送，所以在Service层的RabbitTemplate需要设置一下：
```java
@Service
public class RabbitMQServiceImpl implements RabbitMQService {
	@Resource
    private RabbitmqConfirmCallback rabbitmqConfirmCallback;

    @Resource
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        //指定 ConfirmCallback
        rabbitTemplate.setConfirmCallback(rabbitmqConfirmCallback);
        //指定 ReturnCallback
        rabbitTemplate.setReturnCallback(rabbitmqConfirmCallback);
    }
    
    @Override
    public String sendMsg(String msg) throws Exception {
        Map<String, Object> message = getMessage(msg);
        try {
            CorrelationData correlationData = (CorrelationData) message.remove("correlationData");
            rabbitTemplate.convertAndSend(RabbitMQConfig.RABBITMQ_DEMO_DIRECT_EXCHANGE, RabbitMQConfig.RABBITMQ_DEMO_DIRECT_ROUTING, message, correlationData);
            return "ok";
        } catch (Exception e) {
            e.printStackTrace();
            return "error";
        }
    }
    
	private Map<String, Object> getMessage(String msg) {
        String msgId = UUID.randomUUID().toString().replace("-", "").substring(0, 32);
        CorrelationData correlationData = new CorrelationData(msgId);
        String sendTime = sdf.format(new Date());
        Map<String, Object> map = new HashMap<>();
        map.put("msgId", msgId);
        map.put("sendTime", sendTime);
        map.put("msg", msg);
        map.put("correlationData", correlationData);
        return map;
    }
}
```

大功告成！接下来我们进行测试，发送一条消息，我们可以控制台：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200801214001880.png)
假设发送一条信息没有路由匹配到队列，可以看到如下信息：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200801214142468.png)
这就是confirm模式。它的作用是**为了保障生产者投递消息到RabbitMQ不会出现消息丢失**。
## 3.2 事务机制(ACK)
最开始的那张图已经讲过，**消费者从队列中获取到消息后，会直接确认签收，假设消费者宕机或者程序出现异常，数据没有正常消费，这种情况就会出现数据丢失**。

所以关键在于把自动签收改成手动签收，正常消费则返回确认签收，如果出现异常，则返回拒绝签收重回队列。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200801215323179.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3llaG9uZ3poaTE5OTQ=,size_16,color_FFFFFF,t_70)
代码怎么实现呢，请看演示：

首先在消费者的**application.yml**文件中设置事务提交为**manual**手动模式：
```yml
spring:
  rabbitmq:
    listener:
      simple:
		acknowledge-mode: manual # 手动ack模式
        concurrency: 1 # 最少消费者数量
        max-concurrency: 10 # 最大消费者数量
```
然后编写消费者的监听器：
```java
@Component
public class RabbitDemoConsumer {

    enum Action {
        //处理成功
        SUCCESS,
        //可以重试的错误，消息重回队列
        RETRY,
        //无需重试的错误，拒绝消息，并从队列中删除
        REJECT
    }

    @RabbitHandler
    @RabbitListener(queuesToDeclare = @Queue(RabbitMQConfig.RABBITMQ_DEMO_TOPIC))
    public void process(String msg, Message message, Channel channel) {
        long tag = message.getMessageProperties().getDeliveryTag();
        Action action = Action.SUCCESS;
        try {
            System.out.println("消费者RabbitDemoConsumer从RabbitMQ服务端消费消息：" + msg);
            if ("bad".equals(msg)) {
                throw new IllegalArgumentException("测试：抛出可重回队列的异常");
            }
            if ("error".equals(msg)) {
                throw new Exception("测试：抛出无需重回队列的异常");
            }
        } catch (IllegalArgumentException e1) {
            e1.printStackTrace();
            //根据异常的类型判断，设置action是可重试的，还是无需重试的
            action = Action.RETRY;
        } catch (Exception e2) {
            //打印异常
            e2.printStackTrace();
            //根据异常的类型判断，设置action是可重试的，还是无需重试的
            action = Action.REJECT;
        } finally {
            try {
                if (action == Action.SUCCESS) {
                    //multiple 表示是否批量处理。true表示批量ack处理小于tag的所有消息。false则处理当前消息
                    channel.basicAck(tag, false);
                } else if (action == Action.RETRY) {
                    //Nack，拒绝策略，消息重回队列
                    channel.basicNack(tag, false, true);
                } else {
                    //Nack，拒绝策略，并且从队列中删除
                    channel.basicNack(tag, false, false);
                }
                channel.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```
解释一下上面的代码，如果没有异常，则手动确认回复RabbitMQ服务端basicAck(消费成功)。

如果抛出某些可以重回队列的异常，我们就回复basicNack并且设置重回队列。

如果是抛出不可重回队列的异常，就回复basicNack并且设置从RabbitMQ的队列中删除。

接下来进行测试，发送一条普通的消息"hello"：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200802215129880.png)
解释一下ack返回的三个方法的意思。

①成功确认
```java
void basicAck(long deliveryTag, boolean multiple) throws IOException;
```
消费者成功处理后调用此方法对消息进行确认。
- deliveryTag：该消息的index
- multiple：是否批量.。true：将一次性ack所有小于deliveryTag的消息。

②失败确认
```java
void basicNack(long deliveryTag, boolean multiple, boolean requeue) throws IOException;
```
- deliveryTag：该消息的index。
- multiple：是否批量。true：将一次性拒绝所有小于deliveryTag的消息。
- requeue：被拒绝的是否重新入队列。

③失败确认
```java
void basicReject(long deliveryTag, boolean requeue) throws IOException;
```
- deliveryTag:该消息的index。
- requeue：被拒绝的是否重新入队列。

basicNack()和basicReject()的区别在于：**basicNack()可以批量拒绝，basicReject()一次只能拒接一条消息**。

# 四、遇到的坑
## 4.1 启用nack机制后，导致的死循环
上面的代码我故意写了一个bug。测试发送一条"bad"，然后会抛出重回队列的异常。这就有个问题：重回队列后消费者又消费，消费抛出异常又重回队列，就造成了死循环。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200802215521688.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3llaG9uZ3poaTE5OTQ=,size_16,color_FFFFFF,t_70)
那怎么避免这种情况呢？

既然nack会造成死循环的话，我提供的一个思路是**不使用basicNack()，把抛出异常的消息落库到一张表中，记录抛出的异常，消息体，消息Id。通过定时任务去处理**。

如果你有什么好的解决方案，也可以留言讨论~

## 4.2 double ack
有的时候比较粗心，不小心开启了自动Ack模式，又手动回复了Ack。那就会报这个错误：
```java
消费者RabbitDemoConsumer从RabbitMQ服务端消费消息：java技术爱好者
2020-08-02 22:52:42.148 ERROR 4880 --- [ 127.0.0.1:5672] o.s.a.r.c.CachingConnectionFactory       : Channel shutdown: channel error; protocol method: #method<channel.close>(reply-code=406, reply-text=PRECONDITION_FAILED - unknown delivery tag 1, class-id=60, method-id=80)
2020-08-02 22:52:43.102  INFO 4880 --- [cTaskExecutor-1] o.s.a.r.l.SimpleMessageListenerContainer : Restarting Consumer@f4a3a8d: tags=[{amq.ctag-8MJeQ7el_PNbVJxGOOw7Rw=rabbitmq.demo.topic}], channel=Cached Rabbit Channel: AMQChannel(amqp://guest@127.0.0.1:5672/,5), conn: Proxy@782a1679 Shared Rabbit Connection: SimpleConnection@67c5b175 [delegate=amqp://guest@127.0.0.1:5672/, localPort= 56938], acknowledgeMode=AUTO local queue size=0
```

出现这个错误，可以检查一下yml文件是否添加了以下配置：
```yml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual
        concurrency: 1
        max-concurrency: 10
```

如果上面这个配置已经添加了，还是报错，**有可能你使用@Configuration配置了SimpleRabbitListenerContainerFactory，根据SpringBoot的特性，代码优于配置，代码的配置覆盖了yml的配置，并且忘记设置手动manual模式**：
```java
@Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        //设置手动ack模式
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        return factory;
    }
```

如果你还是有报错，那可能是写错地方了，写在生产者的项目了。以上的配置应该配置在消费者的项目。因为ack模式是针对消费者而言的。我就是写错了，写在生产者，折腾了几个小时，泪目~

## 4.3 性能问题
其实手动ACK相对于自动ACK肯定是会慢很多，我在网上查了一些资料，性能相差大概有10倍。所以一般在实际应用中不太建议开手动ACK模式。不过也不是绝对不可以开，具体情况具体分析，看并发量，还有数据的重要性等等。

所以**在实际项目中还需要权衡一下并发量和数据的重要性，再决定具体的方案**。

## 4.4 启用手动ack模式，如果没有及时回复，会造成队列异常
如果开启了手动ACK模式，但是由于代码有bug的原因，没有回复RabbitMQ服务端，那么这条消息就会放到Unacked状态的消息堆里，只有等到消费者的连接断开才会转到Ready消息。如果消费者一直没有断开连接，那Unacked的消息就会越来越多，占用内存就越来越大，最后就会出现异常。

这个问题，我没法用我的电脑演示，我的电脑太卡了。

# 五、总结
通过上面的学习后，总结了RabbitMQ防止数据丢失有三种方式：
- 消息持久化
- 生产者消息确认机制(confirm模式)
- 消费者消息确认模式(ack模式)

上面所有例子的代码都上传github了：

> https://github.com/yehongzhi/mall

**如果你觉得这篇文章对你有用，点个赞吧**~

**你的点赞是我创作的最大动力**~

想第一时间看到我更新的文章，可以微信搜索公众号「`java技术爱好者`」，**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**
![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC82LzMwLzE3MzA1Y2MwOGE3ZWQ1ZDc?x-oss-process=image/format,png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！