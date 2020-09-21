---
title: 适配器模式与SpringMVC
date: 2020-05-31 13:41:20
index_img: https://static.lovebilibili.com/adaptera_index.jpg
tags:
	- java
	- 设计模式
---

# 适配器模式

## 定义

适配器模式是将一个接口转换成客户希望的另一个接口，使接口不兼容的那些类可以一起工作，其别名为包装器(Wrapper)。适配器模式既可以作为类结构型模式，也可以作为对象结构型模式。

<!-- more -->

## 通俗解释

用生活中的例子就是充电器的转接头或者数据线转接头，也就是两个类不兼容的情况下，通过适配器类来做到兼容。

## 举个例子

我看了网上很多人的博客，关于适配器模式的一些例子，主要有两种，一种叫类适配器，一种叫对象适配器。写完这两个例子后，我有种恍然大悟的感觉！

### 类适配器

首先有一个接口是目标接口`PayService`，目标方法`pay()`。

```java
public interface PayService {
	
    String pay(String channel, String amount) throws Exception;
}
```

然后有一个被适配的类`CheckHelper`，适配方法`checkedPay()`

```java
public class CheckHelper {
    //检查支付渠道和支付金额
    public boolean checkedPay(String channel, String amount) {
        try {
            //字符串转成数字，如果出现转换异常返回fasle
            int mount = Integer.parseInt(amount);
            //PayEnum定义了一些支付渠道，比如支付宝、微信、银联等等
            List<String> channelList = Arrays.stream(PayEnum.values())
                .map(PayEnum::getChannel)
                .collect(Collectors.toList());
            //包含在支付渠道中，并且金额大于0，返回true，否则返回false
            return channelList.contains(channel) && mount > 0;
        } catch (Exception e) {
            return false;
        }
    }
}
```

需求是要使得在接口`PayService`调用`CheckHelper`的`checkedPay()`方法，现在使用类适配器的方式演示：

```java
public class PayAdapter extends CheckHelper implements PayService {
   
    @Override
    public String pay(String channel, String amount) throws Exception {
        boolean checked = super.checkedPay(channel, amount);
        if (!checked) {
            return "支付失败，支付参数有误";
        }
        return "支付成功，渠道为：" + channel + ",金额：" + amount;
    }
}
```

其实就是使用继承的方式来完成，适配器类继承`CheckHelper`类，然后使用`super`来调用被适配类

`CheckHelper`的`checkedPay()`方法，一目了然了。

### 对象适配器

明显使用类适配器的方式不太灵活，因为`java`是单继承，所以我们可以改成成员变量的方式，也就是对象适配器。代码如下：

```java
public class PayAdapter implements PayService {
	//使用成员变量
    private CheckHelper checkHelper = new CheckHelper();

    @Override
    public String pay(String channel, String amount) throws Exception {
        //调用CheckHelper的checkedPay()方法
        boolean checked = checkHelper.checkedPay(channel, amount);
        if (!checked) {
            return "支付失败，支付参数有误";
        }
        return "支付成功，渠道为：" + channel + ",金额：" + amount;
    }
}
```

那么肯定有人会说，你这样直接`new`一个对象不好，可以使用`SpringIOC`注入，于是又可以写成这样：

```java
//注册到Spring容器中
@Component("checkHelper")
public class CheckHelper {
    
}
```

```java
public class PayAdapter implements PayService {

    @Resource(name = "checkHelper")
    private CheckHelper checkHelper;

    @Override
    public String pay(String channel, String amount) throws Exception {
        boolean checked = checkHelper.checkedPay(channel, amount);
        if (!checked) {
            return "支付失败，支付参数有误";
        }
        return "支付成功，渠道为：" + channel + ",金额：" + amount;
    }
}
```

然后有人可能已经开始察觉了，这不就是平时我们使用的依赖注入吗？没错！所以我开始就说了，写完这两个例子后，我恍然大悟了。原来适配器模式我们一直都在用，只是没认出来罢了。

## 总结一下

那么我们用适配器模式有什么优点呢？为什么要这样写：

1.解耦，降低了对象与对象之间的耦合性。

2.增加了类的复用，这点是比较重要的。

