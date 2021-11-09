> **文章已收录到我的Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 写在前面

前一篇文章写了Gateway的Predicate(用于路由转发)，那么这篇文章就介绍另一个主要的核心，那就是Filter(过滤器)。

过滤器有什么作用呢？工作流程是怎么样的呢？请看下图：

![](https://static.lovebilibili.com/gateway_filter_01.png)

从图中很明显可以看出，在请求后端服务前后都需要经过Filter，于是乎Filter的作用就明确了，在PreFilter(请求前处理)可以做参数校验、流量监控、日志记录、修改请求内容等等，在PostFilter(请求后处理)可以做响应内容修改。

# 过滤器

Filter分为局部和全局两种：

- 局部Filter(GatewayFilter的子类)是作用于单个路由。如果需要使用全局路由，需要配置Default Filters。
- 全局Filter(GlobalFilter的子类)，不需要配置路由，系统初始化作用到所有路由上。

## 局部过滤器

SpringCloud Gateway内置了很多路由过滤器，他们都是由GatewayFilter的工厂类产生。

### AddRequestParameter GatewayFilter

该过滤器可以给请求添加参数。

比如我在consumer服务有一个带有`userName`参数的接口，我想请求网关路由转发的时候给加上一个`userName=yehongzhi`的参数。

```java
@RequestMapping(value = "/getOrder",method = RequestMethod.GET)
public String getOrder(@RequestParam(name = "userName") String userName) {
    return "获取到传入的用户名称:" + userName;
}
```

配置如下：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_parameter
          uri: http://localhost:8081/getOrder
          predicates:
            - Method=GET
            - Path=/getOrder
          filters:
            - AddRequestParameter=userName,yehongzhi
```

那么当我请求网关时，输入`http://localhost:9201/getOrder`，我们能看到默认加上的userName。

![](https://static.lovebilibili.com/gateway_filter_02.png)

### StripPrefix GatewayFilter

该过滤器可以去除指定数量的路径前缀。

比如我想把请求网关的路径前缀的第一级去掉，就可以这样配置实现：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: strip_prefix_gateway
          uri: http://localhost:8081
          predicates:
            - Path=/consumer/**
          filters:
            - StripPrefix=1
```

当请求路径`http://localhost:9201/consumer/getDetail/1`，能获得结果。

![](https://static.lovebilibili.com/gateway_filter_04.png)

相当于请求`http://localhost:8081/getDetail/1`，结果是一样的。

![](https://static.lovebilibili.com/gateway_filter_03.png)

### PrefixPath GatewayFilter

该过滤器与上一个过滤器相反，是给原有的路径加上指定的前缀。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: prefix_path_gateway
          uri: http://localhost:8081
          predicates:
            - Path=/getUserInfo/**
          filters:
            - PrefixPath=/consumer
```

当请求`http://localhost:9201/getUserInfo/1`时，跟请求`http://localhost:8081/consumer/getUserInfo/1`是一样的。

![](https://static.lovebilibili.com/gateway_filter_05.png)

![](https://static.lovebilibili.com/gateway_filter_06.png)

### Hystrix GatewayFilter

网关当然有熔断机制，所以该过滤器集成了Hystrix，实现了熔断的功能。怎么使用呢？首先需要引入Hystrix的maven依赖。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

在gateway服务添加fallback()方法

```java
@RestController
public class FallBackController {
    @RequestMapping("/fallback")
    public String fallback(){
        return "系统繁忙，请稍后再试！";
    }
}
```

在网关服务的配置如下，对GET请求方式的请求路由转发出错时，会触发服务降级：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: hystrix_filter
          uri: http://localhost:8081
          predicates:
            - Method=GET
          filters:
            - name: Hystrix
              args:
                name: fallbackcmd
                fallbackUri: forward:/fallback
```

这时我们把`8081`的服务停止，让网关请求不到对应的服务，从而触发服务降级。

![](https://static.lovebilibili.com/gateway_filter_07.png)

### RequestRateLimiter GatewayFilter

该过滤器可以提供限流的功能，使用RateLimiter实现来确定是否允许当前请求继续进行，如果请求太大默认会返回HTTP 429-太多请求状态。怎么用呢？首先还是得引入Maven依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

接着增加个配置类。

```java
@Configuration
public class LimiterConfig {

    /**
     * ip限流器
     */
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
    }
}
```

然后配置如下，对GET方式的请求增加限流策略：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: hystrix_filter
          uri: http://localhost:8081
          predicates:
            - Method=GET
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 1 #每秒允许处理的请求数量
                redis-rate-limiter.burstCapacity: 2 #每秒最大处理的请求数量
                key-resolver: "#{@ipKeyResolver}" #限流策略，对应策略的Bean
  redis:
    port: 6379
    host: 192.168.1.5 #redis地址
```

然后启动服务，连续请求地址`http://localhost:9201/getDetail/1`，就会触发限流，报429错误。

