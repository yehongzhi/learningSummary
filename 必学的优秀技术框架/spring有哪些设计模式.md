# 思维导图

![](https://static.lovebilibili.com/spring_design_mode_swdt.png)

> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 概述

一个优秀的框架肯定离不开各种设计模式的运用，Spring框架也不例外。因为网上很多文章比较散乱，所以想总结一下在Spring中用到的设计模式，希望大家看完之后能对spring有更深层次的理解。

# 工厂模式

工厂模式我们都知道是把创建对象交给工厂，以此来降低类与类之间的耦合。工厂模式在Spring中的应用非常广泛，这里举的例子是ApplicationContext和BeanFactory，这也是Spring的IOC容器的基础。

首先看BeanFactory，这是最底层的接口。

```java
public interface BeanFactory {
    Object getBean(String name) throws BeansException;
    
    <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException;
    
    Object getBean(String name, Object... args) throws BeansException;
    
    <T> T getBean(Class<T> requiredType) throws BeansException;
    
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
    //省略...
}
```

ApplicationContext则是扩展类，也是一个接口，他的作用是当容器启动时，一次性创建所有的bean。

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
    @Nullable
    String getId();

    String getApplicationName();

    String getDisplayName();

    long getStartupDate();

    @Nullable
    ApplicationContext getParent();

    AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;
}
```

ApplicationContext有三个实现类，分别是：

- ClassPathXmlApplication：从类路径ClassPath中寻找指定的XML配置文件，找到并装载ApplicationContext的实例化工作。
- FileSystemXMLApplicationContext：从指定的文件系统路径中寻找指定的XML配置文件，找到并装载ApplicationContext的实例化工作。
- XMLWebApplicationContext：从Web系统中的XML文件载入Bean定义的信息，Web应用寻找指定的XML配置文件，找到并装载完成ApplicationContext的实例化工作。

因此这几个类的关系我们清楚了，类图就是这样：

![](https://static.lovebilibili.com/ApplicationContext.png)

在哪里初始化呢，这讲起来有些复杂，就不展开细讲，提一下。主要看AbstractApplicationContext类的refresh()方法。

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        //省略...
        try {
            //省略...
            //初始化所有的单实例 Bean(没有配置赖加载的)
            finishBeanFactoryInitialization(beanFactory);
        }catch (BeansException ex) {
            //省略...
        }finally {
            //省略...
        }
    }
}
```

# 单例模式

在系统中，有很多对象我们都只需要一个，比如线程池、Spring的上下文对象，日志对象等等。单例模式的好处在于**对一些重量级的对象，省略了创建对象花费的时间，减少了系统的开销**，第二点是使用单例**可以减少new操作的次数，减少了GC线程回收内存的压力**。

实际上，在Spring中的Bean默认的作用域就是**singleton**(单例)的。如何实现的呢？