3.灵活性和扩展性都非常好，通过使用配置文件，可以很方便地更换适配器，也可以在不修改原有代码的基础上增加新的适配器类，完全符合“开闭原则”。这点我待会在下面`SpringMVC`的应用中详细说明。

## 在SpringMVC中的应用

我们都知道`SpringMVC`定义一个映射的方式很简单，使用`@RequestMapping`注解，如下所示：

```java
@RestController
public class PayController {
    @RequestMapping("/pay")
    public String pay(String channel,String amount)throws Exception{
        return "";
    }
}
```

实际上除了上面这种常用的方式外，还有其他的方式定义：

> 实现`Controller`接口

```java
@org.springframework.stereotype.Controller("/path")
public class TestController implements Controller {
    @Override
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        return null;
    }
}
```

> 实现`HttpRequestHandler `接口

```java
@Controller("/httpPath")
public class HttpController implements HttpRequestHandler {

    @Override
    public void handleRequest(HttpServletRequest request,
                              HttpServletResponse response
    ) throws ServletException, IOException {
        //业务处理，页面跳转，返回响应结果等等
    }
}
```

> 实现`Servlet`接口

```java
@Controller("/servletPath")
public class ServletController implements Servlet {
    //Servlet生命周期函数
    //重写init()方法
  	//重写getServletConfig()方法

    @Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
		//业务处理
    }

    //重写getServletInfo()方法
    //重写destroy()方法
}
```

还要配置一个`SimpleServletHandlerAdapter`适配器的`bean`，因为默认只加载前面三种适配器，所以这种适配器需要自己手动添加。从这里也可以看出`SpringMVC`已经不推荐这种创建方式。

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

> `HandlerFunction`接口，关于响应式接口的开发

最后一种是使用`HandlerFunction`函数式接口，这是`Spring5.0`后引入的方式，主要用于做响应式接口的开发，这里就不举例子了。后面我会写一篇文章再详述。

**问题：**以上就有五种方式定义`Mapping`映射，那么`SpringMVC`是如何去适配的呢？并且具有良好的扩展性和维护性呢？

## 源码分析

首先我们把目光放在`DispatcherServlet`类的`doDispatch()`方法

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}
				
                //重点： 获取到对应的适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
                //省略...
                //重点： 调用HandlerAdapter接口的handle()方法，得到ModelAndView结果
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
				//省略...
			}
			catch (Exception ex) {
				//省略...
			}
			catch (Throwable err) {
				//省略..
			}
		}
	}
```

先不要慌张，其实学过策略模式你一眼就可以看出来，实际上这里就是运用了类似于策略模式的方式，根据不同的对象获取到对应的适配器，然后执行`HandlerAdapter`接口的`handle()`方法得到结果。

关键是这个`getHandlerAdapter()`方法，是怎么获取到对应的`HandlerAdapter`。

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
            //这个handlerAdapters有全部的适配器，遍历handlerAdapters集合
			for (HandlerAdapter adapter : this.handlerAdapters) {
                //如果匹配
				if (adapter.supports(handler)) {
                    //就返回这个适配器
					return adapter;
				}
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```

那么你看到上面这个`this.handlerAdapters`肯定会有疑问，`handlerAdapters`集合里面的适配器是什么时候初始化的？哪里初始化？继续看。

在`DispatcherServlet`的`initStrategies()`方法中有一堆初始化方法。

```java
protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
    	//这个就是初始化适配器的方法，handlerAdapters就是在这里初始化的
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```

接着我们看`initHandlerAdapters()`方法

```java
private void initHandlerAdapters(ApplicationContext context) {
		this.handlerAdapters = null;
		//省略...
		//如果为null，刚开始当然为null，所以加载handlerAdapters集合
		if (this.handlerAdapters == null) {
            //关键又在于getDefaultStrategies方法
			this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No HandlerAdapters found in servlet '" + getServletName() + "': using default");
			}
		}
	}
```

然后我们又去`getDefaultStrategies()`方法中看你会发现：

```java
    protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
		String key = strategyInterface.getName();
        //defaultStrategies中获取值，key就是HandlerAdapter.class对象
		String value = defaultStrategies.getProperty(key);
    	//省略...
	}
```

