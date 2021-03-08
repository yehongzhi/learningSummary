> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 什么是注解

注解是JDK1.5引入的新特性，主要用于简化代码，提高编程的效率。其实在日常开发中，注解并不少见，比如Java内置的`@Override`、`@SuppressWarnings`，或者Spring提供的`@Service`、`@Controller`等等，随着这些注解使用的频率越来越高，作为开发人员当真有必要深入学习一番。

# Java内置的注解

先说说Java内置的三个注解，分别是：

`@Override`：检查当前的方法定义是否覆盖父类中的方法，如果没有覆盖，编译器就会报错。

`@SuppressWarnings`：忽略编译器的警告信息。

![](https://static.lovebilibili.com/zhujie_3.png)

![](https://static.lovebilibili.com/zhujie_4.png)

`@Deprecated`：用于标识该类或方法已过时，建议开发人员不要使用该类或方法。

![](https://static.lovebilibili.com/zhujie_1.png)

![](https://static.lovebilibili.com/zhujie_2.png)

# 元注解

元注解其实就是描述注解的注解。主要有四个元注解，分别是：

## @Target

用于描述注解的使用范围，也就是注解可以用在什么地方，取值有：

CONSTRUCTOR：用于描述构造器。

FIELD：用于描述字段。

LOCAL_VARIABLE：用于描述局部变量。

METHOD：用于描述方法。

PACKAGE：用于描述包。

PARAMETER：用于描述参数。

TYPE：用于描述类，包括class，interface，enum。 

## @Retention

**表示需要在什么级别保存该注释信息，用于描述注解的生命周期**，取值由枚举RetentionPoicy定义。

![](https://static.lovebilibili.com/zhujie_5.png)

SOURCE：在源文件中有效（即源文件保留），仅出现在源代码中，而被编译器丢弃。

CLASS：在class文件中有效（即class保留），但会被JVM丢弃。

RUNTIME：JVM将在运行期也保留注释，因此可以通过反射机制读取注解的信息。

如果只是做一些检查性操作，使用SOURCE，比如@Override，@SuppressWarnings。

如果要在编译时进行一些预处理操作，就用 CLASS。

如果需要获取注解的属性值，去做一些运行时的逻辑，可以使用RUNTIME。

## @Documented

将此注解包含在 javadoc 中 ，它代表着此注解会被javadoc工具提取成文档。它是一个标记注解，没有成员。

![](https://static.lovebilibili.com/zhujie_6.png)

## @Inherited

是一个标记注解，用来指定该注解可以被继承。使用 @Inherited 注解的 Class 类，表示这个注解可以被用于该 Class 类的子类。

# 自定义注解

下面实战一下，自定义一个注解@LogApi，用于方法上，当被调用时即打印日志，在控制台显示调用方传入的参数和调用返回的结果。

## 定义注解

首先定义注解`@LogApi`，在方法上使用，为了能在反射中读取注解信息，当然是设置为`RUNTIME`。

```java
@Target(value = ElementType.METHOD)
@Documented
@Retention(value = RetentionPolicy.RUNTIME)
public @interface LogApi {
}
```

这种没有属性的注解，属于标记注解。

多说几句，如果需要传递属性值，也可以设置属性值value，比如`@RequestMapping`注解。

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
public @interface RequestMapping {
    @AliasFor("path")
	String[] value() default {};
}
```

如果在使用时。只设置value值，可以忽略value，比如这样：

```java
//完整是@RequestMapping(value = {"/list"})
//忽略value不写
@RequestMapping("/list")
public Map<String, Object> list() throws Exception {
    Map<String, Object> userMap = new HashMap<>();
    userMap.put("1号佳丽", "李嘉欣");
    userMap.put("2号佳丽", "袁咏仪");
    userMap.put("3号佳丽", "张敏");
    userMap.put("4号佳丽", "张曼玉");
    return userMap;
}
```

## 标记注解

刚刚定义完注解之后，就可以在需要的地方标记注解，很简单。

```java
@LogApi
@RequestMapping("/list")
public Map<String, Object> list() throws Exception {
	//业务代码...
}
```

## 解析注解

最关键的一步来了，解析注解，一般在项目中会使用Spring的AOP技术解析注解，当然如果只需要解析一次的话，也可以使用Spring容器的生命周期函数。

这里的场景是打印每次方法被调用的日志，所以使用AOP比较合适。

创建一个切面类`LogApiAspect`进行解析。

```java
@Aspect
@Component
public class LogApiAspect {
	//切面点为标记了@LogApi注解的方法
    @Pointcut("@annotation(io.github.yehongzhi.user.annotation.LogApi)")
    public void logApi() {
    }
    
	//环绕通知
    @Around("logApi()")
    @SuppressWarnings("unchecked")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        long starTime = System.currentTimeMillis();
        //通过反射获取被调用方法的Class
        Class type = joinPoint.getSignature().getDeclaringType();
        //获取类名
        String typeName = type.getSimpleName();
        //获取日志记录对象Logger
        Logger logger = LoggerFactory.getLogger(type);
        //方法名
        String methodName = joinPoint.getSignature().getName();
        //获取参数列表
        Object[] args = joinPoint.getArgs();
        //参数Class的数组
        Class[] clazz = new Class[args.length];
        for (int i = 0; i < args.length; i++) {
            clazz[i] = args[i].getClass();
        }
        //通过反射获取调用的方法method
        Method method = type.getMethod(methodName, clazz);
        //获取方法的参数
        Parameter[] parameters = method.getParameters();
        //拼接字符串，格式为{参数1:值1,参数2::值2}
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < parameters.length; i++) {
            Parameter parameter = parameters[i];
            String name = parameter.getName();
            sb.append(name).append(":").append(args[i]).append(",");
        }
        if (sb.length() > 0) {
            sb.deleteCharAt(sb.lastIndexOf(","));
        }
        //执行结果
        Object res;
        try {
            //执行目标方法，获取执行结果
            res = joinPoint.proceed();
            logger.info("调用{}.{}方法成功，参数为[{}]，返回结果[{}]", typeName, methodName, sb.toString(), JSONObject.toJSONString(res));
        } catch (Exception e) {
            logger.error("调用{}.{}方法发生异常", typeName, methodName);
            //如果发生异常，则抛出异常
            throw e;
        } finally {
            logger.info("调用{}.{}方法，耗时{}ms", typeName, methodName, (System.currentTimeMillis() - starTime));
        }
        //返回执行结果
        return res;
    }
}
```

定义完切面类后，需要在启动类添加启动AOP的注解。

```java
@SpringBootApplication
//添加此注解，开启AOP
@EnableAspectJAutoProxy
public class UserApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class, args);
    }

}
```

## 测试

我们再在Controller控制层增加一个有参数的接口。

```java
@LogApi
@RequestMapping("/get/{id}")
public String get(@PathVariable(name = "id") String id) throws Exception {
    HashMap<String, Object> user = new HashMap<>();
    user.put("id", id);
    user.put("name", "关之琳");
    user.put("经典角色", "十三姨");
    return JSONObject.toJSONString(user);
}
```

启动项目，然后请求接口`list()`，我们可以看到控制台出现被调用方法的日志信息。

![](https://static.lovebilibili.com/zhujie_7.png)

请求有参数的接口`get()`，可以看到参数名称和参数值都被打印在控制台。

![](https://static.lovebilibili.com/zhujie_8.png)

这种记录接口请求参数和返回值的功能，在实际项目中基本上都会使用，因为这能利于系统的排错和性能调优等等。

我们也可以在这个例子中，学会使用注解和切面编程，可谓是一举两得！

# 总结

注解的使用能大大地减少开发的代码量，所以在实际项目的开发中会使用到非常多的注解。特别是做一些公共基础的功能，比如日志记录，事务管理，权限控制这些功能，使用注解就非常高效且优雅。

对于自定义注解，主要有三个步骤，**定义注解，标记注解，解析注解**，并不是很难。

这篇文章讲到这里了，感谢大家的阅读，希望看完这篇文章能有所收获！

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！