主要看DefaultSingletonBeanRegistry的getSingleton()方法：

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    /** 保存单例Objects的缓存集合ConcurrentHashMap，key：beanName --> value：bean实例 */
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "Bean name must not be null");
        synchronized (this.singletonObjects) {
            //检查缓存中是否有实例，如果缓存中有实例，直接返回
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                //省略...
                try {
                    //通过singletonFactory获取单例
                    singletonObject = singletonFactory.getObject();
                    newSingleton = true;
                }
                //省略...
                if (newSingleton) {
                    addSingleton(beanName, singletonObject);
                }
            }
            //返回实例
            return singletonObject;
        }
    }
    
    protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
}
```

从源码中可以看出，是通过ConcurrentHashMap的方式，如果在Map中存在则直接返回，如果不存在则创建，并且put进Map集合中，并且整段逻辑是使用同步代码块包住的，所以是线程安全的。

# 策略模式

策略模式，简单来说就是封装好一组策略算法，外部客户端根据不同的条件选择不同的策略算法解决问题。这在很多框架，还有日常开发都会用到的一种设计模式。在Spring中，我这里举的例子是Resource类，这是所有资源访问类所实现的接口。

针对不同的访问资源的方式，Spring定义了不同的Resource类的实现类。我们看一张类图：

![](https://static.lovebilibili.com/Resource.png)

简单介绍一下Resource的实现类：

- **UrlResource**：访问网络资源的实现类。
- **ServletContextResource**：访问相对于 ServletContext 路径里的资源的实现类。
- **ByteArrayResource**：访问字节数组资源的实现类。
- **PathResource**：访问文件路径资源的实现类。
- **ClassPathResource**：访问类加载路径里资源的实现类。

写一段伪代码来示范一下Resource类的使用：

```java
@RequestMapping(value = "/resource", method = RequestMethod.GET)
public String resource(@RequestParam(name = "type") String type,
                       @RequestParam(name = "arg") String arg) throws Exception {
    Resource resource;
    //这里可以优化为通过工厂模式，根据type创建Resource的实现类
    if ("classpath".equals(type)) {
        //classpath下的资源
        resource = new ClassPathResource(arg);
    } else if ("file".equals(type)) {
        //本地文件系统的资源
        resource = new PathResource(arg);
    } else if ("url".equals(type)) {
        //网络资源
        resource = new UrlResource(arg);
    } else {
        return "fail";
    }
    InputStream is = resource.getInputStream();
    ByteArrayOutputStream os = new ByteArrayOutputStream();
    int i;
    while ((i = is.read()) != -1) {
        os.write(i);
    }
    String result = new String(os.toByteArray(), StandardCharsets.UTF_8);
    is.close();
    os.close();
    return "type:" + type + ",arg:" + arg + "\r\n" + result;
}
```

这就是策略模式的思想，通过外部条件使用不同的算法解决问题。其实很简单，因为每个实现类的getInputStream()方法都不一样，我们看ClassPathResource的源码，是通过类加载器加载资源：

```java
public class ClassPathResource extends AbstractFileResolvingResource {

	private final String path;

	@Nullable
	private ClassLoader classLoader;

	@Nullable
	private Class<?> clazz;
    
    @Override
    public InputStream getInputStream() throws IOException {
        InputStream is;
        //通过类加载器加载类路径下的资源
        if (this.clazz != null) {
            is = this.clazz.getResourceAsStream(this.path);
        }
        else if (this.classLoader != null) {
            is = this.classLoader.getResourceAsStream(this.path);
        }
        else {
            is = ClassLoader.getSystemResourceAsStream(this.path);
        }
        //如果输入流is为null，则报错
        if (is == null) {
            throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
        }
        //返回InputStream
        return is;
    }
}
```

再看UrlResource的源码，获取InputStream的实现又是另一种策略。

```java
public class UrlResource extends AbstractFileResolvingResource {
	@Nullable
	private final URI uri;

	private final URL url;

	private final URL cleanedUrl;
    
    @Override
	public InputStream getInputStream() throws IOException {
        //获取连接
		URLConnection con = this.url.openConnection();
		ResourceUtils.useCachesIfNecessary(con);
		try {
            //获取输入流，并返回
			return con.getInputStream();
		}
		catch (IOException ex) {
			// Close the HTTP connection (if applicable).
			if (con instanceof HttpURLConnection) {
				((HttpURLConnection) con).disconnect();
			}
			throw ex;
		}
	}
}
```

# 代理模式

Spring除了IOC(控制反转)之外的另一个核心就是AOP(面向切面编程)。AOP能够将与业务无关的，却被业务模块所共同调用的逻辑(**比如日志，权限控制等等**)封装起来，减少系统重复代码，降低系统之间的耦合，有利于系统的维护和扩展。

Spring AOP主要是基于动态代理实现的，如果要代理的类，实现了某个接口，则使用JDK动态代理，如果没有实现接口则使用Cglib动态代理。

我们看DefaultAopProxyFactory的createAopProxy()方法，Spring通过此方法创建动态代理类：

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
    @Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " + "Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
}
```