然后重点就在于这个`defaultStrategies`对象。我们继续看，很快看到了。

```java
	//DispatcherServlet.properties文件名
	private static final String DEFAULT_STRATEGIES_PATH = "DispatcherServlet.properties";
	//Properties对象，全局变量
	private static final Properties defaultStrategies;

	static {
		try {
			ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
            //加载DispatcherServlet.properties文件
			defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
		}
		catch (IOException ex) {
			throw new IllegalStateException("Could not load '" + DEFAULT_STRATEGIES_PATH + "': " + ex.getMessage());
		}
	}
```

所以明显可以看到所有的适配器类都是写在`DispatcherServlet.properties`文件里了！默认加载这三种适配器。

```properties
org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
```

### 得到结论：

适配器实现类是从`DispatcherServlet.properties`文件加载到内存中的。

## HandlerAdapter接口

所以关键在于`HandlerAdapter`接口，接口信息如下：

```java
public interface HandlerAdapter {
	
    //子类去实现，用于判断上级接口
	boolean supports(Object handler);
	
    //子类实现这个方法，返回响应的结果
	@Nullable
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
	
    //判断是否使用浏览器缓存，返回-1表示不使用浏览器缓存
	long getLastModified(HttpServletRequest request, Object handler);

}
```

学过策略模式的应该很清楚了，上面讲过有5种方式定义`Mapping`。

所以应该可以猜测`HandlerAdapter`接口有五个子类。打开类图：

![](https://static.lovebilibili.com/HandlerAdapter.png)

果然是有五个实现的子类分别对应五种方式！

那么我们找其中一个实现类，比如最简单的`SimpleControllerHandlerAdapter`，来分析一下：

```java
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

	@Override
    //getHandlerAdapter()方法就会调用这个方法判断，然后返回对应的适配器实现类
    //这里返回的就是SimpleControllerHandlerAdapter适配器
	public boolean supports(Object handler) {
		return (handler instanceof Controller);
	}

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		//执行Controller接口的handleRequest，也就是mapping映射的方法
		return ((Controller) handler).handleRequest(request, response);
	}
	
    //判断是否使用浏览器缓存，返回-1表示不使用浏览器缓存
	@Override
	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}

}
```

下面画一张图来总结一下以上的分析过程：

![](https://static.lovebilibili.com/adapterProcessPic.png)

这不就像策略模式吗...只能解释为设计模式有很多都比较类似。假设`SpringMVC`要增加一种定义`Mapping`的方式，那就很容易了，增加对应的适配器实现类，对原有的代码没有任何的侵入，这就非常符合开闭原则。接下来我们就对适配器进行扩展，自定义一个适配器。

## 自定义SpringMVC适配器

首先要定义一个适配器`MyHandlerAdapter`，实现`HandlerAdapter`接口。

```java
public class MyHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return handler instanceof MyController;
    }

    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        return ((MyController) handler).handleRequest(request, response);
    }

    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        //不使用浏览器缓存，返回-1
        return -1;
    }
}
```

接着定义一个`MyController`接口。

```java
public interface MyController {
    /**
     * 处理请求
     */
    @Nullable
    ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception;
}
```

注册适配器到`Spring`容器中。

```java
@Configuration
@EnableWebMvc
public class WebMvcConfig extends WebMvcConfigurerAdapter {
	//注册自定义的适配器
    @Bean
    public MyHandlerAdapter myHandlerAdapter() {
        return new MyHandlerAdapter();
    }
}
```

最后创建一个`MyTestController`实现`MyController`进行测试。

```java
@Controller("/myTest")
public class MyTestController implements MyController {

    @Override
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        response.getWriter().println("MyTestController Test success!!!");
        return null;
    }
}
```

启动项目，然后在浏览器输入访问地址，即可看到。

![](https://static.lovebilibili.com/adapter_test.png)

当你理解透彻之后，你就可以这样自定义一个适配器，来加深一下理解，验证之前的分析的正确性。

沉下心学习，才能跑得更快！

以上就是适配器模式的学习，更多的java技术分享，就关注**java技术爱好者**吧！

<img src="https://me.lovebilibili.com/img/wechat.jpg-slim" alt="100" style="zoom:50%;" />

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！