![](https://static.lovebilibili.com/gateway_filter_08.png)

因为内置的过滤器实在是太多了，这里就不一一列举了，有兴趣的同学可以到官网自行学习。

### 自定义局部过滤器

如果内置的局部过滤器不能满足需求，那么我们就得使用自定义过滤器，怎么用呢？下面用一个例子，我们自定义一个白名单的过滤器，userName在白名单内的才可以访问，不在白名单内的就返回401错误码(Unauthorized)。

局部过滤器需要实现GatewayFilter和Ordered接口，代码如下：

```java
public class WhiteListGatewayFilter implements GatewayFilter, Ordered {
	//白名单集合
    private List<String> whiteList;
	//通过构造器初始化白名单
    WhiteListGatewayFilter(List<String> whiteList) {
        this.whiteList = whiteList;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String userName = exchange.getRequest().getQueryParams().getFirst("userName");
        //白名单不为空，并且userName包含在白名单内，才可以访问
        if (!CollectionUtils.isEmpty(whiteList) && whiteList.contains(userName)) {
            return chain.filter(exchange);
        }
        //如果白名单为空或者userName不在白名单内，则返回401
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }

    //优先级，值越小优先级越高
    @Override
    public int getOrder() {
        return 0;
    }
}
```

接着再定义一个过滤器工厂，注入到Spring容器中，代码如下：

```java
@Component
public class WhiteListGatewayFilterFactory extends AbstractConfigurable<WhiteListGatewayFilterFactory.Config> implements GatewayFilterFactory<WhiteListGatewayFilterFactory.Config> {

    private static final String VALUE = "value";

    protected WhiteListGatewayFilterFactory() {
        super(WhiteListGatewayFilterFactory.Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Collections.singletonList(VALUE);
    }

    @Override
    public GatewayFilter apply(Config config) {
        //获取配置的白名单
        String whiteString = config.getValue();
        List<String> whiteList = new ArrayList<>(Arrays.asList(whiteString.split(",")));
        //创建WhiteListGatewayFilter实例，返回
        return new WhiteListGatewayFilter(whiteList);
    }
	//用于接收配置参数
    public static class Config {

        private String value;

        public String getValue() {
            return value;
        }

        public void setValue(String value) {
            this.value = value;
        }
    }
}
```

最后，我们可以在application.yaml配置文件中加上配置使用：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: white_list_filter
          uri: http://localhost:8081
          predicates:
            - Method=GET
          filters:
            - WhiteList=yehongzhi #等号后面配置的是白名单，用逗号隔开
```

接着启动项目，先请求`localhost:9201/getDetail/1`，不带userName，按预期会返回401，不能访问。

![](https://static.lovebilibili.com/gateway_filter_09.png)

请求带有`userName=yehongzhi`的地址`http://localhost:9201/getDetail/1?userName=yehongzhi`，是在白名单内的，所以能正常访问。

![](https://static.lovebilibili.com/gateway_filter_10.png)

## 全局过滤器

全局过滤器在系统初始化时就作用于所有的路由，不需要单独去配置。全局过滤器的接口定义类是`GlobalFilter`，Gateway本身也有很多内置的过滤器，我们打开类图看看：

![](https://static.lovebilibili.com/gateway_filter_11.png)

我们拿几个比较有代表性的来做介绍，比如负载均衡的全局过滤器`LoadBalancerClientFilter`。

### LoadBalancerClientFilter

该过滤器会解析到以`lb://`开头的uri，比如这样的配置：

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        service: ${spring.application.name}
    gateway:
      routes:
        - id: consumer
          uri: lb://consumer #使用lb协议，consumer是服务名，不再使用IP地址配置
          order: 1
          predicates:
            - Path=/consumer/** 
```

这个全局过滤器就会取到consumer这个服务名，然后通过`LoadBalancerClient`获取到`ServiceInstance`服务实例。根据获取到的服务实例，重新组装请求的url。

这就是一个全局过滤器应用的例子，它是作用于全局，而且并不需要配置。下面我们探索一下自定义全局过滤器，假设需要统计用户的IP地址访问网关的总次数，怎么做呢？

### 自定义全局过滤器

自定义全局过滤器需要实现`GlobalFilter`接口和`Ordered`接口。

```java
@Component
public class IPAddressStatisticsFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        InetSocketAddress host = exchange.getRequest().getHeaders().getHost();
        if (host == null || host.getHostName() == null) {
            exchange.getResponse().setStatusCode(HttpStatus.BAD_REQUEST);
            return exchange.getResponse().setComplete();
        }
        String hostName = host.getHostName();
        AtomicInteger count = IpCache.CACHE.getOrDefault(hostName, new AtomicInteger(0));
        count.incrementAndGet();
        IpCache.CACHE.put(hostName, count);
        System.out.println("IP地址：" + hostName + ",访问次数：" + count.intValue());
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 10101;
    }
}

//用于保存次数的缓存
public class IpCache {
    public static final Map<String, AtomicInteger> CACHE = new ConcurrentHashMap<>();
}
```

启动项目，然后请求服务，可以看到控制台打印结果。

```java
IP地址：192.168.1.4,访问次数：1
IP地址：192.168.1.4,访问次数：2
IP地址：192.168.1.4,访问次数：3
IP地址：localhost,访问次数：1
IP地址：localhost,访问次数：2
IP地址：localhost,访问次数：3
IP地址：192.168.1.4,访问次数：4
```

# 总结

通过上一篇的Predicates和这篇的Filters基本上把服务网关的功能都实现了，包括路由转发、权限拦截、流量统计、流量控制、服务熔断、日志记录等等。所以网关对于微服务架构来说，网关服务是一个非常重要的部分，有很多一线的互联网公司还会自研服务网关。因此掌握服务网关对于后端开发可以说是必备技能，感谢大家的阅读。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！