JDK动态代理和Cglib动态代理的区别：

- JDK动态代理只能对实现了接口的类生成代理，没有实现接口的类不能使用。
- Cglib动态代理即使被代理的类没有实现接口，也可以使用，因为Cglib动态代理是使用继承被代理类的方式进行扩展。
- Cglib动态代理是通过继承的方式，覆盖被代理类的方法来进行代理，所以如果方法是被final修饰的话，就不能进行代理。

从源码中可以看出，Spring会先判断是否实现了接口，如果实现了接口就使用JDK动态代理，如果没有实现接口则使用Cglib动态代理，也可以通过配置，强制使用Cglib动态代理，配置如下：

```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

# 模板模式

模板模式在Spring中用得太多了，它定义一个算法的骨架，而将一些步骤延迟到子类中。 一般定义一个抽象类为骨架，子类重写抽象类中的模板方法实现算法骨架中特定的步骤。模板模式可以不改变一个算法的结构即可重新定义该算法的某些特定步骤。

Spring中的事务管理器就运用模板模式的设计，首先看PlatformTransactionManager类。这是最底层的接口，定义提交和回滚的方法。

```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
    
    void commit(TransactionStatus status) throws TransactionException;
    
    void rollback(TransactionStatus status) throws TransactionException;
}
```

毫无意外，使用了抽象类作为骨架，接着看AbstractPlatformTransactionManager类。

```java
@Override
public final void commit(TransactionStatus status) throws TransactionException {
    //省略...
    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    if (defStatus.isLocalRollbackOnly()) {
        //省略...
        //调用processRollback()
        processRollback(defStatus, false);
        return;
    }

    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        //省略...
        //调用processRollback()
        processRollback(defStatus, true);
        return;
    }
    //调用processCommit()
    processCommit(defStatus);
}

//这个方法定义了骨架，里面会调用一个doRollback()的模板方法
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
    if (status.hasSavepoint()) {
        //省略...
    }
    else if (status.isNewTransaction()) {
        //调用doRollback()模板方法
        doRollback(status);
    }
    else {
        //省略...
    }
    //省略了很多代码...
}

private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    //省略...
    if (status.hasSavepoint()) {
        //省略...
    }
    else if (status.isNewTransaction()) {
        //省略...
        //调用doCommit()模板方法
        doCommit(status);
    }
    else if (isFailEarlyOnGlobalRollbackOnly()) {
        unexpectedRollback = status.isGlobalRollbackOnly();
    }
    //省略了很多代码...
}

//模板方法doRollback()，把重要的步骤延迟到子类去实现
protected abstract void doRollback(DefaultTransactionStatus status) throws TransactionException;

