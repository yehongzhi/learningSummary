# 思维导图

![](https://static.lovebilibili.com/zczx_swdt.png)

> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 一、前言

伴随着Eurka2.0版本已停止维护，开始要考虑使用微服务新一代的开源的注册中心替代Eureka。

![](https://static.lovebilibili.com/zczx_27.png)

目前据我了解，Consul和Nacos是比较流行的两种替代方案。这篇文章就介绍一下这两种注册中心在微服务中的简单使用，希望对读者有所帮助。

# 二、注册中心的作用

注册中心在微服务的架构中相当于一个“服务的通讯录”。当一个服务启动时，需要向注册中心注册服务，注册中心保存了所有服务的服务名称和服务地址的映射关系。当服务A想调用服务D时，则从注册中心获取服务D的服务地址，然后调用。

我画张图给大家描述会更清楚一点，大概如下：

![](https://static.lovebilibili.com/zczx_1.png)

可能会有人问，为什么不直接通过服务地址调用服务D呢，还要从注册中心去获取服务D的服务地址。因为一个服务背后是不止一台机器的，比如服务D可能在实际生产中是由三台机器支持的，对外只暴露一个服务名称，这样可以避免写死服务的IP地址在代码中(写在配置文件里)，在服务扩展时就非常方便了。

除了**服务注册**之外，注册中心还提供**服务订阅**，当有新的服务注册时，注册中心会实时推送到各个服务。

还有**服务健康监测**，可以在管理界面看到注册中心中的服务的状态。

# 三、Consul

由Go语言开发，支持多数据中心分布式高可用的服务发布和服务注册，采用ralt算法保证服务的一致性，且支持健康检查。

## 3.1 安装(win10版)

第一步，上官网下载安装包。

![](https://static.lovebilibili.com/zczx_2.png)

第二步，解压zip包，并配置环境变量。

![](https://static.lovebilibili.com/zczx_3.png)

![](https://static.lovebilibili.com/zczx_4.png)

第三步，唱跳rap篮球键ctrl+R，cmd，输入命令`consul`：

![](https://static.lovebilibili.com/zczx_5.png)

这就安装成功了，超简单！输入consul -version验证一下，会显示版本号：

![](https://static.lovebilibili.com/zczx_7.png)

第四步，启动。输入命令`consul.exe agent -dev`本地启动：

![](https://static.lovebilibili.com/zczx_8.png)

第五步，在浏览器中输入`http://localhost:8500`打开管理界面。

![](https://static.lovebilibili.com/zczx_9.png)

## 3.2 服务注册

接下来就需要创建两个服务，分别是订单(order)和用户(user)，注册到consul。下面我就演示其中一个user服务。

首先创建一个SpringBoot工程，Maven配置如下：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.1.RELEASE</version>
</parent>
<groupId>io.github.yehongzhi</groupId>
<artifactId>user</artifactId>
<version>0.0.1-SNAPSHOT</version>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency><!-- 健康监测的包 -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency><!-- spring-cloud-consul服务治理的jar包 -->
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        <version>2.0.1.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

然后yml配置文件如下：

```yaml
server:
  port: 8601
spring:
  application:
    name: user
  cloud:
    consul:
      port: 8500
      host: 127.0.0.1
      discovery:
        service-name: user
        instance-id: ${spring.application.name}:${spring.cloud.consul.host}:${server.port}
        health-check-path: /actuator/health
        health-check-interval: 10s
        prefer-ip-address: true
        heartbeat:
          enabled: true
```

在启动类加上开启服务注册的注解：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class UserApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class, args);
    }

}
```

最后启动项目即可，我这里启动两个user，端口号分别是8601，8602：

![](https://static.lovebilibili.com/zczx_10.png)

## 3.3 服务调用

再创建一个订单项目(order)，和user配置类似，注册服务到consul中。

![](https://static.lovebilibili.com/zczx_11.png)

下面演示一下用order服务调用user服务，首先定义user的接口：

```java
@RestController
@RequestMapping("/mall/user")
public class UserController {
    @RequestMapping("/list")
    public Map<String, Object> list() throws Exception {
        Map<String, Object> userMap = new HashMap<>();
        userMap.put("1号佳丽", "李嘉欣");
        userMap.put("2号佳丽", "袁咏仪");
        userMap.put("3号佳丽", "张敏");
        userMap.put("4号佳丽", "张曼玉");
        return userMap;
    }
}
```

接着在order服务调用user服务，使用RestTemplate的方式：

```java
@RestController
@RequestMapping("/mall/order")
public class OrderController {

    @Resource
    private LoadBalancerClient loadBalancerClient;

    @RequestMapping("/callUser")
    public String list() throws Exception {
        //从注册中心中获取user服务实例，包括服务的IP，端口号等信息
        ServiceInstance instance = loadBalancerClient.choose("user");
        //调用user服务
        String userList = new RestTemplate().getForObject(instance.getUri().toString() + "/mall/user/list", String.class);
        return "调用" + instance.getServiceId() + "服务，端口号：" + instance.getPort() + ",返回结果：" + userList;
    }
}
```

启动两个user服务，一个order服务，调用order的接口，可以看到结果：

![](https://static.lovebilibili.com/zczx_12.png)

![](https://static.lovebilibili.com/zczx_13.png)

负载均衡**默认是轮询访问**，所以交替调用8601和8602的user服务。

consul的简单入门就讲到这里了，除了**服务治理**之外，consul还可以用于做配置中心，读者有兴趣可以自己探索一下。**我这里用的是dev模式，相当于单机模式，仅用于学习，实际生产的话肯定是集群模式**，后面如果有时间我再专门写一篇演示一下consul集群的搭建。

下面讲另一款注册中心，阿里出品的Nacos。

# 四、Nacos

以下介绍来源于官网：

Nacos 致力于帮助您**发现、配置和管理微服务**。Nacos 提供了一组简单易用的特性集，帮助您快速实现**动态服务发现、服务配置、服务元数据及流量管理**。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

总结就是，Nacos提供三种功能：服务发现及管理、动态配置服务、动态DNS服务。

我这里主要讲服务发现，也就是作为**注册中心**的功能。

## 4.1 安装

首先[下载](https://github.com/alibaba/nacos/releases/tag/1.3.1)安装包，目前稳定版是1.3.1，推荐在Linux或者Mac系统上使用，我懒得开虚拟机，所以我就直接在win系统安装。

![](https://static.lovebilibili.com/zczx_14.png)

我这里仅用于学习，使用单机模式，官网上介绍，双击startup.cmd文件启动即可。

![](https://static.lovebilibili.com/zczx_15.png)

实际上，会报错。

![](https://static.lovebilibili.com/zczx_17.png)

这个错误，我发现github上有人提出来，再后面加个参数就可以了。

![](https://static.lovebilibili.com/zczx_18.png)

但是又有人说后面的版本已经优化了，没有这个错误。反正如果遇到的话，就加个参数启动吧。完整命令是`startup.cmd -m standalone`。

如果不想在启动命令后面加参数，可以配置mysql(版本要求：5.6.5+)，初始化mysql数据库，数据库初始化文件：nacos-mysql.sql。

![](https://static.lovebilibili.com/zczx_26.png)

修改conf/application.properties文件配置：

```properties
db.num=1
db.url.0=jdbc:mysql://数据库地址:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=账号
db.password=密码
```

启动成功，命令行窗口可以看到以下提示：

![](https://static.lovebilibili.com/zczx_19.png)

启动成功后，可以在浏览器打开`http://localhost:8848/nacos/`，进入管理界面。账号密码默认都是nacos。

![](https://static.lovebilibili.com/zczx_20.png)

![](https://static.lovebilibili.com/zczx_21.png)

## 4.2 服务注册

接下来还是一样，创建两个服务注册到nacos，为了跟前面的区分，项目名后缀加上"nacos"。首先添加maven配置，如下：

```java
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.5.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
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
    </dependencies>
</dependencyManagement>
<dependencies>
    <dependency><!-- SpringWeb依赖 -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency><!-- SpringCloud nacos服务发现的依赖 -->
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>
```

启动类加上注解@EnableDiscoveryClient。

```java
@SpringBootApplication
@EnableDiscoveryClient
public class UsernacosApplication {
    public static void main(String[] args) {
        SpringApplication.run(UsernacosApplication.class, args);
    }
}
```

配置文件application.properties文件加上配置。

```properties
server.port=8070
spring.application.name=usernacos
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
```

创建一个UserController接口，提供给其他微服务调用。

```java
@RestController
@RequestMapping("/mall/userNacos")
public class UserController {
    @RequestMapping("/list")
    public Map<String, Object> list() {
        Map<String, Object> userMap = new HashMap<>();
        userMap.put("周杰伦", "爱在西元前");
        userMap.put("张学友", "只想一生跟你走");
        userMap.put("刘德华", "忘情水");
        userMap.put("陈奕迅", "K歌之王");
        userMap.put("卫兰", "就算世界没有童话");
        return userMap;
    }
}
```

运行启动类的main方法，可以看到注册中心多了一个usernacos服务。

![](https://static.lovebilibili.com/zczx_22.png)

## 4.3 服务调用

相同的配置和方法，再创建一个ordernacos服务，作为消费者。

```java
@RestController
@RequestMapping("/mall/orderNacos")
public class OrderController {
    @Resource
    private LoadBalancerClient loadBalancerClient;

    @RequestMapping("/callUser")
    public String callUser() {
        ServiceInstance instance = loadBalancerClient.choose("usernacos");
        String url = instance.getUri().toString() + "/mall/userNacos/list";
        RestTemplate restTemplate = new RestTemplate();
        //调用usernacos服务
        String result = restTemplate.getForObject(url, String.class);
        return "调用" + instance.getServiceId() + "服务，端口号：" + instance.getPort() + ",返回结果：" + result;
    }
}
```

启动2个usernacos服务，1个ordernacos服务。

![](https://static.lovebilibili.com/zczx_23.png)

测试接口`http://localhost:8170/mall/orderNacos/callUser`，order能顺利调用user，默认负载均衡策略也是轮询机制。

![](https://static.lovebilibili.com/zczx_24.png)

![](https://static.lovebilibili.com/zczx_25.png)

# 五、总结

国内用的比较多的是Nacos，我觉得原因有几点：

- 因为阿里目前用的就是Nacos，经历过双十一，各种秒杀活动等高并发场景的验证。
- 文档比较齐全，关键有中文文档，对于国内很多英文水平不是很好的开发者看起来真的很爽。
- 很多从阿里出来的程序员，把阿里的技术带到了各个中小型互联网公司，一般技术选型肯定选自己熟悉的嘛。
- 管理界面有中(英)文版本，易于操作。
- 还有社区比较活跃，很多问题可以在网上找到解决方案。

这篇文章主要介绍了SpringCloud微服务关于注册中心的两种流行的实现方案，接下来还会继续介绍其他关于微服务的组件，敬请期待。

上面所有例子的代码都上传Github了：

> https://github.com/yehongzhi/mall

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！