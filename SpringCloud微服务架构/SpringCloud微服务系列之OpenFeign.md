# 思维导图

![](https://static.lovebilibili.com/fegin_swdt.png)

> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 前言

目前在SpringCloud技术栈中，调用服务用得最多的就是OpenFeign，所以这篇文章讲一下OpenFeign，希望对大家有所帮助。

# 一、构建工程

使用Nacos作为注册中心，不会搭建Nacos的话，可以参考上一篇注册中心的文章。

首先父工程parent引入依赖。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>0.2.2.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-openfeign</artifactId>
            <version>2.0.1.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency><!-- SpringCloud nacos服务发现的依赖 -->
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.nacos</groupId>
        <artifactId>nacos-client</artifactId>
        <version>1.2.0</version>
    </dependency>
</dependencies>
```

搭建提供者provider工程和消费者consumer工程。

provider工程继承父工程的pom文件，编写启动类如下：

```java
@SpringBootApplication
@EnableDiscoveryClient//注册中心
public class ProviderApplication {
    public static void main(String[] args) throws Exception {
        SpringApplication.run(ProviderApplication.class, args);
    }
}
```

provider工程的配置文件如下：

```yaml
server:
  port: 8080
spring:
  application:
    name: provider
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        service: ${spring.application.name}
```

提供接口，Controller如下：

```java
@RestController
public class ProviderController {
    @RequestMapping("/provider/list")
    public List<String> list() {
        List<String> list = new ArrayList<>();
        list.add("java技术爱好者");
        list.add("SpringCloud");
        list.add("没有人比我更懂了");
        return list;
    }
}
```

消费者consumer工程也继承parent的pom文件，加上Feign依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
        <!-- 版本在parent的pom文件中指定了 -->
    </dependency>
</dependencies>
```

编写启动类，如下：

```java
@SpringBootApplication
@EnableDiscoveryClient
//开启feign接口扫描，指定扫描的包
@EnableFeignClients(basePackages = {"com.yehongzhi.springcloud"})
public class ConsumerApplication {
    public static void main(String[] args) throws Exception {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```

环境搭建完成后，接下来讲两种实现使用方式。

# 二、声明式

这种很简单，消费者consumer工程增加一个ProviderClient接口。

```java
@FeignClient(name = "provider")
//会扫描指定包下，标记FeignClient注解的接口
//会根据服务名，从注册中心找到对应的IP地址
public interface ProviderClient {
    //这里跟提供者接口的URL一致
    @RequestMapping("/provider/list")
    String list();
}
```

然后再用消费者工程的ConsumerController接口来测试。

```java
@RestController
public class ConsumerController {
	//引入Feign客户端
    @Resource
    private ProviderClient providerClient;

    @RequestMapping("/consumer/callProvider")
    public String callProvider() {
        //使用Feign客户端调用其他服务的接口
        return providerClient.list();
    }
}
```

最后我们启动提供者工程，消费者工程，注册中心，测试。

![](https://static.lovebilibili.com/fegin_1.png)

然后调用消费者的ConsumerController接口。

![](https://static.lovebilibili.com/fegin_2.png)

# 三、继承式

细心的同学可能发现，其实声明式会写多一次提供者接口的定义，也就是有重复的代码，既然有重复的定义，那我们就可以抽取出来，所以就有了继承式。

第一步，创建一个普通的Maven项目api工程，把接口定义在api中。

![](https://static.lovebilibili.com/fegin_3.png)

第二步，服务提供者工程的ProviderController实现Provider接口。

```java
@RestController
public class ProviderController implements ProviderApi {
    
    public String list() {
        List<String> list = new ArrayList<>();
        list.add("java技术爱好者");
        list.add("SpringCloud");
        list.add("没有人比我更懂了");
        return list.toString();
    }
}
```

第三步，消费者工程的ProviderClient无需定义，只需要继承ProviderApi，然后加上@FeignClient即可。

```java
@FeignClient(name = "provider")
public interface ProviderClient extends ProviderApi {
}
```

其他不用变了，最后启动服务提供者，消费者，注册中心测试一下。

![](https://static.lovebilibili.com/fegin_4.png)

测试成功！上面继承式的好处就在于，只需要在api工程定义一次接口，服务提供者去实现具体的逻辑，消费者则继承接口贴个注解即可，非常方便快捷。

缺点就在于如果有人动了api的接口，则会导致很多服务消费者、提供者出现报错，耦合性比较强。api工程相当于一个公共的工程，消费者和服务者都会依赖此工程，所以一般要求不能随便删api上面的接口。

# 四、Feign的相关配置

下面讲一下Feign的一些常用的相关配置。

## 4.1 请求超时设置

Feign底层其实还是使用Ribbon，默认是1秒。所以超过1秒就报错。

接下来试验一下。我在服务提供者的接口加上一段休眠1.5秒的代码，然后用消费者去消费。

```java
@RestController
public class ProviderController implements ProviderApi {
    public String list() {
        List<String> list = new ArrayList<>();
        list.add("java技术爱好者");
        list.add("SpringCloud");
        list.add("没有人比我更懂了");
        try {
            //休眠1.5秒
            Thread.sleep(1500);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return list.toString();
    }
}
```

消费者调用后，由于超过1秒，可以看到控制台报错。

![](https://static.lovebilibili.com/fegin_5.png)

如果想调整超时时间，可以在消费者这边，加上配置：

```yaml
ribbon:
  ReadTimeout:  5000 #请求时间5秒
  ConnectTimeout: 5000 #连接时间5秒
```

为了显示出效果，我们在消费者的代码里加上耗时计算：

```java
@RestController
public class ConsumerController {

    @Resource
    private ProviderClient providerClient;

    @RequestMapping("/consumer/callProvider")
    public String callProvider() throws Exception {
        long star = System.currentTimeMillis();
        String list = providerClient.list();
        long end = System.currentTimeMillis();
        return "响应结果：" + list + ",耗时：" + (end - star) / 1000 + "秒";
    }
}
```

最后启动测试，可以看到，超过1秒也能请求成功。

![](https://static.lovebilibili.com/fegin_6.png)

## 4.2 日志打印功能

首先需要配置Feign的打印日志的级别。

```java
@Configuration
public class FeignConfig {
    /**
     * NONE：默认的，不显示任何日志
     * BASIC：仅记录请求方法、URL、响应状态码及执行时间
     * HEADERS：出了BASIC中定义的信息之外，还有请求和响应的头信息
     * FULL：除了HEADERS中定义的信息之外，还有请求和响应的正文及元素
     */
    @Bean
    public Logger.Level feginLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

第二步，需要设置打印的Feign接口。Feign为每个客户端创建一个logger。默认情况下，logger的名称是Feigh接口的完整类名。需要注意的是，**Feign的日志打印只会对DEBUG级别做出响应**。

```yaml
#与server同级
logging:
  level:
    com.yehongzhi.springcloud.consumer.feign.ProviderClient: debug
```

设置完成后，控制台可以看到详细的请求信息。
![](https://static.lovebilibili.com/fegin_7.png)

## 4.3 Feign实现熔断

openFeign实际上是已经引入了hystrix的相关jar包，所以可以直接使用，设置超时时间，超时后调用FallBack方法，实现熔断机制。

首先在消费者工程添加Maven依赖。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

第二步，在配置中开启熔断机制，添加超时时间。

```yaml
#默认是不支持的，所以这里要开启，设置为true
feign:
  hystrix:
    enabled: true
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
```

第三步，编写FallBack类。

```java
//ProviderClient是贴了@FeignClient注解的接口
@Component
public class ProviderClientFallBack implements ProviderClient {
    @Override
    public String list() {
        return Arrays.asList("调用fallBack接口", "返回未知结果").toString();
    }
}
```

第四步，在对应的Feign接口添加fallback属性。

```java
//fallback属性，填写刚刚编写的FallBack回调类
@Component
@FeignClient(name = "provider", fallback = ProviderClientFallBack.class)
public interface ProviderClient extends ProviderApi {
}
```

最后可以测试一下，超过设置的3秒，则会熔断，调用FallBack方法返回。

![](https://static.lovebilibili.com/fegin_8.png)

## 4.4 设置负载均衡

前面说过OpenFeign底层是使用Ribbon，Ribbon是负责做负载均衡的组件。所以是可以通过配置设置负载均衡的策略。

默认的是轮询策略。如果要换成其他策略，比如随机，怎么换呢。

很简单，改一下配置即可：

```yaml
#服务名称
provider:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
#NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #配置规则 随机
#NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #配置规则 轮询
#NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RetryRule #配置规则 重试
#NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule #配置规则 响应时间权重
#NFLoadBalancerRuleClassName: com.netflix.loadbalancer.BestAvailableRule #配置规则 最空闲连接策略
```

# 总结

OpenFeign把RestTemplete，Ribbon，Hystrix糅合在了一起，在使用时就可以更加方便，优雅地完成整个服务的暴露，调用等。避免做一些重复的复制粘贴接口URL，或者重复定义接口等。还是非常值得去学习的。

以前我在的公司搭建的SpringCloud微服务就没有使用Feign，架构师自己写了一个AOP代理类进行服务调用，超时时间5秒写死在代码里，当时有个微服务接口要上传文件，总是超时，又改不了超时时间，一超时就调熔断方法返回服务请求超时，导致非常痛苦。

如果当时使用Feign，插拔式，可配置的方式，也许就没那么麻烦了。

那么feign就讲到这里了，上面所有例子的代码都上传Github了：

> https://github.com/yehongzhi/example

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！