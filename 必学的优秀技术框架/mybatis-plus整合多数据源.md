> **文章已收录到我的Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 提出问题

在平时开发中，经常会遇到在一个项目里需要使用多个数据源的情况，比如有一部分数据在数据源A，另一部分数据在数据源B，业务需要把这两部分的数据做合并然后从接口返回。

又或者操作完数据源A后，需要切换数据源，操作数据源B。这样的需求，怎么实现？

# 解决问题

其实在mybatis-plus就有相关的实现，是一个基于SpringBoot快速集成多数据源的启动器。

首先要搭建一个springBoot+Mybatis+Mybatis-Plus的项目，搭建项目就不演示了，比较简单。这里讲怎么使用多数据源，首先引入`dynamic-datasource-spring-boot-starter`。

```xml
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
  <version>${version}</version><!--版本号-->
</dependency>
```

第二步，修改application配置文件，配置数据源。

```properties
# 默认数据源
spring.datasource.dynamic.primary=master
# 数据源A
spring.datasource.dynamic.datasource.master.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.dynamic.datasource.master.url=jdbc:mysql://192.168.0.101:3306/user?createDatabaseIfNotExist=true
spring.datasource.dynamic.datasource.master.username=root
spring.datasource.dynamic.datasource.master.password=
# 数据源B
spring.datasource.dynamic.datasource.slave.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.dynamic.datasource.slave.url=jdbc:mysql://192.168.0.102:3306/user?createDatabaseIfNotExist=true
spring.datasource.dynamic.datasource.slave.username=root
spring.datasource.dynamic.datasource.slave.password=
```

第三步，使用@DS注解切换数据源。一般我喜欢放在Mapper接口上面。注解的值则是数据源的名称。

现在有两个数据源，如图所示：

