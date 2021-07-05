# 前言

mybatis相信都不陌生，目前互联网公司大部分都使用mybatis作为持久层框架，无他，因为可以直接在xml文件中编写SQL语句操作数据库，灵活。但是我们在使用的时候，也会发现有很多增删改查的SQL是每个表都会有的基本操作，如果每个表都写一套增删改查的SQL显然是非常耗时耗力的。

于是乎，就有了mybatis-plus这个框架。正如官网所说，mybatis-plus是`为简化开发而生`。

mybatis-plus有以下特点：

- 只做增强不做改变，引入它不会对现有工程产生影响，如丝般顺滑。
- 只需简单配置，即可快速进行单表CRUD操作，节省大量时间。
- 代码生成，物理分页，性能分析等功能一应俱全。

# 一、整合mybatis-plus

这里用的是SpringBoot2.5.2做演示。首先导入依赖：

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
<!-- 引入 mybatis-plus -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.3.0</version>
</dependency>
```

然后在application.properties文件配置数据库信息：

```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/user?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=root123456
#mapper.xml文件路径地址
mybatis-plus.mapper-locations=classpath:mapper/*Mapper.xml
```

在启动类加上扫描注解：

```java
@SpringBootApplication
@MapperScan(basePackages = "com.yehongzhi.mydemo.mapper")
public class MydemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(MydemoApplication.class, args);
    }
}
```

其实这样就完成了，但是我们要建个表进行测试。

```sql
CREATE TABLE `user` (
  `id` char(36) NOT NULL DEFAULT '' COMMENT 'ID',
  `name` varchar(255) DEFAULT '' COMMENT '姓名',
  `age` int(3) DEFAULT NULL COMMENT '年龄',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
-- 初始化4条数据
INSERT INTO `user`.`user` (`id`, `name`, `age`) VALUES ('1345fc0985b111eba0e488d7f66fdab8', '观辰', '20');
INSERT INTO `user`.`user` (`id`, `name`, `age`) VALUES ('d47561e885b011eba0e488d7f66fdab8', '姚大秋', '30');
INSERT INTO `user`.`user` (`id`, `name`, `age`) VALUES ('ef2741fe87f011eba0e488d7f66fdab8', '周星驰', '60');
INSERT INTO `user`.`user` (`id`, `name`, `age`) VALUES ('ff784f6b85b011eba0e488d7f66fdab8', '李嘉晟', '33');
```

建了表之后，再创建一个User实体类对应：

```java
//表名
@TableName("user")
public class User {
	//主键
    @TableId(type = IdType.UUID)
    private String id;
	//姓名
    private String name;
	//年龄
    private Integer age;
    //getter、setter方法
}
```

接着创建UserMapper接口类,，然后继承BaseMapper：

```java
@Repository
public interface UserMapper extends BaseMapper<User> {
    
}
```

创建一个UserMapper.xml与UserMapper对应：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.yehongzhi.mydemo.mapper.UserMapper">

</mapper>
```

最后我们写一个UserService接口，查询user表：

```java
@Service
public class UserServiceImpl implements UserService {
    @Resource
    private UserMapper userMapper;

    @Override
    public List<User> getList() {
        return userMapper.selectList(null);
    }
}
```

启动项目，测试是没问题的：

![](https://static.lovebilibili.com/mybatis-plus_01.png)

# 二、CRUD操作

整合完了之后，按照mybatis-plus的官方说明，是有简单的单表CRUD操作功能。

这些单表的CRUD操作其实都放在BaseMapper里面了，所以当我们继承了BaseMapper类之后，就会获得mybatis-plus的增强特性，其中就包括单表的CRUD操作。

## 1、insert操作

BaseMapper直接提供一个insert()方法，传一个实体类。

```java
@Service
public class UserServiceImpl implements UserService {
    @Resource
    private UserMapper userMapper;
    
    @Override
    public int insertOne(User user) {
        return userMapper.insert(user);
    }
}
```

接着在Controller调用，因为在User类上面贴了注解`@TableId`，所以会自动生成ID。

```java
@TableName("user")
public class User {
    @TableId(type = IdType.UUID)
    private String id;
    //省略...
}
```

Controller层代码：

```java
@RequestMapping("/insert")
public String insertUser(@RequestParam(name = "name") String name,
                         @RequestParam(name = "age") Integer age) {
    User user = new User();
    user.setName(name);
    user.setAge(age);
    //在实体类使用了@TableId注解，ID会自动生成
    int i = userService.insertOne(user);
    return i == 1 ? "success" : "fail";
}
```

## 2、update操作

BaseMapper直接提供一个updateById()方法，传一个实体类。

```java
@Override
public int updateOne(User user) {
    return userMapper.updateById(user);
}
```

Controller层代码：

```java
@RequestMapping("/update")
public String updateUser(@RequestParam(name = "id") String id,
                         @RequestParam(name = "name",required = false) String name,
                         @RequestParam(name = "age",required = false) Integer age) {
    User user = new User();
    user.setId(id);
    user.setName(name);
    user.setAge(age);
    int i = userService.updateOne(user);
    return i == 1 ? "success" : "fail";
}
```

## 3、delete操作

BaseMapper直接提供一个deleteById()方法，传主键值。

```java
@Override
public int deleteOne(String id) {
    return userMapper.deleteById(id);
}
```

Controller层代码：

```java
@RequestMapping("/delete")
public String deleteUser(@RequestParam(name = "id") String id) {
    int i = userService.deleteOne(id);
    return i == 1 ? "success" : "fail";
}
```

除此之外，还有批量删除deleteBatchIds方法，传主键值的集合：

```java
@Override
public int deleteBatch(List<String> ids){
    return userMapper.deleteBatchIds(ids);
}
```

Controller层代码：

```java
@RequestMapping("/deleteBatch")
public String deleteBatchUser(@RequestParam(name = "ids") String ids) {
    List<String> list = Arrays.asList(ids.split(","));
    int i = userService.deleteBatch(list);
    return i == list.size() ? "success" : "fail";
}
```

# 三、条件构造器(Wrapper)

条件构造器在Mybatis-plus各种方法中都有出现，比如：

```java
Integer selectCount(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

List<T> selectList(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

int update(@Param(Constants.ENTITY) T entity, @Param(Constants.WRAPPER) Wrapper<T> updateWrapper);

int delete(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
```

Wrapper通俗点理解就是定义where语句后面的查询条件，是Mybatis-Plus里功能比较强大的工具。Wrapper是一个抽象类，下面有很多子类，我们先看个类图混个眼熟。

![](https://static.lovebilibili.com/mybatis-plus_02.png)

常用的子类实现有四个，分别是：

- QueryWrapper
- UpdateWrapper
- LambdaQueryWrapper
- LambdaUpdateWrapper

## QueryWrapper

主要用于生成where条件，举个例子，我们用name查询user表：

```java
public List<User> queryUserByName(String name) {
    //相当于：SELECT * FROM user WHERE name = ?
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    //eq()表示相等
    queryWrapper.eq("name", name);
    return userMapper.selectList(queryWrapper);
}
```

我们看日志打印：

```java
==>  Preparing: SELECT id,name,age FROM user WHERE name = ?
==> Parameters: 姚大秋(String)
<==    Columns: id, name, age
<==        Row: d47561e885b011eba0e488d7f66fdab8, 姚大秋, 30
<==      Total: 1
```

假如我们要like查询，可以这样写：

```java
public List<User> queryUserLikeName(String name) {
    //相当于：SELECT * FROM user WHERE name like %#{name}%
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.like("name",name);
    return userMapper.selectList(queryWrapper);
}
```

假如要查询年龄大于30岁，可以这样：

```java
public List<User> queryUserGtByAge(int age) {
    //相当于：SELECT * FROM user WHERE age > ?
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.gt("age", age);
    //小于是lt()
    //大于等于是ge()
    //小于等于是le()
    //范围的话，则使用between()
    return userMapper.selectList(queryWrapper);
}
```

如果要查询某个字段不为空，可以这样：

```java
public List<User> queryUserByNameNotNull() {
    //相当于：SELECT * FROM user WHERE name IS NOT NULL
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.isNotNull("name");
    //查询某个字段为空，则使用isNull()
    return userMapper.selectList(queryWrapper);
}
```

如果使用IN查询，可以这样：

```java
public List<User> queryUserByIds(List<String> ids) {
    //相当于：SELECT * FROM user WHERE name IN ('id1','id2');
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.in("id", ids);
    //相反也提供了notIn()方法
    return userMapper.selectList(queryWrapper);
}
```

如果需要排序，可以这样写：

```java
public List<User> queryUserOrderByAge() {
    //相当于：SELECT * FROM user ORDER BY age ASC
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.orderByAsc("age");
    //相反的，如果降序则使用orderByDesc()方法
    return userMapper.selectList(queryWrapper);
}
```

如果需要子查询，可以这样写：

```java
public List<User> queryUserInSql() {
    //相当于：SELECT * FROM user WHERE id IN (SELECT id FROM user WHERE age > 30)
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.inSql("id","select id from user where age > 30");
    return userMapper.selectList(queryWrapper);
}
```

大部分的方法都是源自AbstractWrapper抽象类，除此之外还有很多功能，这里就不一一介绍下去了，有兴趣的可以到官网的[条件构造器](https://mp.baomidou.com/guide/wrapper.html)慢慢探索。

## UpdateWrapper

UpdateWrapper也是AbstractWrapper抽象类的子类实现，所以上述的设置条件的方法都有，不同的是，UpdateWrapper会有set()和setSql()设置更新的值。举个例子：

```java
public int updateUserNameById(String id, String name) {
    //相当于：UPDATE user SET name = ? where id = ?
    UpdateWrapper<User> userUpdateWrapper = new UpdateWrapper<>();
    userUpdateWrapper.eq("id", id);
    //设置set关键字后面的语句，相当于set name = #{name}
    userUpdateWrapper.set("name", name);
    return userMapper.update(new User(), userUpdateWrapper);
}
```

setSql()就是设置SET关键字后面拼接的部分SQL语句，举个例子：

```java
public int updateUserBySql(String id, String name) {
    //相当于：UPDATE user SET name = ? where id = ?
    UpdateWrapper<User> userUpdateWrapper = new UpdateWrapper<>();
    userUpdateWrapper.setSql("name = '" + name + "'");
    userUpdateWrapper.eq("id", id);
    return userMapper.update(new User(), userUpdateWrapper);
}
```

setSql()就是纯粹的SQL语句拼接，我们可以看到SET后面接的是name='大D'，而不是占位符会有SQL注入的风险。

```java
==>  Preparing: UPDATE user SET name = '大D' WHERE id = ?
==> Parameters: d47561e885b011eba0e488d7f66fdab8(String)
<==    Updates: 1
```

## LambdaQueryWrapper

用LambdaQueryWrapper的好处在于消除硬编码，比如QueryWrapper在查询`name=?`时，需要这样写：

```java
QueryWrapper<User> queryWrapper = new QueryWrapper<>();
//使用"name"字符串，就是硬编码
queryWrapper.eq("name", name);
```

如果换成LambdaQueryWrapper就可以这样写：

```java
public List<User> lambdaQueryUserByName(String name) {
    //相当于：SELECT id,name,age FROM user WHERE name = ?
    LambdaQueryWrapper<User> lambdaQueryWrapper = new QueryWrapper<User>().lambda();
    lambdaQueryWrapper.eq(User::getName, name);
    return userMapper.selectList(lambdaQueryWrapper);
}
```

再比如使用模糊查询，可以这样写：

```java
public List<User> lambdaQueryUserLikeName(String name) {
    LambdaQueryWrapper<User> lambdaQueryWrapper = new QueryWrapper<User>().lambda();
    lambdaQueryWrapper.like(User::getName, name);
    return userMapper.selectList(lambdaQueryWrapper);
}
```

## LambdaUpdateWrapper

跟上面差不多，LambdaUpdateWrapper也可以消除硬编码：

```java
public int lambdaUpdateUserNameById(String id, String name) {
    //相当于：UPDATE user SET name=? WHERE id = ?
    LambdaUpdateWrapper<User> lambdaUpdateWrapper = new UpdateWrapper<User>().lambda();
    lambdaUpdateWrapper.set(User::getName, name);
    lambdaUpdateWrapper.eq(User::getId, id);
    return userMapper.update(new User(), lambdaUpdateWrapper);
}
```

# 分页查询

Mybatis-plus提供了分页插件支持分页查询，只需要几个步骤即可实现。

首先添加一个配置类：

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        return new PaginationInterceptor();
    }
}
```

第二步，在UserMapper类里定义分页查询方法：

```java
@Repository
public interface UserMapper extends BaseMapper<User> {
	//分页查询方法
    IPage<User> selectPageByName(Page<User> page, @Param("name") String name);
}
```

对应的UserMapper.xml方法：

```xml
<select id="selectPageByName" parameterType="java.lang.String" resultType="com.yehongzhi.mydemo.model.User">
    select * from `user` where name = #{name}
</select>
```

第三步，在userService里调用：

```java
public IPage<User> selectPageByName(long pageNo, long pageSize, String name) {
    Page<User> page = new Page<>();
    //设置当前页码
    page.setCurrent(pageNo);
    //设置每页显示数
    page.setSize(pageSize);
    return userMapper.selectPageByName(page, name);
}
```

最后写个Controller接口测试：

```java
@RequestMapping("/queryPage/ByName")
public IPage<User> selectPageByName(@RequestParam("pageNo") long pageNo,
                                    @RequestParam("pageSize") long pageSize,
                                    @RequestParam("name") String name) {
    return userService.selectPageByName(pageNo, pageSize, name);
}
```

查看控制台日志：

```java
==>  Preparing: SELECT COUNT(1) FROM `user` WHERE name = ?
==> Parameters: 杜琪峰(String)
<==    Columns: COUNT(1)
<==        Row: 1
==>  Preparing: select * from `user` where name = ? LIMIT ?,?
==> Parameters: 杜琪峰(String), 0(Long), 10(Long)
<==    Columns: id, name, age
<==        Row: d47561e885b011eba0e488d7f66fdab8, 杜琪峰, 30
<==      Total: 1
```

分页查询成功！

# 自定义主键生成器

有时在实际开发中，可能会遇到，Mybatis-plus提供的主键生成策略并不能满足，需要自定义主键ID生成策略，怎么设置呢？

很简单，根据官网的说明，我们先定义一个主键生成器：

![](https://static.lovebilibili.com/mybatis-plus_03.png)

```java
@Component
public class SnowflakeKeyGenerator implements IdentifierGenerator {
	//自己实现的一个雪花ID生成工具类
    @Resource
    private SnowflakeIdWorker snowflakeIdWorker;

    @Override
    public Number nextId(Object entity) {
        //使用雪花ID生成器，生成一个雪花ID
        long nextId = snowflakeIdWorker.nextId();
        System.out.println(String.format("使用自定义ID生成器，生成雪花ID:%s", nextId));
        return nextId;
    }
}
```

然后我们在需要使用该自定义ID生成器的实体类上面加上注解属性：

```java
@TableName("user")
public class User {
	//属性设置为：IdType.ASSIGN_ID
    @TableId(type = IdType.ASSIGN_ID)
    private String id;
	
    //省略...
}
```

接着测试一下，我们可以看到控制台有打印日志：

![](https://static.lovebilibili.com/mybatis-plus_04.png)

# 总结

除了上面介绍的功能之外，Mybatis-plus还有很多功能，比如：代码生成器、扩展等等。这里就不再一一介绍了，实际上我们掌握上面介绍的CRUD、条件构造器、分页查询、自定义主键策略，基本上已经足够日常的开发。

当然如果你觉得会用还不够，还想要看懂框架的源码实现，那没问题。下一篇文章，我就讲讲框架的源码分析，敬请期待，

非常感谢你的阅读，希望这篇文章能给到你帮助和启发。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！