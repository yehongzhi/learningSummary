# 思维导图

![](https://static.lovebilibili.com/springmvc_swdt.png)

> 微信公众号已开启：【**java技术爱好者**】，还没关注的记得关注哦~
>
> **文章已收录到我的Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 概述

SpringMVC再熟悉不过的框架了，因为现在最火的SpringBoot的内置MVC框架就是SpringMVC。我写这篇文章的动机是想通过回顾总结一下，重新认识SpringMVC，所谓温故而知新嘛。

为了了解SpringMVC，先看一个流程示意图：

![](https://static.lovebilibili.com/springmvc_1.png)

从流程图中，我们可以看到：

- 接收前端传过来Request请求。
- 根据映射路径找到对应的处理器处理请求，处理完成之后返回ModelAndView。
- 进行视图解析，视图渲染，返回响应结果。

总结就是：**参数接收，定义映射路径，页面跳转，返回响应结果**。

当然这只是最基本的核心功能，除此之外还可以**定义拦截器，全局异常处理，文件上传下载**等等。

# 一、搭建项目

在以前的老项目中，因为还没有SpringBoot，没有自动配置，所以需要使用**web.xml**文件去定义一个DispatcherServlet。现在互联网应用基本上都使用SpringBoot，所以我就直接使用SpringBoot进行演示。很简单，引入依赖即可：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

# 二、定义Controller

使用SpringMVC定义Controller处理器，总共有五种方式。

## 2.1 实现Controller接口

早期的SpringMVC是通过这种方式定义：

```java
/**
 * @author Ye Hongzhi 公众号：java技术爱好者
 * @name DemoController
 * @date 2020-08-25 22:28
 **/
@org.springframework.stereotype.Controller("/demo/controller")
public class DemoController implements Controller {
    @Override
    public ModelAndView handleRequest(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws Exception {
        //业务处理
        return null;
    }
}
```

## 2.2 实现HttpRequestHandler接口

跟第一种方式差不多，也是通过实现接口的方式：

```java
/**
 * @author Ye Hongzhi 公众号：java技术爱好者
 * @name HttpDemoController
 * @date 2020-08-25 22:45
 **/
@Controller("/http/controller")
public class HttpDemoController implements HttpRequestHandler{
    @Override
    public void handleRequest(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws ServletException, IOException {
        //业务处理
    }
}
```

## 2.3 实现Servlet接口

这种方式已经不推荐使用了，不过从这里可以看出**SpringMVC的底层使用的还是Servlet**。

```java
@Controller("/servlet/controller")
public class ServletDemoController implements Servlet {
    //以下是Servlet生命周期方法
    @Override
    public void init(ServletConfig servletConfig) throws ServletException {
    }

    @Override
    public ServletConfig getServletConfig() {
        return null;
    }

    @Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
    }

    @Override
    public String getServletInfo() {
        return null;
    }

    @Override
    public void destroy() {
    }
}
```

因为不推荐使用这种方式，所以默认是不加载这种适配器的，需要加上：

```java
@Configuration
@EnableWebMvc
public class WebMvcConfig extends WebMvcConfigurerAdapter {

    @Bean
    public SimpleServletHandlerAdapter simpleServletHandlerAdapter() {
        return new SimpleServletHandlerAdapter();
    }
}
```

## 2.4 使用@RequestMapping

这种方式是最常用的，因为上面那些方式定义需要使用一个类定义一个路径，就会导致产生很多类。使用注解就相对轻量级一些。

```java
@Controller
@RequestMapping("/requestMapping/controller")
public class RequestMappingController {

    @RequestMapping("/demo")
    public String demo() {
        return "HelloWord";
    }
}
```

### 2.4.1 支持Restful风格

而且支持**Restful风格**，使用**method**属性定义对资源的操作方式：

```java
	@RequestMapping(value = "/restful", method = RequestMethod.GET)
    public String get() {
        //查询
        return "get";
    }

    @RequestMapping(value = "/restful", method = RequestMethod.POST)
    public String post() {
        //创建
        return "post";
    }

    @RequestMapping(value = "/restful", method = RequestMethod.PUT)
    public String put() {
        //更新
        return "put";
    }

    @RequestMapping(value = "/restful", method = RequestMethod.DELETE)
    public String del() {
        //删除
        return "post";
    }
```

### 2.4.2 支持Ant风格

```java
	//匹配 /antA 或者 /antB 等URL
    @RequestMapping("/ant?")
    public String ant() {
        return "ant";
    }

    //匹配 /ant/a/create 或者 /ant/b/create 等URL
    @RequestMapping("/ant/*/create")
    public String antCreate() {
        return "antCreate";
    }

    //匹配 /ant/create 或者 /ant/a/b/create 等URL
    @RequestMapping("/ant/**/create")
    public String antAllCreate() {
        return "antAllCreate";
    }
```

## 2.5 使用HandlerFunction

最后一种是使用HandlerFunction函数式接口，这是`Spring5.0`后引入的方式，主要用于做响应式接口的开发，也就是Webflux的开发。

有兴趣的可以网上搜索相关资料学习，这个讲起来可能要很大篇幅，这里就不赘述了。

# 三、接收参数

定义完Controller之后，需要接收前端传入的参数，怎么接收呢。

## 3.1 接收普通参数

在@RequestMapping映射方法上写上接收参数名即可：

```java
@RequestMapping(value = "/restful", method = RequestMethod.POST)
public String post(Integer id, String name, int money) {
    System.out.println("id:" + id + ",name:" + name + ",money:" + money);
    return "post";
}
```

![](https://static.lovebilibili.com/springmvc_2.png)

## 3.2 @RequestParam参数名绑定

如果不想使用形参名称作为参数名称，可以使用@RequestParam进行参数名称绑定：

```java
	/**
     * value: 参数名
     * required: 是否request中必须包含此参数，默认是true。
     * defaultValue: 默认参数值
     */
    @RequestMapping(value = "/restful", method = RequestMethod.GET)
    public String get(@RequestParam(value = "userId", required = false, defaultValue = "0") String id) {
        System.out.println("id:" + id);
        return "get";
    }
```

![](https://static.lovebilibili.com/springmvc_3.png)

## 3.3 @PathVariable路径参数

通过@PathVariable将URL中的占位符{xxx}参数映射到操作方法的入参。演示代码如下：

```java
@RequestMapping(value = "/restful/{id}", method = RequestMethod.GET)
public String search(@PathVariable("id") String id) {
    System.out.println("id:" + id);
    return "search";
}
```

![](https://static.lovebilibili.com/springmvc_4.png)

## 3.4 @RequestHeader绑定请求头属性

获取请求头的信息怎么获取呢？

![](https://static.lovebilibili.com/springmvc_5.png)

使用@RequestHeader注解，用法和@RequestParam类似：

```java
	@RequestMapping("/head")
    public String head(@RequestHeader("Accept-Language") String acceptLanguage) {
        return acceptLanguage;
    }
```

![](https://static.lovebilibili.com/springmvc_6.png)

## 3.5 @CookieValue绑定请求的Cookie值

获取Request中Cookie的值：

```java
	@RequestMapping("/cookie")
    public String cookie(@CookieValue("_ga") String _ga) {
        return _ga;
    }
```

![](https://static.lovebilibili.com/springmvc_7.png)

## 3.6 绑定请求参数到POJO对象

定义了一个User实体类：

```java
public class User {
    private String id;
    private String name;
    private Integer age;
    //getter、setter方法
}
```

定义一个@RequestMapping操作方法：

```java
	@RequestMapping("/body")
    public String body(User user) {
        return user.toString();
    }
```

只要请求参数与属性名相同自动填充到user对象中：

![](https://static.lovebilibili.com/springmvc_8.png)

![](https://static.lovebilibili.com/springmvc_9.png)

### 3.6.1 支持级联属性

现在多了一个Address类存储地址信息：

```java
public class Address {
    private String id;
    private String name;
    //getter、setter方法
}
```

在User中加上address属性：

```java
public class User {
    private String id;
    private String name;
    private Integer age;
    private Address address;
    //getter、setter方法
}
```

传参时只要传入address.name、address.id即会自动填充：

![](https://static.lovebilibili.com/springmvc_11.png)

![](https://static.lovebilibili.com/springmvc_10.png)

### 3.6.2 @InitBinder解决接收多对象时属性名冲突

如果有两个POJO对象拥有相同的属性名，不就产生冲突了吗？比如刚刚的user和address，其中他们都有id和name这两个属性，如果同时接收，就会冲突：

```java
	//user和address都有id和name这两个属性	
	@RequestMapping(value = "/twoBody", method = RequestMethod.POST)
    public String twoBody(User user, Address address) {
        return user.toString() + "," + address.toString();
    }
```

这时就可以使用@InitBinder绑定参数名称：

```java
	@InitBinder("user")
    public void initBindUser(WebDataBinder webDataBinder) {
        webDataBinder.setFieldDefaultPrefix("u.");
    }

    @InitBinder("address")
    public void initBindAddress(WebDataBinder webDataBinder) {
        webDataBinder.setFieldDefaultPrefix("addr.");
    }
```

![](https://static.lovebilibili.com/springmvc_12.png)

![](https://static.lovebilibili.com/springmvc_13.png)

### 3.6.3 @Requestbody自动解析JSON字符串封装到对象

前端传入一个json字符串，自动转换成pojo对象，演示代码：

```java
	@RequestMapping(value = "/requestBody", method = RequestMethod.POST)
    public String requestBody(@RequestBody User user) {
        return user.toString();
    }
```

注意的是，要使用**POST请求，发送端的Content-Type设置为application/json，数据是json字符串**：

![](https://static.lovebilibili.com/springmvc_14.png)

![](https://static.lovebilibili.com/springmvc_15.png)

甚至有一些人喜欢用一个Map接收：

![](https://static.lovebilibili.com/springmvc_16.png)

但是**千万不要用Map接收，否则会造成代码很难维护**，后面的老哥估计看不懂你这个Map里面有什么数据，所以最好还是定义一个POJO对象。

# 四、参数类型转换

实际上，SpringMVC框架本身就内置了很多类型转换器，比如你传入字符串的数字，接收的入参定为int，long类型，都会自动帮你转换。

就在包**org.springframework.core.convert.converter**下，如图所示：

![](https://static.lovebilibili.com/springmvc_17.png)

有的时候如果内置的类型转换器不足够满足业务需求呢，怎么扩展呢，很简单，看我操作。什么是Java技术爱好者(战术后仰)。

首先有样学样，内置的转换器实现Converter接口，我也实现：

```java
public class StringToDateConverter implements Converter<String, Date> {
    @Override
    public Date convert(String source) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        try {
            //String转换成Date类型
            return sdf.parse(source);
        } catch (Exception e) {
            //类型转换错误
            e.printStackTrace();
        }
        return null;
    }
}
```

接着把转换器注册到Spring容器中：

```java
@Configuration
public class ConverterConfig extends WebMvcConfigurationSupport {
    @Override
    protected void addFormatters(FormatterRegistry registry) {
        //添加类型转换器
        registry.addConverter(new StringToDateConverter());
    }
}
```

接着看测试，所有的日期字符串，都自动被转换成Date类型了，非常方便：

![](https://static.lovebilibili.com/springmvc_19.png)

![](https://static.lovebilibili.com/springmvc_18.png)

# 五、页面跳转

在前后端未分离之前，页面跳转的工作都是由后端控制，采用JSP进行展示数据。虽然现在互联网项目几乎不会再使用JSP，但是我觉得还是需要学习一下，因为有些旧项目还是会用JSP，或者需要重构。

如果你在RequestMapping方法中直接返回一个字符串是不会跳转到指定的JSP页面的，需要做一些配置。

第一步，加入解析jsp的Maven配置。

```xml
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-jasper</artifactId>
    <version>7.0.59</version>
</dependency>
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>jstl</artifactId>
</dependency>
```

第二步，添加视图解析器。

```java
@Configuration
public class WebAppConfig extends WebMvcConfigurerAdapter {
    @Bean
    public InternalResourceViewResolver viewResolver() {
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setPrefix("/");
        viewResolver.setSuffix(".jsp");
        viewResolver.setViewClass(JstlView.class);
        return viewResolver;
    }
}
```

第三步，设置IDEA的配置。

![](https://static.lovebilibili.com/springmvc_21.png)

![](https://static.lovebilibili.com/springmvc_20.png)

![](https://static.lovebilibili.com/springmvc_22.png)

第四步，创建jsp页面。

![](https://static.lovebilibili.com/springmvc_23.png)

第五步，创建Controller控制器。

```java
@Controller
@RequestMapping("/view")
public class ViewController {
    @RequestMapping("/hello")
    public String hello() throws Exception {
        return "hello";
    }
}
```

这样就完成了，启动项目，访问/view/hello就看到了：

![](https://static.lovebilibili.com/springmvc_25.png)

就是这么简单，对吧

# 六、@ResponseBody

如果采用前后端分离，页面跳转不需要后端控制了，后端只需要返回json即可，怎么返回呢？

使用@ResponseBody注解即可，这个注解会把对象自动转成json数据返回。

@ResponseBody注解可以放在类或者方法上，源码如下：

```java
//用在类、方法上
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ResponseBody {
}
```

演示一下：

```java
@RequestMapping("/userList")
@ResponseBody
public List<User> userList() throws Exception {
    List<User> list = new ArrayList<>();
    list.add(new User("1","姚大秋",18));
    list.add(new User("2","李星星",18));
    list.add(new User("3","冬敏",18));
    return list;
}
```

测试一下/view/userList：

![](https://static.lovebilibili.com/springmvc_26.png)

# 七、@ModelAttribute

@ModelAttribute用法比较多，下面一一讲解。

## 7.1 用在无返回值的方法上

在Controller类中，在执行所有的RequestMapping方法前都会先执行@ModelAttribute注解的方法。

```java
@Controller
@RequestMapping("/modelAttribute")
public class ModelAttributeController {
	//先执行这个方法
    @ModelAttribute
    public void modelAttribute(Model model){
        //在request域中放入数据
        model.addAttribute("userName","公众号：java技术爱好者");
    }

    @RequestMapping("/index")
    public String index(){
        //跳转到inex.jsp页面
        return "index";
    }
}
```

index.jsp页面如下：

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>首页</title>
</head>
<body>
<!-- 获取到userName属性值 -->
<h1>${userName}</h1>
</body>
</html>
```

相当于一个Controller的拦截器一样，在执行RequestMapping方法前先执行@ModelAttribute注解的方法。所以要慎用。

启动项目，访问/modelAttribute/index可以看到：

![](https://static.lovebilibili.com/springmvc_27.png)

即使在index()方法中没有放入userName属性值，jsp页面也能获取到，因为在执行index()方法之前的modelAttribute()方法已经放入了。

## 7.2 放在有返回值的方法上

其实调用顺序是一样，也是在RequestMapping方法前执行，不同的在于，方法的返回值直接帮你放入到Request域中。

```java
//放在有参数的方法上
@ModelAttribute
public User userAttribute() {
    //相当于model.addAttribute("user",new User("1", "Java技术爱好者", 18));
    return new User("1", "Java技术爱好者", 18);
}

@RequestMapping("/user")
public String user() {
    return "user";
}
```

创建一个user.jsp:

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>首页</title>
</head>
<body>
<h1>ID:${user.id}</h1>
<h1>名称:${user.name}</h1>
<h1>年龄:${user.age}岁</h1>
</body>
</html>
```

测试一下：

![](https://static.lovebilibili.com/springmvc_28.png)

放入Request域中的属性值默认是类名的首字母小写驼峰写法，如果你想自定义呢？很简单，可以这样写：

```java
//自定义属性名为"u"
@ModelAttribute("u")
public User userAttribute() {
    return new User("1", "Java技术爱好者", 18);
}
/**
JSP就要改成这样写：
<h1>ID:${u.id}</h1>
<h1>名称:${u.name}</h1>
<h1>年龄:${u.age}岁</h1>
*/
```

## 7.3 放在RequestMapping方法上

```java
@Controller
@RequestMapping("/modelAttribute")
public class ModelAttributeController {
    
    @RequestMapping("/jojo")
    @ModelAttribute("attributeName")
    public String jojo() {
        return "JOJO！我不做人了！";
    }
}
```

这种情况下RequestMapping方法的返回的值就不是JSP视图了。而是把返回值放入Request域中的属性值，属性名为attributeName。视图则是RequestMapping注解上的URL，所以创建一个对应的JSP页面：

![](https://static.lovebilibili.com/springmvc_29.png)

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>首页</title>
</head>
<body>
<h1>${attributeName}</h1>
</body>
</html>
```

测试一下：

![](https://static.lovebilibili.com/springmvc_30.png)

## 7.4 放在方法入参上

放在入参上，意思是从前面的Model中提取出对应的属性值，当做入参传入方法中使用。如下所示：

```java
@ModelAttribute("u")
public User userAttribute() {
    return new User("1", "Java技术爱好者", 18);
}

@RequestMapping("/java")
public String user1(@ModelAttribute("u") User user) {
    //拿到@ModelAttribute("u")方法返回的值，打印出来
    System.out.println("user:" + user);
    return "java";
}
```

测试一下：

![](https://static.lovebilibili.com/springmvc_31.png)

# 八、拦截器

拦截器算重点内容了，很多时候都要用拦截器，比如登录校验，权限校验等等。SpringMVC怎么添加拦截器呢？

很简单，实现HandlerInterceptor接口，接口有三个方法需要重写。

- preHandle()：在业务处理器处理请求之前被调用。预处理。
- postHandle()：在业务处理器处理请求执行完成后，生成视图之前执行。后处理。
- afterCompletion()：在DispatcherServlet完全处理完请求后被调用，可用于清理资源等。返回处理（已经渲染了页面）；

自定义的拦截器，实现的接口HandlerInterceptor：

```java
public class DemoInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //预处理，返回true则继续执行。如果需要登录校验，校验不通过返回false即可，通过则返回true。
        System.out.println("执行preHandle()方法");
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        //后处理
        System.out.println("执行postHandle()方法");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        //在DispatcherServlet完全处理完请求后被调用
        System.out.println("执行afterCompletion()方法");
    }
}
```

然后把拦截器添加到Spring容器中：

```java
@Configuration
public class ConverterConfig extends WebMvcConfigurationSupport {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new DemoInterceptor()).addPathPatterns("/**");
    }
}
```

/**代表所有路径，测试一下：

![](https://static.lovebilibili.com/springmvc_32.png)

# 九、全局异常处理

SpringMVC本身就对一些异常进行了全局处理，所以有内置的异常处理器，在哪里呢？

看`HandlerExceptionResolver`接口的类图就知道了：

![](https://static.lovebilibili.com/HandlerExceptionResolver.png)

从类图可以看出有四种异常处理器：

- `DefaultHandlerExceptionResolver`，默认的异常处理器。根据各个不同类型的异常，返回不同的异常视图。
- `SimpleMappingExceptionResolver`，简单映射异常处理器。通过配置异常类和view的关系来解析异常。
- `ResponseStatusExceptionResolver`，状态码异常处理器。解析带有`@ResponseStatus`注释类型的异常。
- `ExceptionHandlerExceptionResolver`，注解形式的异常处理器。对`@ExceptionHandler`注解的方法进行异常解析。

第一个默认的异常处理器是内置的异常处理器，对一些常见的异常处理，一般来说不用管它。后面的三个才是需要注意的，是用来扩展的。

## 9.1 SimpleMappingExceptionResolver

翻译过来就是简单映射异常处理器。用途是，我们可以**指定某种异常，当抛出这种异常之后跳转到指定的页面**。请看演示。

第一步，添加spring-config.xml文件，放在resources目录下，文件名见文知意即可：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean class="org.springframework.web.servlet.handler.SimpleMappingExceptionResolver">
        <!-- 定义默认的异常处理页面 -->
        <property name="defaultErrorView" value="err"/>
        <!-- 定义异常处理页面用来获取异常信息的属性名，默认名为exception -->
        <property name="exceptionAttribute" value="ex"/>
        <!-- 定义需要特殊处理的异常，用类名或完全路径名作为key，异常也页名作为值 -->
        <property name="exceptionMappings">
            <props>
                <!-- 异常，err表示err.jsp页面 -->
                <prop key="java.lang.Exception">err</prop>
                <!-- 可配置多个prop -->
            </props>
        </property>
    </bean>
</beans>
```

第二步，在启动类加载xml文件：

```java
@SpringBootApplication
@ImportResource("classpath:spring-config.xml")
public class SpringmvcApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringmvcApplication.class, args);
    }

}
```

第三步，在webapp目录下创建一个err.jsp页面：

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>异常页面</title>
</head>
<body>
<h1>出现异常，这是一张500页面</h1>
<br>
<%-- 打印异常到页面上 --%>
<% Exception ex = (Exception)request.getAttribute("ex"); %>
<br>
<div><%=ex.getMessage()%></div>
<% ex.printStackTrace(new java.io.PrintWriter(out)); %>
</body>
</html>
```

这样就完成了，写一个接口测试一下：

```java
@Controller
@RequestMapping("/exception")
public class ExceptionController {
    @RequestMapping("/index")
    public String index(String msg) throws Exception {
        if ("null".equals(msg)) {
            //抛出空指针异常
            throw new NullPointerException();
        }
        return "index";
    }
}
```

效果如下：

![](https://static.lovebilibili.com/springmvc_33.png)

这种异常处理器，在现在前后端分离的项目中几乎已经看不到了。

## 9.2 ResponseStatusExceptionResolver

这种异常处理器主要用于处理带有`@ResponseStatus`注释的异常。请看演示代码：

自定义一个异常类，并且使用`@ResponseStatus`注解修饰：

```java
//HttpStatus枚举有所有的状态码，这里返回一个400的响应码
@ResponseStatus(value = HttpStatus.BAD_REQUEST)
public class DefinedException extends Exception{
}
```

写一个Controller接口进行测试：

```java
@RequestMapping("/defined")
public String defined(String msg) throws Exception {
    if ("defined".equals(msg)) {
        throw new DefinedException();
    }
    return "index";
}
```

启动项目，测试一下，效果如下：

![](https://static.lovebilibili.com/springmvc_34.png)

## 9.3 ExceptionHandlerExceptionResolver

注解形式的异常处理器，这是用得最多的。使用起来非常简单方便。

第一步，定义自定义异常BaseException：

```java
public class BaseException extends Exception {
    public BaseException(String message) {
        super(message);
    }
}
```

第二步，定义一个错误提示实体类ErrorInfo：

```java
public class ErrorInfo {
    public static final Integer OK = 0;
    public static final Integer ERROR = -1;
    private Integer code;
    private String message;
    private String url;
    //getter、setter
}
```

第三步，定义全局异常处理类GlobalExceptionHandler：

```java
//这里使用了RestControllerAdvice，是@ResponseBody和@ControllerAdvice的结合
//会把实体类转成JSON格式的提示返回，符合前后端分离的架构
@RestControllerAdvice
public class GlobalExceptionHandler {

    //这里自定义了一个BaseException，当抛出BaseException异常就会被此方法处理
    @ExceptionHandler(BaseException.class)
    public ErrorInfo errorHandler(HttpServletRequest req, BaseException e) throws Exception {
        ErrorInfo r = new ErrorInfo();
        r.setMessage(e.getMessage());
        r.setCode(ErrorInfo.ERROR);
        r.setUrl(req.getRequestURL().toString());
        return r;
    }
}
```

完成之后，写一个测试接口：

```java
@RequestMapping("/base")
public String base(String msg) throws Exception {
    if ("base".equals(msg)) {
        throw new BaseException("测试抛出BaseException异常，欧耶！");
    }
    return "index";
}
```

启动项目，测试：

![](https://static.lovebilibili.com/springmvc_35.png)

# 絮叨

SpringMVC的功能实际上肯定还不止我写的这些，不过学会上面这些之后，基本上已经可以应对日常的工作了。

如果要再深入一些，最好是看看SpringMVC源码，我之前写过三篇，**责任链模式与SpringMVC拦截器，适配器模式与SpringMVC，全局异常处理源码分析**。有兴趣可以关注公众号看看我的历史文章。

> 微信公众号已开启：【java技术爱好者】，没关注的同学记得关注哦~
>
> 坚持原创，持续输出兼具广度和深度的技术文章。

上面所有例子的代码都上传Github了：

> https://github.com/yehongzhi/mall

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！