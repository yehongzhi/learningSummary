---
title: SpringMVC全局异常处理
date: 2020-06-14 15:13:07
index_img: https://static.lovebilibili.com/exceptionResolver_index.jpg
tags:
	- java
	- SpringMVC
	- 源码分析
---

# SpringMVC全局异常处理

`SpringMVC`除了可以做`URL映射`和`请求拦截`外，还可以做`全局异常`的处理。全局异常处理可能我们平时比较少机会接触，但是每个项目都肯定会做这个处理。比如在上一间公司，是前后端分离的架构，所以后端只要有运行时异常就会报“系统异常，请稍后再试”。如果想要走上架构师的话，这个肯定是要学会的。

## SpringMVC全局异常处理机制

首先，要知道全局异常处理，`SpringMVC`提供了两种方式：

- 实现`HandlerExceptionResolver`接口，自定义异常处理器。
- 使用`HandlerExceptionResolver`接口的子类，也就是`SpringMVC`提供的异常处理器。

所以，总得来说就两种方式，一种是自定义异常处理器，第二种是`SpringMVC`提供的。接下来先说`SpringMVC`提供的几种异常处理器的使用方式，然后再讲自定义异常处理器。

`SpringMVC`提供的异常处理器有哪些呢？我们可以直接看源码的类图。