//模板方法doCommit()，把重要的步骤延迟到子类去实现
protected abstract void doCommit(DefaultTransactionStatus status) throws TransactionException;
```

模板方法则由各种事务管理器的实现类去实现，也就是把骨架中重要的doRollback()延迟到子类。一般来说，Spring默认是使用的事务管理器的实现类是DataSourceTransactionManager。

```java
//通过继承AbstractPlatformTransactionManager抽象类
public class DataSourceTransactionManager extends AbstractPlatformTransactionManager
		implements ResourceTransactionManager, InitializingBean {
    //重写doCommit()方法，实现具体commit的逻辑
    @Override
    protected void doCommit(DefaultTransactionStatus status) {
        DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
        Connection con = txObject.getConnectionHolder().getConnection();
        if (status.isDebug()) {
            logger.debug("Committing JDBC transaction on Connection [" + con + "]");
        }
        try {
            con.commit();
        }
        catch (SQLException ex) {
            throw new TransactionSystemException("Could not commit JDBC transaction", ex);
        }
    }
    
    //重写doRollback()方法，实现具体的rollback的逻辑
    @Override
	protected void doRollback(DefaultTransactionStatus status) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
		Connection con = txObject.getConnectionHolder().getConnection();
		if (status.isDebug()) {
			logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
		}
		try {
			con.rollback();
		}
		catch (SQLException ex) {
			throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
		}
	}
}
```

如果你是用Hibernate框架，Hibernate也有自身的实现，这就体现了设计模式的开闭原则，通过继承或者组合的方式进行扩展，而不是直接修改类的代码。Hibernate的事务管理器则是HibernateTransactionManager。

```java
public class HibernateTransactionManager extends AbstractPlatformTransactionManager
		implements ResourceTransactionManager, BeanFactoryAware, InitializingBean {
    
    //重写doCommit()方法，实现Hibernate的具体commit的逻辑
    @Override
	protected void doCommit(DefaultTransactionStatus status) {
		HibernateTransactionObject txObject = (HibernateTransactionObject) status.getTransaction();
		Transaction hibTx = txObject.getSessionHolder().getTransaction();
		Assert.state(hibTx != null, "No Hibernate transaction");
		if (status.isDebug()) {
			logger.debug("Committing Hibernate transaction on Session [" +
					txObject.getSessionHolder().getSession() + "]");
		}
		try {
			hibTx.commit();
		}
		catch (org.hibernate.TransactionException ex) {
			throw new TransactionSystemException("Could not commit Hibernate transaction", ex);
		}
        //省略...
	}
    
    //重写doRollback()方法，实现Hibernate的具体rollback的逻辑
    @Override
	protected void doRollback(DefaultTransactionStatus status) {
		HibernateTransactionObject txObject = (HibernateTransactionObject) status.getTransaction();
		Transaction hibTx = txObject.getSessionHolder().getTransaction();
		Assert.state(hibTx != null, "No Hibernate transaction");
		//省略...
		try {
			hibTx.rollback();
		}
		catch (org.hibernate.TransactionException ex) {
			throw new TransactionSystemException("Could not roll back Hibernate transaction", ex);
		}
        //省略...
		finally {
			if (!txObject.isNewSession() && !this.hibernateManagedSession) {
				txObject.getSessionHolder().getSession().clear();
			}
		}
	}
}
```

其实模板模式在日常开发中也经常用，比如一个方法中，前后代码都一样，只有中间有一部分操作不同，就可以使用模板模式进行优化代码，这可以大大地减少冗余的代码，非常实用。

# 适配器模式与责任链模式

适配器模式是一种结构型设计模式， 它**能使接口不兼容的对象能够相互合作，将一个类的接口，转换成客户期望的另外一个接口**。

在SpringAOP中有一个很重要的功能就是使用的 Advice（通知） 来增强被代理类的功能，Advice主要有MethodBeforeAdvice、AfterReturningAdvice、ThrowsAdvice这几种。每个Advice都有对应的拦截器，如下所示：

![](https://static.lovebilibili.com/spring_design_mode_01.png)

Spring需要将每个 Advice 都封装成对应的拦截器类型返回给容器，所以**需要使用适配器模式对 Advice 进行转换**。对应的就有三个适配器，我们看个类图：

![](https://static.lovebilibili.com/AdvisorAdapter.png)

适配器在Spring中是怎么把通知类和拦截类进行转换的呢，我们先看适配器的接口。定义了两个方法，分别是supportsAdvice()和getInterceptor()。

```java
public interface AdvisorAdapter {
    //判断通知类是否匹配
    boolean supportsAdvice(Advice advice);
    //传入通知类，返回对应的拦截类
    MethodInterceptor getInterceptor(Advisor advisor);
}
```

其实很简单，可以看出转换的方法就是getInterceptor()，通过supportsAdvice()进行判断。我们看前置通知的适配器的实现类MethodBeforeAdviceAdapter。

```java
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {
    //判断是否匹配MethodBeforeAdvice通知类
	@Override
	public boolean supportsAdvice(Advice advice) {
		return (advice instanceof MethodBeforeAdvice);
	}
	//传入MethodBeforeAdvice，转换为MethodBeforeAdviceInterceptor拦截类
	@Override
	public MethodInterceptor getInterceptor(Advisor advisor) {
		MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
		return new MethodBeforeAdviceInterceptor(advice);
	}
}
```

getInterceptor()方法中，调用了对应的拦截类的构造器创建对应的拦截器返回，传入通知类advice作为参数。接着我们看拦截器MethodBeforeAdviceInterceptor。

```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {
	//成员变量，通知类
    private MethodBeforeAdvice advice;
    
	//定义了有参构造器，外部通过有参构造器创建MethodBeforeAdviceInterceptor
    public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
        Assert.notNull(advice, "Advice must not be null");
        this.advice = advice;
    }
	//当调用拦截器的invoke方法时，就调用通知类的before()方法，实现前置通知
    @Override
    public Object invoke(MethodInvocation mi) throws Throwable {
        //调用通知类的before()方法，实现前置通知
        this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
        return mi.proceed();
    }

}
```

那么在哪里初始化这些适配器呢，我们看DefaultAdvisorAdapterRegistry()。

```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {

    private final List<AdvisorAdapter> adapters = new ArrayList<>(3);
	
    public DefaultAdvisorAdapterRegistry() {
        //初始化适配器，添加到adapters集合，也就是注册
		registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}
    
    @Override
	public void registerAdvisorAdapter(AdvisorAdapter adapter) {
		this.adapters.add(adapter);
	}
    
    //获取所有的拦截器
    @Override
    public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
        List<MethodInterceptor> interceptors = new ArrayList<>(3);
        Advice advice = advisor.getAdvice();
        if (advice instanceof MethodInterceptor) {
            interceptors.add((MethodInterceptor) advice);
        }
        //遍历adapters集合
        for (AdvisorAdapter adapter : this.adapters) {
            //调用supportsAdvice()方法，判断入参的advisor是否有匹配的适配器
            if (adapter.supportsAdvice(advice)) {
                //如果匹配，则调用getInterceptor()转换成对应的拦截器，添加到interceptors集合中
                interceptors.add(adapter.getInterceptor(advisor));
            }
        }
        if (interceptors.isEmpty()) {
            throw new UnknownAdviceTypeException(advisor.getAdvice());
        }
        //返回拦截器集合
        return interceptors.toArray(new MethodInterceptor[0]);
    }
    
}
```

适配器模式在这里就是把通知类转为拦截类，转为拦截类之后，就添加到拦截器集合中。添加到拦截器集合之后，就用到了责任链模式，在ReflectiveMethodInvocation类被调用，我们看JDK动态代理JdkDynamicAopProxy的invoke()方法。

```java
@Override
@Nullable
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    MethodInvocation invocation;
    //这里就是获取拦截器集合，最后就会调用到上文说的getInterceptors()
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
    if (chain.isEmpty()) {
        //省略...
    }else {
        //创建一个MethodInvocation
        invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
        //调用proceed()方法，底层会通过指针遍历拦截器集合，然后实现前置通知等功能
        retVal = invocation.proceed();
    }
    //省略...
}
```

最后就在ReflectiveMethodInvocation里调用proceed()方法，proceed()方法是一个递归的方法，通过指针控制递归的结束。这是很典型的责任链模式。

```java
public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable {
    protected final List<?> interceptorsAndDynamicMethodMatchers;
	//指针
    private int currentInterceptorIndex = -1;
    
    protected ReflectiveMethodInvocation(Object proxy, @Nullable Object target, Method method, @Nullable Object[] arguments, @Nullable Class<?> targetClass, List<Object> interceptorsAndDynamicMethodMatchers) {
		//省略...
        //拦截器的集合
        this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers;
    }
    
    @Override
    @Nullable
    public Object proceed() throws Throwable {
        //	We start with an index of -1 and increment early.
        if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
            //递归结束
            return invokeJoinpoint();
        }
		//获取拦截器，并且当前的指针+1
        Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
        if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
            InterceptorAndDynamicMethodMatcher dm =
                (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
            if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
                return dm.interceptor.invoke(this);
            }
            else {
				//匹配失败，跳过，递归下一个
                return proceed();
            }
        }
        else {
			//匹配拦截器，强转为拦截器，然后执行invoke()方法，然后就会调用拦截器里的成员变量的before()，afterReturning()等等，实现前置通知，后置通知，异常通知
            return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
        }
    }
}
```

这里可能没学过责任链模式的同学会看得有点晕，但是学过责任链模式应该很容易看懂，这其实跟SpringMVC的拦截器的逻辑实现几乎一样的。

# 观察者模式

观察者模式是一种对象行为型模式，当一个对象发生变化时，这个对象所依赖的对象也会做出反应。Spring 事件驱动模型就是观察者模式很经典的一个应用。

## 事件角色

在Spring事件驱动模型中，首先有事件角色ApplicationEvent，这是一个抽象类，抽象类下有四个实现类代表四种事件。

- ContextStartedEvent：ApplicationContext启动后触发的事件。
- ContextStoppedEvent：ApplicationContext停止后触发的事件。
- ContextRefreshedEvent：ApplicationContext初始化或刷新完成后触发的事件。
- ContextClosedEvent：ApplicationContext关闭后触发的事件。

![](https://static.lovebilibili.com/ApplicationEvent.png)

## 事件发布者

有了事件之后，需要有个发布者发布事件，发布者对应的类是ApplicationEventPublisher。

```java
@FunctionalInterface
public interface ApplicationEventPublisher {
    default void publishEvent(ApplicationEvent event) {
        publishEvent((Object) event);
    }
    