![](https://static.lovebilibili.com/dy_data_source_01.png)

slave数据源的user表对应的Mapper接口，如下：

```java
@Mapper
@DS("slave")
public interface UserMapper extends BaseMapper<User> {
}
```

master数据源的commodity表对应的Mapper接口，如下：

```java
@Mapper
@DS("master")
public interface CommodityMapper extends BaseMapper<Commodity> {
}
```

接着我们写一个Service进行测试：

```java
@Service
public class DynamicServiceImpl implements DynamicService {

    @Resource
    private UserMapper userMapper;

    @Resource
    private CommodityMapper commodityMapper;

    @Override
    public ResultObject getUserListAndCommodityList() {
        //查询slave数据源
        List<User> userList = userMapper.selectList(null);
        //查询master数据源
        List<Commodity> commodityList = commodityMapper.selectList(null);
        //把两个数据源查询的整合到一起返回
        ResultObject resultObject = new ResultObject();
        resultObject.setUserList(userList);
        resultObject.setCommodityList(commodityList);
        return resultObject;
    }
}
```

Controller接口，如下：

```java
@RestController
@RequestMapping("/dynamic")
public class DynamicController {

    @Resource
    private DynamicService dynamicService;

    @RequestMapping("/getUserListAndCommodityList")
    public ResultObject getUserListAndCommodityList() {
        return dynamicService.getUserListAndCommodityList();
    }
}
```

启动项目，然后请求接口地址，我们可以看到结果里包含了两个数据源查询到的数据：

![](https://static.lovebilibili.com/dy_data_source_02.png)

# 一些疑问

**问题一：@DS注解可以在什么地方使用？**

看源码，可以知道是在类和方法上使用。

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DS {
    /**
     * groupName or specific database name or spring SPEL name.
     *
     * @return the database you want to switch
     */
    String value();
}
```

一般在Service或者Mapper上面进行标记。在方法上面标记优先级高于在类上标记。

**问题二：支持多层嵌套切换数据源吗？**

什么是嵌套切换数据源，先看个例子：

```java
@Service
@DS("slave")
public class UserServiceImpl implements UserService {

    @Resource
    private UserMapper userMapper;

    @Override
    public List<User> getList() {
        return userMapper.selectList(null);
    }
}
```

```java
@Service
@DS("master")
public class CommodityServiceImpl implements CommodityService {
    
    @Resource
    private UserService userService;
    
    @Override
    public ResultObject getUserAndCommodity() {
        //切换到slave数据源，查询user表
        List<User> userList = userService.getList();
        //切换回master数据源，查询commodity表
        List<Commodity> commodityList = commodityMapper.selectList(null);
        ResultObject resultObject = new ResultObject();
        resultObject.setUserList(userList);
        resultObject.setCommodityList(commodityList);
        return resultObject;
    }
}
```

```java
@RestController
@RequestMapping("/dynamic")
public class DynamicController {

    @Resource
    private CommodityService commodityService;

    @RequestMapping("/getUserListAndCommodityList")
    public ResultObject getUserListAndCommodityList() {
        return commodityService.getUserAndCommodity();
    }
}
```

启动项目，然后请求地址，发现依然能查出结果，证明是**支持嵌套切换数据源的**。

**问题三：如何使用@Transactional注解提供事务支持？**

我们看个例子：
首先定义一个插入对象InsertRequest：

```java
public class InsertRequest {

    private User user;

    private Commodity commodity;
}
```

然后在Service类加上@Transactional注解，并把数据分别插入两个数据源的表中：

```java
@Service
@Transactional
public class DynamicServiceImpl implements DynamicService {

    @Resource
    private UserMapper userMapper;

    @Resource
    private CommodityMapper commodityMapper;
    
    
    @Override
    public void insertUserAndCommodity(InsertRequest insertRequest) {
        commodityMapper.insert(insertRequest.getCommodity());
        userMapper.insert(insertRequest.getUser());
    }
}
```

然后在Controller接口层增加一个入口方法：

```java
@RequestMapping("/insertUserAndCommodity")
public String insertUserAndCommodity(@RequestParam(name = "name") String name,
                                     @RequestParam(name = "age") Integer age,
                                     @RequestParam("commodityName") String commodityName,
                                     @RequestParam("commodityPrice") String commodityPrice) {
    //构建user对象
    User user = new User();
    user.setName(name);
    user.setAge(age);
    //构建commodity对象
    Commodity commodity = new Commodity();
    commodity.setCommodityName(commodityName);
    commodity.setCommodityPrice(commodityPrice);
    //构建insertRequest对象
    InsertRequest insertRequest = new InsertRequest();
    insertRequest.setUser(user);
    insertRequest.setCommodity(commodity);
    //调用插入方法
    dynamicService.insertUserAndCommodity(insertRequest);
    return "success";
}
```

启动项目，测试，结果是500错误：

![](https://static.lovebilibili.com/dy_data_source_03.png)

我们看控制台日志可以发现问题：

![](https://static.lovebilibili.com/dy_data_source_04.png)

学过Spring都知道@Transactional是本地事务管理，不是分布式事务，所以我们不能在需要切换数据源的方法上加@Transactional注解。

不使用分布式事务的情况下，我们只能让其各自管理各自的事务，中间加多一层Service。

```java
@Service
@DS("master")
public class CommodityServiceImpl implements CommodityService {
    
    @Resource
    private CommodityMapper commodityMapper;
    
    @Override
    @Transactional
    public int insertOne(Commodity commodity) {
        return commodityMapper.insert(commodity);
    }
}

@Service
@DS("slave")
public class UserServiceImpl implements UserService {

    @Resource
    private UserMapper userMapper;
    
    @Override
    @Transactional
    public int insertOne(User user) {
        return userMapper.insert(user);
    }
}
```

```java
@Service
public class DynamicServiceImpl implements DynamicService {
    @Resource
    private UserService userService;

    @Resource
    private CommodityService commodityService;
    
    @Override
    public void insertUserAndCommodity(InsertRequest insertRequest) {
        commodityService.insertOne(insertRequest.getCommodity());
        userService.insertOne(insertRequest.getUser());
    }
}
```

这样改完之后，我们再测一下，可以看到数据源切换成功了。

![](https://static.lovebilibili.com/dy_data_source_05.png)

如果需要使用分布式事务，官网也说明提供基于seata的分布式事务方案，这里就不演示了，读者有兴趣可以自己尝试一下。

**问题四：同一个Service里面，定义了两个不同的方法对应不同的数据源，能互相调用吗？**

我们还是以事实说话，先整个例子：

```java
@Override
@DS("slave")
public ResultObject getResultObject() {
    List<User> userList = userService.getList();
    //调用同一个类的另一个方法切换数据源
    List<Commodity> commodityList = getCommodityList();
    ResultObject resultObject = new ResultObject();
    resultObject.setUserList(userList);
    resultObject.setCommodityList(commodityList);
    return resultObject;
}

@Override
@DS("master")
public List<Commodity> getCommodityList() {
    return commodityMapper.selectList(null);
}
```

启动项目，测试，发现报错了：

![](https://static.lovebilibili.com/dy_data_source_06.png)

原因是数据源切换的底层原理是使用AOP实现的，如果是内部方法调用是不会使用AOP的，所以也就没法切换数据源。

解决的方法是把getCommodityList()方法提到另一个类里面去。

# 最后

dynamic-datasource其实还有很多功能没介绍，这里就不一一介绍了，有兴趣的可以到官网上去学习。多数据源在项目开发中是很常见的，所以我觉得学习这个插件还是很有用的。后面如果有时间的话，我会解读一下dynamic-datasource源码，探索一下底层的实现原理。

非常感谢你的阅读，希望这篇文章能给到你帮助和启发。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！