![](https://static.lovebilibili.com/HandlerExceptionResolver.png)

可以看出有四种：

- `DefaultHandlerExceptionResolver`，默认的异常处理器。根据各个不同类型的异常，返回不同的异常视图。
- `SimpleMappingExceptionResolver`，简单映射异常处理器。通过配置异常类和view的关系来解析异常。
- `ResponseStatusExceptionResolver`，状态码异常处理器。解析带有`@ResponseStatus`注释类型的异常。
- `ExceptionHandlerExceptionResolver`，注解形式的异常处理器。对`@ExceptionHandler`注解的方法进行异常解析。

### DefaultHandlerExceptionResolver

这个异常处理器是`SprngMVC`默认的一个处理器，处理一些常见的异常，比如：没有找到请求参数，参数类型转换异常，请求方式不支持等等。

接着我们看`DefaultHandlerExceptionResolver`类的`doResolveException()`方法：

```java
	@Override
	@Nullable
	protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response,@Nullable Object handler, Exception ex) {
		try {
			if (ex instanceof HttpRequestMethodNotSupportedException) {
				return handleHttpRequestMethodNotSupported((HttpRequestMethodNotSupportedException) ex, request,
						response, handler);
			}
			else if (ex instanceof HttpMediaTypeNotSupportedException) {
				return handleHttpMediaTypeNotSupported((HttpMediaTypeNotSupportedException) ex, request, response,
						handler);
			}
			else if (ex instanceof HttpMediaTypeNotAcceptableException) {
				return handleHttpMediaTypeNotAcceptable((HttpMediaTypeNotAcceptableException) ex, request, response,
						handler);
			}
			//省略...以下还有十几种异常的else-if
		}catch (Exception handlerException) {
            //是否打开日志，如果打开，那就记录日志
			if (logger.isWarnEnabled()) {
				logger.warn("Handling of [" + ex.getClass().getName() + "] resulted in Exception", handlerException);
			}
		}
		return null;
	}
```

通过`if-else`判断，判断继承什么异常就显示对应的错误码和错误提示信息。由此可以知道，处理一般有两步，一是设置响应码，二是在响应头设置异常信息。下面是`MissingServletRequestPartException`的处理的源码：

```java
	protected ModelAndView handleMissingServletRequestPartException(MissingServletRequestPartException ex,
			HttpServletRequest request, HttpServletResponse response, @Nullable Object handler) throws IOException {
		//设置响应码，设置异常信息，SC_BAD_REQUEST就是400(bad request)
		response.sendError(HttpServletResponse.SC_BAD_REQUEST, ex.getMessage());
		return new ModelAndView();
	}
	
	//响应码
	public static final int SC_BAD_REQUEST = 400;
```

为什么要存在这个异常处理器呢？

从框架的设计理念来看，这种公共的、常见的异常应该交给框架本身来完成，是一些必需处理的异常。比如参数类型转换异常，如果程序员不处理，还有框架提供默认的处理方式，**不至于出现这种错误而无法排查**。

### SimpleMappingExceptionResolver

这种异常处理器需要提前配置异常类和对应的`view`视图。一般用于使用`JSP`的项目中，出现异常则通过这个异常处理器跳转到指定的页面。

怎么配置？首先搭建`JSP`项目我就不浪费篇幅介绍了。首先要加载一个`XML`文件。

```java
@SpringBootApplication
//在启动类，加载配置文件
@ImportResource("classpath:spring-config.xml")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

然后在`resources`目录下，创建一个`spring-config.xml`文件，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean class="org.springframework.web.servlet.handler.SimpleMappingExceptionResolver">
        <!-- 定义默认的异常处理页面 -->
        <property name="defaultErrorView" value="err"/>
        <!-- 定义异常处理页面用来获取异常信息的变量名，默认名为exception -->
        <property name="exceptionAttribute" value="ex"/>
        <!-- 定义需要特殊处理的异常，用类名或完全路径名作为key，异常也页名作为值 -->
        <property name="exceptionMappings">
            <props>
                <!-- 数组越界异常 -->
                <prop key="java.lang.ArrayIndexOutOfBoundsException">err/arrayIndexOutOfBounds</prop>
                <!-- 空指针异常 -->
                <prop key="java.lang.NullPointerException">err/nullPointer</prop>
            </props>
        </property>
    </bean>
</beans>
```

然后在`webapp`也就是存放`JSP`页面的目录下，创建两个`JSP`页面。

`arrayIndexOutOfBounds.jsp`如下：

```JSP
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>数组越界异常</title>
</head>
<body>
<h1>数组越界异常</h1>
<br>
<%-- 打印异常到页面上 --%>
<% Exception ex = (Exception)request.getAttribute("ex"); %>
<br>
<div><%= ex.getMessage() %></div>
<% ex.printStackTrace(new java.io.PrintWriter(out)); %>
</body>
</html>
```

`nullPointer.jsp`如下：

```JSP
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>空指针异常</title>
</head>
<body>
<h1>空指针异常</h1>
<br>
<%-- 打印异常到页面上 --%>
<% Exception ex = (Exception)request.getAttribute("ex"); %>
<br>
<div><%=ex.getMessage()%></div>
<% ex.printStackTrace(new java.io.PrintWriter(out)); %>
</body>
</html>
```

接着创建两个`Controller`，分别抛出空指针异常和数组越界异常。

```java
@Controller
@RequestMapping("/error")
public class ErrController {

    @RequestMapping("/null")
    public String err() throws Exception{
        String str = null;
        //抛出空指针异常
        int length = str.length();
        System.out.println(length);
        return "index";
    }

    @RequestMapping("/indexOut")
    public String indexOut() throws Exception{
        int[] nums = new int[2];
        for (int i = 0; i < 3; i++) {
            //抛出数组越界异常
            nums[i] = i;
            System.out.println(nums[i]);
        }
        return "index";
    }
}
```

启动项目后，我们发送两个请求，就可以看到：

![](https://static.lovebilibili.com/exceptionResolver_2.png)

![](https://static.lovebilibili.com/exceptionResolver_3.png)

通过上述例子可以看出，其实对于现在**前后端分离的项目**来说，**这种异常处理器已经不是很常用了**。

### ResponseStatusExceptionResolver

这种异常处理器主要用于处理带有`@ResponseStatus`注释的异常。下面演示一下使用方式。

首先自定义异常类继承`Exception`，并且使用`@ResponseStatus`注解修饰。如下：

```java
//value需要使用HttpStatus枚举类型，HttpStatus.FORBIDDEN=403。
@ResponseStatus(value = HttpStatus.FORBIDDEN,reason = "My defined Exception")
public class DefinedException extends Exception{
}
```

然后再在`Controller`层抛出此异常。如下：

```java
@Controller
@RequestMapping("/error")
public class ErrController {
    @RequestMapping("/myException")
    public String ex(@RequestParam(name = "num") Integer num) throws Exception {
        if (num == 1) {
            //抛出自定义异常
            throw new DefinedException();
        }
        return "index";
    }
}
```

然后启动项目，请求接口，可以看到如下信息：

![](https://static.lovebilibili.com/exceptionResolver_4.png)

使用这种异常处理器，需要自定义一个异常，一定要一直往上层抛出异常，如果不往上层抛出，在`service`或者`dao`层就`try-catch`处理掉的话，是不会触发的。

### ExceptionHandlerExceptionResolver

这个异常处理器才是最重要的，也是最常用，最灵活的，因为是使用注解。首先我们还是简单地演示一下怎么使用：

首先需要定义一个全局的异常处理器。

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

然后我们自定义一个自定义异常类`BaseException`：

```java
public class BaseException extends Exception {
    public BaseException(String message) {
        super(message);
    }
}
```

然后在`Controller`层定义一个方法测试：

```java
@Controller
@RequestMapping("/error")
public class ErrController {
    @RequestMapping("/base")
    public String base() throws BaseException {
        throw new BaseException("系统异常，请稍后重试。");
    }
}
```

老规矩，启动项目，请求接口可以看到结果：

![](https://static.lovebilibili.com/exceptionResolver_1.jpg)

你也可以不自定义异常`BaseException`，而直接拦截常见的各种异常都可以。所以这是一个非常灵活的异常处理器。你也可以做跳转页面，返回`ModelAndView`即可（以免篇幅过长就不演示了，哈哈）。

### 小结

经过以上的演示后我们学习了`SpringMVC`四种异常处理器的工作机制，最后这种作为程序员我觉得是必须掌握的，前面的简单映射异常处理器和状态映射处理器可以选择性掌握，默认的异常处理器了解即可。

那这么多异常处理器，究竟是如何工作的呢？为什么是设计一个接口，下面有一个抽象类加上四个实现子类呢？接下来我们通过源码分析来揭开谜底！

## 源码分析

源码分析从哪里入手呢？在`SpringMVC`中，其实你想都不用想，肯定在`DispatcherServlet`类里。经过我顺藤摸瓜，我定位在了`processHandlerException()`方法。怎么定位的呢？其实很简单，看源码：

```java
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

		boolean errorView = false;
		//异常不为空
		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
                //关键点：执行异常处理
				mv = processHandlerException(request, response, handler, exception);
				//省略...
			}
		}
		//省略...
	}
```

### processHandlerException()

就是这个直接的一个`if-else`判断，那个`processHandlerException()`方法又是怎么处理的呢？

```java
@Nullable
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
      @Nullable Object handler, Exception ex) throws Exception {
   ModelAndView exMv = null;
   //判断异常处理器的集合是否为空
   if (this.handlerExceptionResolvers != null) {
      //不为空则遍历异常处理器 
      for (HandlerExceptionResolver handlerExceptionResolver : this.handlerExceptionResolvers) {
         //调用异常处理器的resolveException()方法进行处理异常
         exMv = handlerExceptionResolver.resolveException(request, response, handler, ex);
         //判断返回的ModelAndView是否为null，不为null则跳出循环，为null则继续下一个异常处理器
         if (exMv != null) {
            break;
         }
      }
   }
   //如果ModelAndView不为空
   if (exMv != null) {
      if (exMv.isEmpty()) {
         //设置异常信息提示
         request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
         return null;
      }
      //如果返回的ModelAndView不包含view
      if (!exMv.hasView()) {
         //设置一个默认的视图 
         String defaultViewName = getDefaultViewName(request);
         if (defaultViewName != null) {
            exMv.setViewName(defaultViewName);
         }
      }
      //省略...
      //返回异常的ModelAndView	
      return exMv;
   }
   throw ex;
}
```

这不就是责任链模式吗！提前加载异常处理器到`handlerExceptionResolvers`集合中，然后遍历去执行，能处理就处理，不能处理就跳到下一个异常处理器处理。

那接下来我们就有一个问题了，`handlerExceptionResolvers`集合是怎么加载异常处理器的？这个问题很简单，就是使用`DispatcherServlet.properties`配置文件。这个文件真的很重要！！！

```properties
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
```

默认是加载以上三种异常处理器到集合中，所以只要带有`@ControllerAdvice`、`@ExceptionHandler`、`@ResponseStatus`注解的都会被扫描。`SimpleMappingExceptionResolver`则是通过`xml`文件(当然也可以使用`@Configuration`)去配置。

### resolveException()

其实在`resolveException()`处理异常的方法中，还使用了模板模式。

```java
	@Override
	@Nullable
	public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response,
			@Nullable Object handler, Exception ex) {
			//省略...
        	//预处理
			prepareResponse(ex, response);
        	//调用了一个抽象方法，抽象方法由子类去实现
			ModelAndView result = doResolveException(request, response, handler, ex);
			//省略...
	}
```

抽象方法`doResolveException()`，由子类实现。

```java
@Nullable
protected abstract ModelAndView doResolveException(HttpServletRequest request,
      HttpServletResponse response, @Nullable Object handler, Exception ex);
```

怎么识别模板方法，其实很简单，只要看到抽象类，有个具体方法里面调用了抽象方法，那很大可能就是模板模式。抽象方法就是模板方法，由子类实现。

子类我们都知道就是那四个异常处理器实现类了。

### 总结

用流程图概括一下：

![](https://static.lovebilibili.com/exceptionResolver_5.png)

经过以上的学习后，我们知道只需要把异常处理器加到集合中，就可以执行。所以我们可以使用直接实现`HandlerExceptionResolver`接口的方式来实现异常处理器。

## 实现HandlerExceptionResolver接口实现全局异常处理

首先自定一个异常类`MyException`。

```java
public class MyException extends Exception {
    public MyException(String message) {
        super(message);
    }
}
```

然后实现`HandlerExceptionResolver`接口定义一个异常处理器。

```java
//注册异常处理器到Spring容器中
@Component
public class MyExceptionHandler implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try {
            //如果属于MyException异常，则输出异常提示到页面
            if (ex instanceof MyException) {
                response.setContentType("text/html;charset=utf-8");
                response.getWriter().println(ex.getMessage());
                //这里返回null，不做处理。也可以返回ModelAndView跳转页面
                return null;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

然后在`Controller`层定义一个方法测试：

```java
@Controller
@RequestMapping("/error")
public class ErrController {
    @RequestMapping("/myEx")
    public String myEx() throws MyException {
        System.out.println("执行myEx()");
        throw new MyException("自定义异常提示信息");
    }
}
```

启动项目，请求接口，我们可以看到：

![](https://static.lovebilibili.com/exceptionResolver_6.png)

# 最后说几句

以上就是我对于`SpringMVC`全局异常处理机制的理解。更多的`java`技术分享，可以关注我的公众号“**java技术爱好者**”，后续会不断更新。
