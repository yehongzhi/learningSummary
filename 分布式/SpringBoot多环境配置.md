> **文章已收录到我的Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 前言

一般来说，在日常开发中都会分多个环境，比如git代码分支会分为dev(开发)、release(测试)、pord(生产)等多个环境。可以说每个环境对应的配置信息(比如数据库、缓存、消息队列MQ等)都不相同。因此不同的环境肯定需要对应不同的配置文件。接下来学习一下怎么配置多环境的配置文件。

# SpringBoot多环境配置

因为SpringBoot做多环境配置比较简单，而且现在大部分项目基本都会使用SpringBoot，所以这里就介绍怎么用SpringBoot做多环境配置。

## 单文件版本

单文件在实际中使用得并不多，不过也可以实现多环境配置，这里简单介绍一下。以`application.yml`配置文件举例，你要在一个配置文件里面配置多个环境的配置，肯定需要分割线将其隔开，所以SpringBoot就规定了使用`---`进行隔开每个环境。

```yaml
spring:
  application:
    name: mydemo
  profiles:
    active: prod # 选择prod环境配置
#整合mybatis
mybatis-plus:
  mapper-locations: classpath:mapper/*Mapper.xml
  type-aliases-package: com.yehongzhi.mydemo.model
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
---
# 开发环境
server:
  port: 8080
spring:
  profiles: dev
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://DEV_IP:3306/user?createDatabaseIfNotExist=true
    username: root
    password: 123456
---
# 测试环境
server:
  port: 8090
spring:
  profiles: release
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://RELEASE_IP:3306/user?createDatabaseIfNotExist=true
    username: root
    password: 123456
---
# 生产环境
server:
  port: 8888
spring:
  profiles: prod
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://PROD_IP:3306/user?createDatabaseIfNotExist=true
    username: root
    password: 123456
```

单文件配置多环境的缺点很明显，就是会导致这个`application.yml`文件非常大，不够清晰。最好是一个环境单独一个文件，这样就清晰很多。于是乎就有了多文件版本。

## 多文件版本

一般SpringBoot的配置文件都是叫`application.yml`或者`application.properties`，这里用`application.yml`举例，配置多环境配置文件，文件名需要满足这样的格式：`application-{profile}.yml`。看下图就明白了。

![](https://static.lovebilibili.com/springBoot_application_1.png)

换而言之，dev环境的配置文件就叫做`application-dev.yml`，那么怎么选择哪个环境的配置文件呢，其实很简单，只需要在`application.yml`加上如下配置：

```yaml
spring:
  profiles:
    active: dev
```

这就表示选择加载`application-dev.yml`文件，何以见得？

一般在启动完成之后，我们可以在控制台搜索关键字`profiles`找到对应的环境。

![](https://static.lovebilibili.com/springBoot_application_2.png)

所以我们就可以在application.yml里面，通过`spring.profiles.active`切换不同的环境。这就是多文件版本。

但是我们在平时开发时发现，这个配置要经常改来改去，非常麻烦，有没有不用改这个配置就可以切换的方法呢？当然有。

首先在`pom.xml`文件增加以下环境变量的配置。

```xml
<profiles>
    <profile><!-- 开发环境 -->
        <id>dev</id>
        <properties>
            <profiles.active>dev</profiles.active>
        </properties>
    </profile>
    <profile><!-- 测试环境 -->
        <id>release</id>
        <properties>
            <profiles.active>release</profiles.active>
        </properties>
    </profile>
    <profile><!-- 生产环境 -->
        <id>prod</id>
        <properties>
            <profiles.active>prod</profiles.active>
        </properties>
    </profile>
</profiles>
```

接着在`application.yml`配置文件中使用`@profiles.active@`来配置环境变量。

```yaml
spring:
  profiles:
    active: '@profiles.active@'
```

接着刷新Maven，可以在IDEA右侧中选择对应的环境，如下图：

![](https://static.lovebilibili.com/springBoot_application_3.png)

当需要切换环境时，就不需要改配置文件的内容，只需要勾选对应的环境即可，就方便很多。

## 结合Nacos配置中心

一般在项目开发中，都需要配置信息能够在运行时更改配置，于是乎就有了配置中心的概念。配置中心当然也有多环境的配置。

在Nacos配置中心就有命名空间的概念，我们可以使用命名空间来实现多环境配置。首先引入Maven依赖：

```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        <version>2.0.2.RELEASE</version>
    </dependency>
</dependencies>
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <profiles.active>dev</profiles.active>
        </properties>
    </profile>
    <profile>
        <id>release</id>
        <properties>
            <profiles.active>release</profiles.active>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <profiles.active>prod</profiles.active>
        </properties>
    </profile>
</profiles>
```

第二步，启动Nacos，然后在创建对应的命名空间和配置文件。

![](https://static.lovebilibili.com/springBoot_application_4.png)

![](https://static.lovebilibili.com/springBoot_application_5.png)

第三步，在项目中增加`bootstrap.yml`文件。

```yaml
spring:
  application:
    name: mydemo
  profiles:
    active: '@profiles.active@'
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        group: DEFAULT_GROUP
        namespace: a4a33d52-371b-451a-a3c1-d01c1d343331 #dev命名空间的ID
        enabled: true
        prefix: ${spring.application.name}
        refresh-enabled: true
```

在IDEA配置项目启动时设置环境变量。

![](https://static.lovebilibili.com/springBoot_application_6.png)

这样就完成了，启动项目，就可以读到Nacos配置中心的dev命名空间的`mydemo-dev.yaml`文件。

因为DataId的定义规则是`${prefix}-${spring.profiles.active}.${file-extension}`。

> prefix默认规则是获取${spring.application.name}的值。可以通过spring.cloud.nacos.config.prefix进行配置。
>
> spring.profiles.active即为当前环境对应的profile。可以通过spring.profiles.active进行配置。
>
> file-extension为配置文件的数据格式。可以通过spring.cloud.nacos.config.file-extension进行配置。

# 总结

以上就是多环境配置的三种方式，多环境配置基本上是创建新项目的基本操作，所以掌握多环境配置还是很有必要的。感谢大家的阅读，希望看完之后能对你有所收获。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！