---
title: 超详细的Sentinel入门
date: 2021-04-05 14:42:55
index_img: https://static.lovebilibili.com/Sentinel_index.jpg
tags:
	- Sentinel
---

> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 一、什么是Sentinel

Sentinel定位是分布式系统的流量防卫兵。目前互联网应用基本上都使用微服务，微服务的稳定性是一个很重要的问题，而**限流、熔断降级**是微服务保持稳定的一个重要的手段。

下面看官网的一张图，了解一下Sentinel的主要特性：

![](https://static.lovebilibili.com/sentinel_01.png)

在Sentinel之前其实就有Hystrix做熔断降级的事情，我们都知道出现新的事物肯定是原来的东西有不足的地方。

> 那Hystrix有什么不足之处呢？

- Hystrix常用的线程池隔离会造成线程上下切换的overhead比较大。
- Hystrix没有监控平台，需要我们自己搭建。
- Hystrix支持的熔断降级维度较少，不够细粒，而且缺少管理控制台。

> Sentinel有哪些组成部分？

- 核心库（Java 客户端）不依赖任何框架/库，能够运行于所有 Java 运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持。
- 控制台（Dashboard）基于 Spring Boot 开发，打包后可以直接运行，不需要额外的 Tomcat 等应用容器。

> Sentinel有哪些特征？

- **丰富的应用场景**。控制突发流量在可控制的范围内，消息削峰填谷，集群流量控制，实时熔断下游不可用的应用等等。
- **完备的实时监控**。Sentinel 提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。

- **广泛的开源生态**。Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Dubbo、gRPC 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。
- **完善的 SPI 扩展点**。Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源等。

# 二、Hello World

一般要学一种没接触过的技术框架，肯定要先做个Hello World熟悉一下。

> 引入Maven依赖

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-core</artifactId>
    <version>1.8.1</version>
</dependency>
```

需要提醒一下，Sentinel仅支持JDK 1.8或者以上的版本

> 定义规则

通过定义规则来控制该资源每秒允许通过的请求次数，例如下面的代码定义了资源 `HelloWorld` 每秒最多只能通过 20 个请求。

```java
private static void initFlowRules(){
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule = new FlowRule();
    rule.setResource("HelloWorld");
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    // Set limit QPS to 20.
    rule.setCount(20);
    rules.add(rule);
    FlowRuleManager.loadRules(rules);
}
```

> 编写Hello World代码

其实代码编写很简单，首先需要定义一个资源entry，然后用`SphU.entry("HelloWorld")`和`entry.exit()`把需要流量控制的代码包围起来。代码如下：

```java
public static void main(String[] args) throws Exception {
    initFlowRules();
    while (true) {
        Entry entry = null;
        try {
            entry = SphU.entry("HelloWorld");
            /*您的业务逻辑 - 开始*/
            System.out.println("hello world");
            /*您的业务逻辑 - 结束*/
        } catch (BlockException e1) {
            /*流控逻辑处理 - 开始*/
            System.out.println("block!");
            /*流控逻辑处理 - 结束*/
        } finally {
            if (entry != null) {
                entry.exit();
            }
        }
    }
}
```

运行结果如下：

![](https://static.lovebilibili.com/sentinel_02.png)

我们根据目录查看日志，文件名格式为${appName}-metrics.log.xxx：

```java
|--timestamp-|------date time----|-resource-|p |block|s |e|rt
1616607101000|2021-03-25 01:31:41|HelloWorld|20|11373|20|0|1|0|0|0
1616607102000|2021-03-25 01:31:42|HelloWorld|20|24236|20|0|0|0|0|0
```

 `p` 代表通过的请求。

 `block` 代表被阻止的请求。

`s` 代表成功执行完成的请求个数。 

`e` 代表用户自定义的异常。

 `rt` 代表平均响应时长。

# 三、使用Sentinel的方式

下面结合实际案例，写一个Controller接口进行示范练习。

```java
@RestController
@RequestMapping("/user")
public class UserController {
    @Resource
    private UserService userService;

    @RequestMapping("/list")
    public List<User> getUserList() {
        return userService.getList();
    }
}

@Service
public class UserServiceImpl implements UserService {
    //模拟查询数据库数据，返回结果
    @Override
    public List<User> getList() {
        List<User> userList = new ArrayList<>();
        userList.add(new User("1", "周慧敏", 18));
        userList.add(new User("2", "关之琳", 20));
        userList.add(new User("3", "王祖贤", 21));
        return userList;
    }
}
```

假设我们要让这个查询接口限流，怎么做呢？

## 1) 抛出异常的方式

`SphU` 包含了 try-catch 风格的 API。用这种方式，当资源发生了限流之后会抛出 `BlockException`。这个时候可以捕捉异常，进行限流之后的逻辑处理。

```java
@RestController
@RequestMapping("/user")
public class UserController {
	//资源名称
    public static final String RESOURCE_NAME = "userList";

    @Resource
    private UserService userService;

    @RequestMapping("/list")
    public List<User> getUserList() {
        List<User> userList = null;
        Entry entry = null;
        try {
            // 被保护的业务逻辑
            entry = SphU.entry(RESOURCE_NAME);
            userList = userService.getList();
        } catch (BlockException e) {
            // 资源访问阻止，被限流或被降级
            return Collections.singletonList(new User("xxx", "资源访问被限流", 0));
        } catch (Exception e) {
            // 若需要配置降级规则，需要通过这种方式记录业务异常
            Tracer.traceEntry(e, entry);
        } finally {
            // 务必保证 exit，务必保证每个 entry 与 exit 配对
            if (entry != null) {
                entry.exit();
            }
        }
        return userList;
    }

}
```

实际上还没写完，还要定义限流的规则。

```java
@SpringBootApplication
public class SpringmvcApplication {

    public static void main(String[] args) throws Exception {
        SpringApplication.run(SpringmvcApplication.class, args);
        //初始化限流规则
        initFlowQpsRule();
    }
	//定义了每秒最多接收2个请求
    private static void initFlowQpsRule() {
        List<FlowRule> rules = new ArrayList<>();
        FlowRule rule = new FlowRule(UserController.RESOURCE_NAME);
        // set limit qps to 2
        rule.setCount(2);
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setLimitApp("default");
        rules.add(rule);
        FlowRuleManager.loadRules(rules);
    }
}
```

然后启动项目，测试。快速刷新几次，我们就看到触发限流的逻辑了。

![](https://static.lovebilibili.com/sentinel_03.png)

## 2) 返回布尔值的方式

抛出异常的方式是当被限流时以抛出异常的形式感知，我们通过捕获异常进行限流的处理，这种方式跟上面不同的在于不抛出异常，而是返回一个布尔值，我们通过判断布尔值来进行限流逻辑的处理。这样我们就可以很容易写出`if-else`结构的代码。

```java
public static final String RESOURCE_NAME_QUERY_USER_BY_ID = "queryUserById";

@RequestMapping("/get/{id}")
public String queryUserById(@PathVariable("id") String id) {
    if (SphO.entry(RESOURCE_NAME_QUERY_USER_BY_ID)) {
        try {
            //被保护的逻辑
            //模拟数据库查询数据
            return JSONObject.toJSONString(new User(id, "Tom", 25));
        } finally {
            //关闭资源
            SphO.exit();
        }
    } else {
        //资源访问阻止，被限流或被降级
        return "Resource is Block!!!";
    }
}
```

添加规则的代码跟前面的例子一样，我就不写了，然后启动项目，测试。

![](https://static.lovebilibili.com/sentinel_04.png)

## 3) 注解的方式

看了上面两种方式，肯定有人会说，代码侵入性太强了，如果原来旧的系统要接入的话，要改原来的代码。众所周知，旧代码是不能动的，否则后果很严重。

那么注解的方式就很好地解决了这个问题。注解式怎么写呢？

```java
@Service
public class UserServiceImpl implements UserService {
    //资源名称
    public static final String RESOURCE_NAME_QUERY_USER_BY_NAME = "queryUserByUserName";

    //value是资源名称，是必填项。blockHandler填限流处理的方法名称
    @Override
    @SentinelResource(value = RESOURCE_NAME_QUERY_USER_BY_NAME, blockHandler = "queryUserByUserNameBlock")
    public User queryByUserName(String userName) {
        return new User("0", userName, 18);
    }

    //注意细节，一定要跟原函数的返回值和形参一致，并且形参最后要加个BlockException参数
    //否则会报错，FlowException: null
    public User queryUserByUserNameBlock(String userName, BlockException ex) {
        //打印异常
        ex.printStackTrace();
        return new User("xxx", "用户名称：{" + userName + "},资源访问被限流", 0);
    }
}
```

写完这个核心代码后，还要加个配置，否则不生效。

引入`sentinel-annotation-aspectj`的Maven依赖。

```java
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-annotation-aspectj</artifactId>
    <version>1.8.1</version>
</dependency>
```

然后将`SentinelResourceAspect`注册为一个Bean。

```java
@Configuration
public class SentinelAspectConfiguration {
    @Bean
    public SentinelResourceAspect sentinelResourceAspect() {
        return new SentinelResourceAspect();
    }
}
```

别忘了添加规则，可以参考第一个例子，这里就不写了。

最后启动项目，测试，刷新多几次接口后，出发限流，可以看到以下结果。

![](https://static.lovebilibili.com/sentinel_05.png)

## 4) 熔断降级

除了可以对接口进行限流之外，当接口出现异常时，Sentinel也可以提供熔断降级的功能。

在`@SentinelResource`注解中有一个属性`fallback`，当抛出非BlockException的异常时，就会进入到fallback方法中，实现熔断机制，这有点类似于Hystrix的FallBack。

我们拿上面的例子做示范，如果userName为空则抛出RuntimeException。然后我们设置fallback属性的属性值，也就是fallback的方法，返回系统异常。

```java
@Override
@SentinelResource(value = RESOURCE_NAME_QUERY_USER_BY_NAME, blockHandler = "queryUserByUserNameBlock", fallback = "queryUserByUserNameFallBack")
public User queryByUserName(String userName) {
    if (userName == null || "".equals(userName)) {
        //抛出异常
        throw new RuntimeException("queryByUserName() command failed, userName is null");
    }
    return new User("0", userName, 18);
}

public User queryUserByUserNameFallBack(String userName, Throwable ex) {
    //打印日志
    ex.printStackTrace();
    return new User("-1", "用户名称：{" + userName + "},系统异常，请稍后重试", 0);
}
```

然后启动项目，故意不传userName，进行测试，可以看到走了fallback的方法逻辑。

![](https://static.lovebilibili.com/sentinel_06.png)

IDEA控制台也可以看到自定义的异常信息。

![](https://static.lovebilibili.com/sentinel_07.png)

# 四、管理控制台

上面讲完了Sentinel的基本用法，实际上重头戏在Sentinel的管理控制台，管理控制台提供了很多实用的功能。下面我们看看怎么使用。

首先下载控制台的jar包，当然你也可以通过下载源码编译得到。

```java
//下载页面地址
https://github.com/alibaba/Sentinel/releases
```

然后使用以下命令启动：

```java
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.1.jar
```

启动成功后，访问`http://localhost:8080`，默认登录的用户名和密码都是`sentinel`。

![](https://static.lovebilibili.com/sentinel_08.png)

登录进去之后，可以看到主页面，有许多功能菜单，这里就不一一介绍了。

![](https://static.lovebilibili.com/sentinel_09.png)

## 客户端接入控制台

那么我们自己的应用怎么接入到控制台，使用控制台对应用的流量进行监控呢，诸位客官，请继续往下看。

首先添加maven依赖，客户端需要引入 Transport 模块来与 Sentinel 控制台进行通信。

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>1.8.1</version>
</dependency>
```

配置filter，把所有访问的 Web URL 自动统计为 Sentinel 的资源。

```java
@Configuration
public class FilterConfig {
    @Bean
    public FilterRegistrationBean sentinelFilterRegistration() {
        FilterRegistrationBean<Filter> registration = new FilterRegistrationBean<>();
        registration.setFilter(new CommonFilter());
        registration.addUrlPatterns("/*");
        registration.setName("sentinelFilter");
        registration.setOrder(1);

        return registration;
    }
}
```

在启动命令中加入以下配置，`-Dcsp.sentinel.dashboard.server=consoleIp:port` 指定控制台地址和端口，`-Dcsp.sentinel.api.port=xxxx` 指定客户端监控 API 的端口(默认是8019，因为控制台已经使用了8719，应用端为了防止冲突就使用8720)：

```java
-Dserver.port=8888 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dcsp.sentinel.api.port=8720 -Dproject.name=sentinelDemo
```

![](https://static.lovebilibili.com/sentinel_11.png)

启动项目，我们可以看到多了一个应用名称sentinelDemo，点击机器列表，查看健康状况。

![](https://static.lovebilibili.com/sentinel_12.png)

请求`/user/list`接口，然后我们可以看到实时监控的接口的QPS情况。

![](https://static.lovebilibili.com/sentinel_10.png)

这样就代表客户端接入控制台成功了！

## 动态规则

Sentinel 的理念是开发者只需要关注资源的定义，当资源定义成功后可以动态增加各种流控降级规则。Sentinel 提供两种方式修改规则：

- 通过 API 直接修改 (`loadRules`)
- 通过 `DataSource` 适配不同数据源修改

手动通过API定义规则，前面Hello World的例子已经写过，是一种硬编码的形式，因为不够灵活，所以肯定不能应用于生产环境。

所以要引入`DataSource`，规则设置可以存储在数据源中，通过更新数据源中存储的规则，推送到Sentinel规则中心，客户端就可以实时获取最新的规则，根据最新的规则进行限流、降级。

一般`DataSource`拓展常见的实现方式有：

- 拉模式：**客户端主动向某个规则管理中心定期轮询拉取规则**，这个规则中心可以是SQL、文件等。优点是比较简单，缺点是无法及时获取变更。
- 推模式：规则中心统一推送，客户端通过注册监听器的方式时刻监听变化，比如使用Nacos、Zookeeper 等配置中心。这种方式有更好的实时性和一致性保证，比较推荐使用这种方式。

### 拉模式

pull模式的数据源一般是可写入的(比如本地文件)。首先要在客户端注册数据源，将对应的读数据源注册至对应的 RuleManager；然后将写数据源注册至 transport 的 `WritableDataSourceRegistry` 中。

由此看出这是一个双向读写的过程，我们既可以在应用本地直接修改文件来更新规则，也可以通过 Sentinel 控制台推送规则。下图为控制台推送规则的流程图。

![](https://static.lovebilibili.com/sentinel_13.png)

首先引入maven依赖。

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-extension</artifactId>
    <version>1.8.1</version>
</dependency>
```

使用SPI机制进行扩展，创建一个实现类，实现InitFunc接口的init()方法。

```java
public class FileDataSourceInit implements InitFunc {

    public FileDataSourceInit() {
    }

    @Override
    public void init() throws Exception {
        String filePath = System.getProperty("user.home") + "\\sentinel\\rules\\sentinel.json";
        ReadableDataSource<String, List<FlowRule>> ds = new FileRefreshableDataSource<>(
            filePath, source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {
            })
        );
        // 将可读数据源注册至 FlowRuleManager.
        FlowRuleManager.register2Property(ds.getProperty());

        WritableDataSource<List<FlowRule>> wds = new FileWritableDataSource<>(filePath, this::encodeJson);
        // 将可写数据源注册至 transport 模块的 WritableDataSourceRegistry 中.
        // 这样收到控制台推送的规则时，Sentinel 会先更新到内存，然后将规则写入到文件中.
        WritableDataSourceRegistry.registerFlowDataSource(wds);
    }

    private <T> String encodeJson(T t) {
        return JSON.toJSONString(t);
    }
}
```

在项目的 `resources/META-INF/services` 目录下创建文件，名为`com.alibaba.csp.sentinel.init.InitFunc` ，内容则是FileDataSourceInit的全限定名称：

```java
io.github.yehongzhi.springmvc.config.FileDataSourceInit
```

![](https://static.lovebilibili.com/sentinel_14.png)

接着在${home}目录下，创建`\sentinel\rules`目录，再创建sentinel.json文件。

![](https://static.lovebilibili.com/sentinel_15.png)

然后启动项目，发送请求，当客户端接收到请求后就会触发初始化操作。初始化完成后我们到控制台，然后设置流量限流规则。

![](https://static.lovebilibili.com/sentinel_16.png)

新增后，本地文件`sentinel.json`同时也保存了规则内容（压缩成一行的json）。

```json
[{"clusterConfig":{"acquireRefuseStrategy":0,"clientOfflineTime":2000,"fallbackToLocalWhenFail":true,"resourceTimeout":2000,"resourceTimeoutStrategy":0,"sampleCount":10,"strategy":0,"thresholdType":0,"windowIntervalMs":1000},"clusterMode":false,"controlBehavior":0,"count":3.0,"grade":1,"limitApp":"default","maxQueueingTimeMs":500,"resource":"userList","strategy":0,"warmUpPeriodSec":10}]
```

我们可以通过修改文件来更新规则内容，也可以通过控制台推送规则到文件中，这就是拉模式。缺点是不保证一致性，实时性不保证，拉取过于频繁也可能会有性能问题。

### 推模式

刚刚说了拉模式实时性不能保证，推模式就解决了这个问题。除此之外还可以持久化，也就是数据保存在数据源中，即使重启也不会丢失之前的配置，这也解决了原始模式存在内存中不能持久化的问题。

可以和Sentinel配合使用的数据源有很多种，比如ZooKeeper，Nacos，Apollo等等。这里介绍使用Nacos的方式。

首先要启动Nacos服务器，然后登录到Nacos控制台，添加一个命名空间，添加配置。

![](https://static.lovebilibili.com/sentinel_19.png)

接着我们就要改造Sentinel的源码。因为官网提供的Sentinel的jar是原始模式的，所以需要改造，所以我们需要拉取源码下来改造一下，然后自己编译jar包。

> 源码地址：https://github.com/alibaba/Sentinel

拉取下来之后，导入到IDEA中，然后我们可以看到以下目录结构。

![](https://static.lovebilibili.com/sentinel_17.png)

首先修改sentinel-dashboard的`pom.xml`文件：

![](https://static.lovebilibili.com/sentinel_18.png)

第二步，把test目录下的四个关于Nacos关联的类，移到rule目录下。

![](https://static.lovebilibili.com/sentinel_20.png)

FlowRuleNacosProvider和FlowRuleNacosPublisher不需要怎么改造，本人不太喜欢名称后缀，所以去掉了后面的后缀。

![](https://static.lovebilibili.com/sentinel_21.png)

接着NacosConfig添加Nacos的地址配置。

![](https://static.lovebilibili.com/sentinel_22.png)

最关键的是FlowControllerV1的改造，这是规则配置的增删改查的一些接口。

把移动到rule目录下的两个服务，添加到FlowControllerV1类中。

```java
@Autowired
@Qualifier("flowRuleNacosProvider")
private DynamicRuleProvider<List<FlowRuleEntity>> ruleProvider;
@Autowired
@Qualifier("flowRuleNacosPublisher")
private DynamicRulePublisher<List<FlowRuleEntity>> rulePublisher;
```

添加私有方法publishRules()，用于推送配置：

```java
private void publishRules(/*@NonNull*/ String app) throws Exception {
    List<FlowRuleEntity> rules = repository.findAllByApp(app);
    rulePublisher.publish(app, rules);
}
```

修改apiQueryMachineRules()方法。

![](https://static.lovebilibili.com/sentinel_23.png)

修改apiAddFlowRule()方法。

![](https://static.lovebilibili.com/sentinel_24.png)

修改apiUpdateFlowRule()方法。

![](https://static.lovebilibili.com/sentinel_25.png)

修改apiDeleteFlowRule()方法。

![](https://static.lovebilibili.com/sentinel_26.png)

Sentinel控制台的项目就改造完成了，用于生产环境就编译成jar包运行，如果是学习可以直接在IDEA运行。

我们在前面创建的HelloWord工程的pom.xml文件加上依赖。

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
    <version>1.8.1</version>
</dependency>
```

然后在application.yml文件加上以下配置：

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            namespace: 05f447bc-8a0b-4686-9c34-344d7206ea94
            dataId: springmvc-sentinel-flow-rules
            groupId: SENTINEL_GROUP
            # 规则类型，取值见：
            # org.springframework.cloud.alibaba.sentinel.datasource.RuleType
            rule-type: flow
            data-type: json
  application:
    name: springmvc-sentinel-flow-rules
```

以上就完成了全部的配置和改造，启动Sentinel控制台，还有Java应用。

打开Nacos控制台，我们添加限流配置如下：

![](https://static.lovebilibili.com/sentinel_27.png)

配置内容如下：

```json
[{"app":"springmvc-sentinel-flow-rules","clusterConfig":{"acquireRefuseStrategy":0,"clientOfflineTime":2000,"fallbackToLocalWhenFail":true,"resourceTimeout":2000,"resourceTimeoutStrategy":0,"sampleCount":10,"strategy":0,"thresholdType":0,"windowIntervalMs":1000},"clusterMode":false,"controlBehavior":0,"count":1.0,"grade":1,"limitApp":"default","maxQueueingTimeMs":500,"resource":"userList","strategy":0,"warmUpPeriodSec":10},{"app":"springmvc-sentinel-flow-rules","clusterConfig":{"acquireRefuseStrategy":0,"clientOfflineTime":2000,"fallbackToLocalWhenFail":true,"resourceTimeout":2000,"resourceTimeoutStrategy":0,"sampleCount":10,"strategy":0,"thresholdType":0,"windowIntervalMs":1000},"clusterMode":false,"controlBehavior":0,"count":3.0,"grade":1,"limitApp":"default","maxQueueingTimeMs":500,"resource":"queryUserByUserName","strategy":0,"warmUpPeriodSec":10}]
```

然后我们打开Sentinel控制台，能看到配置，证明Nacos的配置推送成功了。

![](https://static.lovebilibili.com/sentinel_28.png)

我们尝试调用Java应用的接口，测试是否生效。

![](https://static.lovebilibili.com/sentinel_29.png)

可以看到限流是生效的，再看看Sentinel监控的QPS情况。

![](https://static.lovebilibili.com/sentinel_30.png)

从QPS监控的情况看，最高的QPS只有3，其他请求都被拒绝了，证明限流配置是实时生效的。

![](https://static.lovebilibili.com/sentinel_31.png)

配置信息也被持久化到Nacos相关的配置表中。

这时候，再回头看Sentinel官网上关于推模式的架构图就比较清楚了。

![](https://static.lovebilibili.com/sentinel_33.png)

# 总结

本篇文章主要介绍了Sentinel的基本用法，还有动态规则的两种方式，除此之外当然还有许多功能，这里由于篇幅问题就不一一介绍了，有兴趣的朋友可以自己探索一下。我个人觉得Sentinel是一个非常优秀的组件，比原来用的Hystrix的确有着非常大的改进，值得推荐。

我们看到官网上登记的企业列表，也有很多知名企业在使用，相信以后Sentinel会越来越好。

![](https://static.lovebilibili.com/sentinel_32.png)

这篇文章就讲到这里了，感谢大家的阅读，希望看完大家能有所收获！

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！

