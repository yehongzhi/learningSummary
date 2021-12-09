> **文章已收录到我的Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 起源

需要了解一门技术，首先从为什么产生开始说起是最好的。JWT主要用于用户登录鉴权，所以我们从最传统的session认证开始说起。

## session认证

众所周知，http协议本身是无状态的协议，那就意味着当有用户向系统使用账户名称和密码进行用户认证之后，下一次请求还要再一次用户认证才行。因为我们不能通过http协议知道是哪个用户发出的请求，所以如果要知道是哪个用户发出的请求，那就需要在服务器保存一份用户信息(保存至session)，然后在认证成功后返回cookie值传递给浏览器，那么用户在下一次请求时就可以带上cookie值，服务器就可以识别是哪个用户发送的请求，是否已认证，是否登录过期等等。这就是传统的session认证方式。

session认证的缺点其实很明显，由于session是保存在服务器里，所以如果分布式部署应用的话，会出现session不能共享的问题，很难扩展。于是乎为了解决session共享的问题，又引入了redis，接着往下看。

## token认证

这种方式跟session的方式流程差不多，不同的地方在于保存的是一个token值到redis，token一般是一串随机的字符(比如UUID)，value一般是用户ID，并且设置一个过期时间。每次请求服务的时候带上token在请求头，后端接收到token则根据token查一下redis是否存在，如果存在则表示用户已认证，如果token不存在则跳到登录界面让用户重新登录，登录成功后返回一个token值给客户端。

优点是多台服务器都是使用redis来存取token，不存在不共享的问题，所以容易扩展。缺点是每次请求都需要查一下redis，会造成redis的压力，还有增加了请求的耗时，每个已登录的用户都要保存一个token在redis，也会消耗redis的存储空间。

有没有更好的方式呢？接着往下看。

# 什么是JWT

JWT(全称：Json Web Token)是一个开放标准(RFC 7519)，它定义了一种紧凑的、自包含的方式，用于作为JSON对象在各方之间安全地传输信息。该信息可以被验证和信任，因为它是数字签名的。

上面说法比较文绉绉，简单点说就是一种认证机制，让后台知道该请求是来自于受信的客户端。

首先我们先看一个流程图：