    void publishEvent(Object event);

}
```

@FunctionalInterface表示这是一个函数式接口，函数式接口只有一个抽象方法。ApplicationContext类又继承了

ApplicationEventPublisher类，所以我们可以使用ApplicationContext发布事件。

## 事件监听者

发布事件后需要有事件的监听者，事件监听者通过实现接口ApplicationListener来定义，这是一个函数式接口，并且带有泛型，要求E参数是ApplicationEvent的子类。

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	void onApplicationEvent(E event);
}
```

下面我们演示一下怎么使用，首先继承抽象类ApplicationEvent定义一个事件角色PayApplicationEvent。

```java
public class PayApplicationEvent extends ApplicationEvent {

    private String message;

    public PayApplicationEvent(Object source, String message) {
        super(source);
        this.message = message;
    }

    public String getMessage() {
        return message;
    }
}
```

接着定义一个PayApplicationEvent事件的监听者PayListener。

```java
@Component
public class PayListener implements ApplicationListener<PayApplicationEvent> {
    
    @Override
    public void onApplicationEvent(PayApplicationEvent event) {
        String message = event.getMessage();
        System.out.println("监听到PayApplicationEvent事件，消息为：" + message);
    }
}
```

最后我们使用ApplicationContext发布事件。

```java
@SpringBootApplication
public class SpringmvcApplication {

    public static void main(String[] args) throws Exception {
        ApplicationContext applicationContext = SpringApplication.run(SpringmvcApplication.class, args);
        applicationContext.publishEvent(new PayApplicationEvent(applicationContext,"成功支付100元！"));
    }

}
```

启动之后我们可以看到控制台打印：

![](https://static.lovebilibili.com/spring_design_mode_02.png)

## 絮叨

实际上，Spring中使用到的设计模式在源码中随处可见，并不止我列举的这些，所以Spring的源码非常值得去阅读和学习，受益良多。反过来看，如果不会设计模式，读起源码来也是非常费劲的，所以我建议还是先学会设计模式再去学习源码。

希望大家看完之后，能对Spring有更深入的了解，那么这篇文章就讲到这里了，感谢大家的阅读。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！