![](https://static.lovebilibili.com/jwt_01.png)

流程描述一下：

1. 用户使用账号、密码登录应用，登录的请求发送到Authentication Server。
2. Authentication Server进行用户验证，然后创建JWT字符串返回给客户端。
3. 客户端请求接口时，在请求头带上JWT。
4. Application Server验证JWT合法性，如果合法则继续调用应用接口返回结果。

可以看出与token方式有一些不同的地方，就是不需要依赖redis，用户信息存储在客户端。所以关键在于生成JWT，和解析JWT这两个地方。

## JWT的数据结构

JWT一般是这样一个字符串，分为三个部分，以"."隔开：

```java
xxxxx.yyyyy.zzzzz
```

![](https://static.lovebilibili.com/jwt_02.jpg)

### Header

JWT第一部分是头部分，它是一个描述JWT元数据的Json对象，通常如下所示。

```json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

alg属性表示签名使用的算法，默认为HMAC SHA256（写为HS256），typ属性表示令牌的类型，JWT令牌统一写为JWT。

最后，使用Base64 URL算法将上述JSON对象转换为字符串保存。

### Payload

JWT第二部分是Payload，也是一个Json对象，除了包含需要传递的数据，还有七个默认的字段供选择。

分别是，iss：发行人、exp：到期时间、sub：主题、aud：用户、nbf：在此之前不可用、iat：发布时间、jti：JWT ID用于标识该JWT。

如果自定义字段，可以这样定义：

```json
{
    //默认字段
    "sub":"主题123",
    //自定义字段
    "name":"java技术爱好者",
    "isAdmin":"true",
    "loginTime":"2021-12-05 12:00:03"
}
```

需要注意的是，默认情况下JWT是未加密的，任何人都可以解读其内容，因此如果一些敏感信息不要存放在此，以防信息泄露。

JSON对象也使用Base64 URL算法转换为字符串保存。

### Signature

JWT第三部分是签名。是这样生成的，首先需要指定一个secret，该secret仅仅保存在服务器中，保证不能让其他用户知道。然后使用Header指定的算法对Header和Payload进行计算，然后就得出一个签名哈希。也就是Signature。

那么Application Server如何进行验证呢？可以利用JWT前两段，用同一套哈希算法和同一个secret计算一个签名值，然后把计算出来的签名值和收到的JWT第三段比较，如果相同则认证通过。

# JWT的优点

- json格式的通用性，所以JWT可以跨语言支持，比如Java、JavaScript、PHP、Node等等。
- 可以利用Payload存储一些非敏感的信息。
- 便于传输，JWT结构简单，字节占用小。
- 不需要在服务端保存会话信息，易于应用的扩展。

首先引入Maven依赖。

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```

创建工具类，用于创建jwt字符串和解析jwt。

```java
@Component
public class JwtUtil {

    @Value("${jwt.secretKey}")
    private String secretKey;

    public String createJWT(String id, String subject, long ttlMillis, Map<String, Object> map) throws Exception {
        JwtBuilder builder = Jwts.builder()
                .setSubject(null) // 发行者
                .setId(id)
                .setSubject(subject)
                .setIssuedAt(new Date()) // 发行时间
                .signWith(SignatureAlgorithm.HS256, secretKey) // 签名类型 与 密钥
                .compressWith(CompressionCodecs.DEFLATE);// 对载荷进行压缩
        if (!CollectionUtils.isEmpty(map)) {
            builder.setClaims(map);
        }
        if (ttlMillis > 0) {
            builder.setExpiration(new Date(System.currentTimeMillis() + ttlMillis));
        }
        return builder.compact();
    }


    public Claims parseJWT(String jwtString) {
        return Jwts.parser().setSigningKey(secretKey)
                .parseClaimsJws(jwtString)
                .getBody();
    }
}
```

接着在application.yml配置文件配置`jwt.secretKey`。

```yml
## 用户生成jwt字符串的secretKey
jwt:
  secretKey: ak47
```

接着创建一个响应体。

```java
public class BaseResponse {

    private String code;

    private String msg;

    public static BaseResponse success() {
        return new BaseResponse("0", "成功");
    }

    public static BaseResponse fail() {
        return new BaseResponse("1", "失败");
    }
    //构造器、getter、setter方法
}

public class JwtResponse extends BaseResponse {

    private String jwtData;

    public static JwtResponse success(String jwtData) {
        BaseResponse success = BaseResponse.success();
        return new JwtResponse(success.getCode(), success.getMsg(), jwtData);
    }

    public static JwtResponse fail(String jwtData) {
        BaseResponse fail = BaseResponse.fail();
        return new JwtResponse(fail.getCode(), fail.getMsg(), jwtData);
    }
    //构造器、getter、setter方法
}
```

接着创建一个UserController：

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Resource
    private UserService userService;

    @RequestMapping(value = "/login", method = RequestMethod.POST)
    public JwtResponse login(@RequestParam(name = "userName") String userName,
                             @RequestParam(name = "passWord") String passWord){
        String jwt = "";
        try {
            jwt = userService.login(userName, passWord);
            return JwtResponse.success(jwt);
        } catch (Exception e) {
            e.printStackTrace();
            return JwtResponse.fail(jwt);
        }
    }
}
```

还有UserService：

```java
@Service
public class UserServiceImpl implements UserService {

    @Resource
    private JwtUtil jwtUtil;

    @Resource
    private UserMapper userMapper;

    @Override
    public String login(String userName, String passWord) throws Exception {
        //登录验证
        User user = userMapper.findByUserNameAndPassword(userName, passWord);
        if (user == null) {
            return null;
        }
        //如果能查出，则表示账号密码正确，生成jwt返回
        String uuid = UUID.randomUUID().toString().replace("-", "");
        HashMap<String, Object> map = new HashMap<>();
        map.put("name", user.getName());
        map.put("age", user.getAge());
        return jwtUtil.createJWT(uuid, "login subject", 0L, map);
    }
}
```

还有UserMapper.xml：

```java
@Mapper
public interface UserMapper {
    User findByUserNameAndPassword(@Param("userName") String userName, @Param("passWord") String passWord);

}
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="io.github.yehongzhi.jwtdemo.mapper.UserMapper">
    <select id="findByUserNameAndPassword" resultType="io.github.yehongzhi.jwtdemo.model.User">
        select * from user where user_name = #{userName} and pass_word = #{passWord}
    </select>
</mapper>
```

user表结构如下：

![](https://static.lovebilibili.com/jwt_03.png)

启动项目，然后用POSTMAN请求login接口。

![](https://static.lovebilibili.com/jwt_04.png)

返回的jwt字符串如下：

```java
eyJhbGciOiJIUzI1NiIsInppcCI6IkRFRiJ9.eNqqVspLzE1VslJ6OnHFsxnzX67coKSjlJgOFDEzqAUAAAD__w.qib2DrjRKcFnY77Cuh_b1zSzXfISOpCA-g8PlAZCWoU
```

接着我们写一个接口接收这个jwt，并做验证。

```java
@RestController
@RequestMapping("/jwt")
public class TestController {

    @Resource
    private JwtUtil jwtUtil;

    @RequestMapping("/test")
    public Map<String, Object> test(@RequestParam("jwt") String jwt) {
        //这个步骤可以使用自定义注解+AOP编程做解析jwt的逻辑，这里为了简便就直接写在controller里
        Claims claims = jwtUtil.parseJWT(jwt);
        String name = claims.get("name", String.class);
        String age = claims.get("age", String.class);
        HashMap<String, Object> map = new HashMap<>();
        map.put("name", name);
        map.put("age", age);
        map.put("code", "0");
        map.put("msg", "请求成功");
        return map;
    }
}
```

![](https://static.lovebilibili.com/jwt_05.png)

像这样能正常解析成功的话，就表示该用户登录未过期，并且已认证成功，所以可以正常调用服务。那么有人会问了，这个jwt字符串能不能被伪造呢？

除非你知道secretKey，否则是不能伪造的。比如客户端随便猜一个secretKey的值，然后伪造一个jwt：

```java
eyJhbGciOiJIUzI1NiIsInppcCI6IkRFRiJ9.eNqqVspLzE1VslJ6OnHFsxnzX67coKSjlJgOFDEzqAUAAAD__w.bHr9p3-t2qR4R50vifRVyaYYImm2viZqiTlDdZHmF5Y
```

然后传进去解析，会报以下错误：

![](https://static.lovebilibili.com/jwt_06.png)

还记得原理吧，是根据前面两部分(Header、Payload)加上secretKey使用Header指定的哈希算法计算出第三部分(Signature)，所以可以看出最关键就是secretKey。secretKey只有服务端自己知道，所以客户端不知道secretKey的值是伪造不了jwt字符串的。

# 总结

最后讲讲JWT的缺点，任何技术都不是完美的，所以我们得用辩证思维去看待任何一项技术。

- 安全性没法保证，所以jwt里不能存储敏感数据。因为jwt的payload并没有加密，只是用Base64编码而已。
- 无法中途废弃。因为一旦签发了一个jwt，在到期之前始终都是有效的，如果用户信息发生更新了，只能等旧的jwt过期后重新签发新的jwt。
- 续签问题。当签发的jwt保存在客户端，客户端一直在操作页面，按道理应该一直为客户端续长有效时间，否则当jwt有效期到了就会导致用户需要重新登录。那么怎么为jwt续签呢？最简单粗暴就是每次签发新的jwt，但是由于过于暴力，会影响性能。如果要优雅一点，又要引入Redis解决，但是这又把无状态的jwt硬生生变成了有状态的，违背了初衷。

所以印证了那句话，没有最好的技术，只有适合的技术。感谢大家的阅读，希望看完之后能对你有所收获。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！

