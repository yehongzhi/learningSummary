# 思维导图

![](https://static.lovebilibili.com/mybatis_swdt.png)

> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 概述

Mybatis是一个比较主流的ORM框架，所以在日常工作中接触得很多。我比较喜欢看优秀框架的源码，因为能写出这种框架的作者肯定有其独特之处。如果能看懂源码的一些巧妙构思，一定是受益匪浅的。

所谓万事开头难，看源码也要找到切入的点。设计模式无疑是源码分析一个很好的切入点，废话不多说，那么我们就开始吧。

# 工厂模式

工厂模式属于创建型模式。工厂模式的作用是把创建对象的逻辑封装起来，提供一个接口给外部创建对象，降低类与类之间的耦合。

在Mybatis中，用到工厂模式主要在DataSourceFactory。这是一个负责创建DataSource数据源的工厂。DataSourceFactory是一个接口，有不同的子类实现，根据不同的配置，生成不同的DataSourceFactory实现类。类图如下：

![](https://static.lovebilibili.com/mybatis_dataSourceFactory.png)

接着我们看一下DataSourceFactory的源码：

```java
public interface DataSourceFactory {
  
  void setProperties(Properties props);

  DataSource getDataSource();

}
```

DataSourceFactory接口定义了两个抽象方法，怎么工作的呢，其实是跟dataSource标签的属性type有关。

```xml
<environment id="development">
    <transactionManager type="JDBC"/>
    <!-- type属性是关键属性 -->
    <dataSource type="POOLED">
        <property name="driver" value="${dataSource.driver}"/>
        <property name="url" value="${dataSource.url}"/>
        <property name="username" value="${dataSource.username}"/>
        <property name="password" value="${dataSource.password}"/>
    </dataSource>
</environment>
```

Mybatis内置的type有三种配置，分别是UNPOOLED，POOLED，JNDI。

**UNPOOLED**，这个数据源的实现只是每次被请求时打开和关闭连接。

**POOLED**，这种数据源的使用“池“的思想，避免了创建新的连接实例时所必需的初始化和认证时间。 

**JNDI**，这个数据源的实现是为了能在如 EJB 或应用服务器这类容器中使用，容器可以集中或在外部配置数据源，然后放置一个 JNDI 上下文的引用。

接着看XMLConfigBuilder的dataSourceElement()方法。

```java
private DataSourceFactory dataSourceElement(XNode context) throws Exception {
    if (context != null) {
        //读取配置文件中dataSource标签的属性type
        String type = context.getStringAttribute("type");
        Properties props = context.getChildrenAsProperties();
        //根据type属性的值，返回不同的子类，使用接口DataSourceFactory接收，体现了面向接口编程的思想
        DataSourceFactory factory = (DataSourceFactory) resolveClass(type).newInstance();
        //设置属性值，比如数据库的url，username，password等等
        factory.setProperties(props);
        //返回factory
        return factory;
    }
    throw new BuilderException("Environment declaration requires a DataSourceFactory.");
}
```

获得DataSourceFactory之后，就通过getDataSource()方法获取数据源，完事了。

```java
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
        if (environment == null) {
            environment = context.getStringAttribute("default");
        }
        //这里for循环主要是配置文件可以配置多个数据源
        for (XNode child : context.getChildren()) {
            String id = child.getStringAttribute("id");
            if (isSpecifiedEnvironment(id)) {
                TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                //dsFactory调动getDataSource()方法，创建dataSource对象
                DataSource dataSource = dsFactory.getDataSource();
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                    .transactionFactory(txFactory)
                    .dataSource(dataSource);
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
}
```

Mybatis使用工厂模式来创建DataSourceFactory，可以做到通过配置去使用不同的DataSourceFactory创建DataSource，非常灵活。在Mybatis中使用到工厂模式还有很多地方，比如SqlSessionFactory，这里就不再展开了，有兴趣的可以自己探索一下。

# 单例模式

单例模式属于创建型设计模式，这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。**保证在应用中只有单一对象被创建**。

Mybatis中用到单例模式的地方有很多，这里举个例子是ErrorContext类。这是一个用于记录该线程的执行环境错误信息，所以是在线程范围内的单例。

```java
public class ErrorContext {
    //使用ThreadLocal保存每个线程中的单例对象
    private static final ThreadLocal<ErrorContext> LOCAL = new ThreadLocal<ErrorContext>();
    //私有化构造器
    private ErrorContext() {
    }
    //向外提供唯一的接口获取单例对象
    public static ErrorContext instance() {
        //从LOCAL中取出context对象
        ErrorContext context = LOCAL.get();
        if (context == null) {
            //如果为null，new一个
            context = new ErrorContext();
            //放入到LOCAL中保存
            LOCAL.set(context);
        }
        //如果不为null，直接返回
        return context;
    }
}
```

# 建造者模式

建造者模式也是属于创建型模式，主要是在创建一个复杂对象时使用，通过一步一步构造最终的对象，将一个复杂对象的构建与它的表示分离。

在Mybatis中，使用到建造者模式的地方体现在ParameterMapping类，这是用于参数映射的一个类。

```java
public class ParameterMapping {
  private Configuration configuration;
  private String property;
  private ParameterMode mode;
  private Class<?> javaType = Object.class;
  private JdbcType jdbcType;
  private Integer numericScale;
  private TypeHandler<?> typeHandler;
  private String resultMapId;
  private String jdbcTypeName;
  private String expression;
  
  //私有化构造器
  private ParameterMapping() {
  }
  
  //通过内部类Builder创建对象
  public static class Builder {
      //初始化ParameterMapping实例
      private ParameterMapping parameterMapping = new ParameterMapping();
      //通过构造器初始化ParameterMapping的一些成员变量
      public Builder(Configuration configuration, String property, TypeHandler<?> typeHandler) {
          parameterMapping.configuration = configuration;
          parameterMapping.property = property;
          parameterMapping.typeHandler = typeHandler;
          parameterMapping.mode = ParameterMode.IN;
      }

      public Builder(Configuration configuration, String property, Class<?> javaType) {
          parameterMapping.configuration = configuration;
          parameterMapping.property = property;
          parameterMapping.javaType = javaType;
          parameterMapping.mode = ParameterMode.IN;
      }
      //设置parameterMapping的mode
      public Builder mode(ParameterMode mode) {
          parameterMapping.mode = mode;
          return this;
      }
      //设置parameterMapping的javaType
      public Builder javaType(Class<?> javaType) {
          parameterMapping.javaType = javaType;
          return this;
      }
      //设置parameterMapping的jdbcType
      public Builder jdbcType(JdbcType jdbcType) {
          parameterMapping.jdbcType = jdbcType;
          return this;
      }
      //设置parameterMapping的numericScale
      public Builder numericScale(Integer numericScale) {
          parameterMapping.numericScale = numericScale;
          return this;
      }
      //设置parameterMapping的resultMapId
      public Builder resultMapId(String resultMapId) {
          parameterMapping.resultMapId = resultMapId;
          return this;
      }
      //设置parameterMapping的typeHandler
      public Builder typeHandler(TypeHandler<?> typeHandler) {
          parameterMapping.typeHandler = typeHandler;
          return this;
      }
      //设置parameterMapping的jdbcTypeName
      public Builder jdbcTypeName(String jdbcTypeName) {
          parameterMapping.jdbcTypeName = jdbcTypeName;
          return this;
      }
      //设置parameterMapping的expression
      public Builder expression(String expression) {
          parameterMapping.expression = expression;
          return this;
      }
      //通过build()方法创建对象，返回
      public ParameterMapping build() {
          resolveTypeHandler();
          validate();
          return parameterMapping;
      }
  	}
}
```

在SqlSourceBuilder类的buildParameterMapping()方法中可以看到建造者模式的实战应用：

```java
//根据参数content，构建parameterMapping实例
private ParameterMapping buildParameterMapping(String content) {
    //属性值的Map集合
    Map<String, String> propertiesMap = parseParameterMapping(content);
    String property = propertiesMap.get("property");
    Class<?> propertyType;
    //省略...
    //创建一个ParameterMapping.Builder对象
    ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
	//省略...
    //这里遍历Map集合，把属性值设置到ParameterMapping对象中，并创建
    for (Map.Entry<String, String> entry : propertiesMap.entrySet()) {
        String name = entry.getKey();
        String value = entry.getValue();
        if ("javaType".equals(name)) {
            javaType = resolveClass(value);
            builder.javaType(javaType);
        } else if ("jdbcType".equals(name)) {
            builder.jdbcType(resolveJdbcType(value));
        } else if ("mode".equals(name)) {
            builder.mode(resolveParameterMode(value));
        } else if ("numericScale".equals(name)) {
            builder.numericScale(Integer.valueOf(value));
        } else if ("resultMap".equals(name)) {
            builder.resultMapId(value);
        } else if ("typeHandler".equals(name)) {
            typeHandlerAlias = value;
        } else if ("jdbcTypeName".equals(name)) {
            builder.jdbcTypeName(value);
        } else if ("property".equals(name)) {
            // Do Nothing
        } else if ("expression".equals(name)) {
            //抛出异常Expression based parameters are not supported yet
        } else {
            //抛出异常
        }
    }
    //省略...
    //创建ParameterMapping对象，并返回
    return builder.build();
}
```

# 模板模式

模板模式是一种行为型模式，一般用在一些比较通用的方法中，定义一个抽象类，编写一个算法的骨架，将一些步骤延迟到子类。就像是请假条一样，开头和结尾都是写好的模板，中间的请假的原因(内容)由请假人(子类)去补充完整，这样可以提高代码的复用。

在Mybatis中，模板模式体现在Executor和BaseExecutor这两个类中。首先看张类图：

![](https://static.lovebilibili.com/mybatis_executor.png)

Executor是一个接口，从命名上可以看出是用来执行SQL语句的对象。下面有一个BaseExecutor的抽象类，这就是用来定义模板方法的。再下面有三个实现类，**SimpleExecutor(简单执行器)，ReuseExecutor(重用执行器)，BatchExecutor(批量执行器)**。实现类就是用来填充模板中间的内容的。

执行器在执行JDBC操作的前后往往有很多需要处理的工作都是相同的，比如查询的时候使用缓存，更新时需要清除缓存等等，所以就很适合使用模板模式。

接着我们看BaseExecutor抽象类的源码，一看就明白了，其实就定义了一个骨架。

```java
public abstract class BaseExecutor implements Executor {
    //查询操作
    private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        List<E> list;
        localCache.putObject(key, EXECUTION_PLACEHOLDER);
        //上面的代码是固定的
        try {
            //这段代码不同的子类有不同的实现，所以是调用抽象方法doQuery()
            list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
        //下面的代码也是固定的
        } finally {
            //清除缓存
            localCache.removeObject(key);
        }
        //添加到缓存中
        localCache.putObject(key, list);
        if (ms.getStatementType() == StatementType.CALLABLE) {
            localOutputParameterCache.putObject(key, parameter);
        }
        //返回结果
        return list;
    }
    
    //抽象方法，由子类去实现
    protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
        throws SQLException;
}
```

那么就有疑问了，一开始如果都不设置的话，默认使用哪个子类的实现。很简单，直接跟着源码去顺藤摸瓜，我们就看到了。

```java
public class Configuration {
    //默认是SIMPLE，也就是SimpleExecutor
    protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
}
```

其实模板模式很简单就能“认出来”，我的理解就是，**抽象类里定义具(具体的方法)，具体方法再调抽(抽象方法)**。那十有八九就是模板模式了。

# 代理模式

代理模式属于结构型模式，代理模式的定义说的很抽象，为**其他对象提供一种代理以控制对这个对象的访问**。在某些情况下，一个对象不适合或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介的作用。

其实代理模式可以简单理解为中介的作用，比如一手房东只关心收租，其他的水电费结算，带人看房这些杂七杂八的东西他不想关心，就交给中介（二手房东），租客要租房就给钱中介，一手房东收钱就找中介，这个中介就是所谓的代理者。

回到Mybatis框架中，SqlSession类就用到代理模式，SqlSession是操作数据库一个会话对象，我们用户一般通过SqlSession做增删改查，但是如果每次做增删改都开启事务，关闭事务，显然是很麻烦，**所以就可以交给代理类来完成这个工作，如果没有开启事务，由代理类自动开启事务**。

Mybatis在这里是使用JDK动态代理，所以SqlSession是一个接口。

```java
public interface SqlSession extends Closeable {
    
    <T> T selectOne(String statement);

    <E> List<E> selectList(String statement);

    <E> List<E> selectList(String statement, Object parameter);

    <K, V> Map<K, V> selectMap(String statement, String mapKey);

    <T> Cursor<T> selectCursor(String statement);

    void select(String statement, Object parameter, ResultHandler handler);

    int insert(String statement);

    int update(String statement);

    int delete(String statement);

    void commit();

    void rollback();
    //省略...
}
```

我们知道动态代理要有一个实现InvocationHandler接口的类，这个类在SqlSessionManager里，是一个内部类，叫做SqlSessionInterceptor，在这个类里做相关的处理。

```java
private class SqlSessionInterceptor implements InvocationHandler {
    public SqlSessionInterceptor() {
        // Prevent Synthetic Access
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //获取SqlSession对象，localSqlSession是一个ThreadLocal，所以是每个线程有自己的sqlSession
        final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
        //如果不为null
        if (sqlSession != null) {
            try {
                //执行方法，返回结果  
                return method.invoke(sqlSession, args);
            } catch (Throwable t) {
                throw ExceptionUtil.unwrapThrowable(t);
            }
            //如果sqlsession为null
        } else {
            //打开session  
            final SqlSession autoSqlSession = openSession();
            try {
                //执行Sqlsession的方法，获得结果  
                final Object result = method.invoke(autoSqlSession, args);
                //commit提交事务
                autoSqlSession.commit();
                //返回结果
                return result;
            } catch (Throwable t) {
                //rollback回滚事务
                autoSqlSession.rollback();
                //抛出异常
                throw ExceptionUtil.unwrapThrowable(t);
            } finally {
                //关闭sqlsession
                autoSqlSession.close();
            }
        }
    }
}
```

然后通过构造器去初始化，提供静态方法返回这个SqlSessionManager实例，请看源码。

```java
//实现了SqlSessionFactory，SqlSession接口
public class SqlSessionManager implements SqlSessionFactory, SqlSession {

    private final SqlSessionFactory sqlSessionFactory;
    //代理类sqlSessionProxy
    private final SqlSession sqlSessionProxy;

    private SqlSessionManager(SqlSessionFactory sqlSessionFactory) {
        this.sqlSessionFactory = sqlSessionFactory;
        //初始化代理类
        this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[]{SqlSession.class},
            new SqlSessionInterceptor());
    }
    //提供一个静态方法给外部获取SqlSessionManager类，可以用SqlSession接收
    public static SqlSessionManager newInstance(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionManager(sqlSessionFactory);
    }
    
    //重写selectOne方法，使用代理类去执行
    @Override
    public <T> T selectOne(String statement) {
        return sqlSessionProxy.<T> selectOne(statement);
    }
    
    //重写insert方法，使用代理类去执行
    @Override
    public int insert(String statement) {
        return sqlSessionProxy.insert(statement);
    }
    //省略...
}
```

这就是Mybatis使用代理模式的一个例子，其实也不是很复杂，还是能看懂的。

但是上面这种方式一般很少用，我们一般都是使用Mapper接口的方式，其实Mapper接口的方式也是使用了代理模式，接下来再继续看。直接看MapperProxy类。

```java
//实现了InvocationHandler接口
public class MapperProxy<T> implements InvocationHandler, Serializable {

    private static final long serialVersionUID = -6424540398559729838L;
    private final SqlSession sqlSession;
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache;
	
    public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
    }
	//代理类做的什么事情，看这个方法
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		//省略...
        //获取mapperMethod对象
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        //执行方法。根据Mapper.xml配置文件的配置进行执行，返回结果
        return mapperMethod.execute(sqlSession, args);
    }

    private MapperMethod cachedMapperMethod(Method method) {
        //从methodCache取出mapperMethod
        MapperMethod mapperMethod = methodCache.get(method);
        //为null
        if (mapperMethod == null) {
            //new一个。这里已经把Mapper.xml的一些配置都封装好了
            mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
            //然后put进methodCache
            methodCache.put(method, mapperMethod);
        }
        //返回mapperMethod
        return mapperMethod;
    }
}
```

mapperMethod的execute()方法的作用就是根据Mapper接口的方法名找到Mapper.xml文件的sql的Id，然后执行相应的操作，返回结果。内部的代码比较长，但是思路就是这样，这里就不展开了。

所以我们平时用的时候，TbCommodityInfoMapper接口假设是这样定义了一个list()方法。

```java
public interface TbCommodityInfoMapper {
    //查询TbCommodityInfo列表
    List<TbCommodityInfo> list();
}
```

那个在TbCommodityInfoMapper.xml就要对应有一个id为list的配置。

```xml
<!-- 命名空间也要和接口的全限定名一致 -->
<mapper namespace="io.github.yehongzhi.commodity.mapper.TbCommodityInfoMapper">
    <!-- 必须要有一个对应的属性id为list的sql配置 -->
    <select id="list" resultType="tbCommodityInfo">
        select `id`, `commodity_name`, `commodity_price`, `description`, `number` from tb_commodity_info
    </select>
</mapper>
```

然后在MapperProxyFactory类，使用工厂模式提供获取代理类。

```java
public class MapperProxyFactory<T> {
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();
	//通过构造器初始化mapperInterface
    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    @SuppressWarnings("unchecked")
    protected T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
    }
	
    //通过这个方法，获取Mapper接口的代理类
    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }

}
```

那个这个getMapper的方法就在SqlSession的子类中被调用。

```java
@Override
public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
}
```

最终用户就是这样使用，这里是纯Mybatis，没有集成Spring的写法。

```java
public class Test {
    public static void main(String[] args) {
        //获取SqlSession对象
        SqlSession sqlSession = SqlSessionFactoryUtils.openSqlSession();
        //再获取Mapper接口的代理类
        TbCommodityInfoMapper tbCommodityInfoMapper = sqlSession.getMapper(TbCommodityInfoMapper.class);
        //通过代理类去执行相应的方法
        List<TbCommodityInfo> commodityInfoList = tbCommodityInfoMapper.list();
    }
}
```

所以代理模式可以说是Mybatis核心的设计模式，用的是非常巧妙。

# 装饰器模式

装饰器模式属于结构型模式，使用装饰类包装对象，动态地给对象添加一些额外的职责。那么在Mybatis中，哪个地方会使用到装饰器模式呢？

没错了，就是缓存。Mybatis有一级缓存和二级缓存的功能，一级缓存默认是打开的，范围在SqlSession中生效，二级缓存需要手动配置打开，范围在全局Configuration，在每个namespace中配置。二级缓存的类型有以下几种，请看配置。

```xml
<mapper namespace="io.github.yehongzhi.commodity.mapper.TbCommodityInfoMapper">
    <!--
        eviction:代表的是缓存回收策略，目前MyBatis提供以下策略。
        (1) LRU,最近最少使用的，一处最长时间不用的对象
        (2) FIFO,先进先出，按对象进入缓存的顺序来移除他们
        (3) SOFT,软引用，移除基于垃圾回收器状态和软引用规则的对象
        (4) WEAK,弱引用，更积极的移除基于垃圾收集器状态和弱引用规则的对象。这里采用的是LRU，移除最长时间不用的对形象
        flushInterval:刷新间隔时间，单位为毫秒，这里配置的是100秒刷新，如果你不配置它，那么当SQL被执行的时候才会去刷新缓存。
        size:引用数目，一个正整数，代表缓存最多可以存储多少个对象，不宜设置过大。设置过大会导致内存溢出。这里配置的是1024个对象
        readOnly:只读，意味着缓存数据只能读取而不能修改，这样设置的好处是我们可以快速读取缓存，缺点是我们没有办法修改缓存
    -->
    <cache eviction="LRU" flushInterval="100000" readOnly="true" size="1024"/>
    
    <select id="list" resultMap="base_column">
        select `id`, `commodity_name`, `commodity_price`, `description`, `number` from tb_commodity_info
    </select>
</mapper>
```

所以这个场景已经很清晰了，也就是在一级缓存的基础上，再添加二级缓存，也就符合装饰器模式的那句话，动态给对象添加职责功能。怎么做呢，我们不妨先看Cache类的类图。

![](https://static.lovebilibili.com/mybatis_cache.png)

先看Cache接口，其实就定义了一些接口方法。

```java
public interface Cache {

    String getId();

    void putObject(Object key, Object value);

    Object getObject(Object key);

    Object removeObject(Object key);

    void clear();

    int getSize();

    ReadWriteLock getReadWriteLock();
}
```

然后再看PerpetualCache类，这是Cache接口最基本的实现，二级缓存要扩展就在这个类上面再去包装来实现扩展。其实就是一个HashMap，再简单包装一下。

```java
public class PerpetualCache implements Cache {

    private final String id;
	//成员变量cache，创建一个HashMap
    private Map<Object, Object> cache = new HashMap<Object, Object>();

    public PerpetualCache(String id) {
        this.id = id;
    }

    @Override
    public String getId() {
        return id;
    }

    @Override
    public int getSize() {
        return cache.size();
    }

    @Override
    public void putObject(Object key, Object value) {
        cache.put(key, value);
    }

    @Override
    public Object getObject(Object key) {
        return cache.get(key);
    }

    @Override
    public Object removeObject(Object key) {
        return cache.remove(key);
    }

    @Override
    public void clear() {
        cache.clear();
    }

    @Override
    public ReadWriteLock getReadWriteLock() {
        return null;
    }
	//省略...
}
```

那么我们再看二级缓存的一个代表性的类LruCache。

````java
public class LruCache implements Cache {
	//一级缓存，保存在这个成员变量中
    private final Cache delegate;
    //实际上这是一个LinkedHashMap，利用LinkedHashMap的LRU算法实现缓存的LRU
    private Map<Object, Object> keyMap;
    private Object eldestKey;
	//构造器缓存对象，初始化
    public LruCache(Cache delegate) {
        this.delegate = delegate;
        setSize(1024);
    }

	//初始化keyMap，重写removeEldestEntry方法，实现LUR算法
    public void setSize(final int size) {
        keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
            private static final long serialVersionUID = 4267176411845948333L;

            @Override
            protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
                boolean tooBig = size() > size;
                //每次put进来时，eldestKey都是最老的key
                if (tooBig) {
                    eldestKey = eldest.getKey();
                }
                return tooBig;
            }
        };
    }

    @Override
    public void putObject(Object key, Object value) {
        //保存进缓存
        delegate.putObject(key, value);
        //这里就是删除掉不常用的值
        cycleKeyList(key);
    }

    @Override
    public Object getObject(Object key) {
        keyMap.get(key); //touch
        return delegate.getObject(key);
    }
    
    private void cycleKeyList(Object key) {
        keyMap.put(key, key);
        if (eldestKey != null) {
            //删除掉Map中最老的key
            delegate.removeObject(eldestKey);
            eldestKey = null;
        }
    }

}
````

关键在于成员变量delegate，这在其他的二级缓存装饰类中都定义了。这个是为了保存PerpetualCache这个基础缓存类的。所以这也就是说二级缓存是在PerpetualCache为基础扩展的。再继续看就更加明白了。

直接看创建SqlSessionFactory的builder方法，一直追踪下去，就可以找到MapperBuilderAssistant类的useNewCache方法。

```java
public class MapperBuilderAssistant extends BaseBuilder {
	//当前Mapper.xml的命名空间
    private String currentNamespace;

    public Cache useNewCache(Class<? extends Cache> typeClass,
                             Class<? extends Cache> evictionClass,
                             Long flushInterval,
                             Integer size,
                             boolean readWrite,
                             boolean blocking,
                             Properties props) {
        Cache cache = new CacheBuilder(currentNamespace)
            //设置基础缓存
            .implementation(valueOrDefault(typeClass, PerpetualCache.class))
            //添加缓存装饰类
            .addDecorator(valueOrDefault(evictionClass, LruCache.class))
            .clearInterval(flushInterval)
            .size(size)
            .readWrite(readWrite)
            .blocking(blocking)
            .properties(props)
            //创建缓存
            .build();
        //把缓存添加到configuration，所以二级缓存是configuration范围的
        configuration.addCache(cache);
        currentCache = cache;
        return cache;
    }
}
```

再看CacheBuilder的build方法，更加清晰了。

```java
public class CacheBuilder {
    
    private Class<? extends Cache> implementation;
    private final List<Class<? extends Cache>> decorators;
    
    public Cache build() {
        setDefaultImplementations();
        Cache cache = newBaseCacheInstance(implementation, id);
        setCacheProperties(cache);
        // issue #352, do not apply decorators to custom caches
        if (PerpetualCache.class.equals(cache.getClass())) {
            //遍历装饰器类
            for (Class<? extends Cache> decorator : decorators) {
                //在cache上添加二级缓存
                cache = newCacheDecoratorInstance(decorator, cache);
                setCacheProperties(cache);
            }
            cache = setStandardDecorators(cache);
        } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
            cache = new LoggingCache(cache);
        }
        //返回缓存
        return cache;
    }

    private void setDefaultImplementations() {
        if (implementation == null) {
            //如果为空，初始化为PerpetualCache
            implementation = PerpetualCache.class;
            //如果装饰器类为空，默认用LruCache
            if (decorators.isEmpty()) {
                decorators.add(LruCache.class);
            }
        }
    }
}
```

所以我们大概可以想到二级缓存就像千层饼一样，一层一层地包装起来。最后debug模式验证一下。

![](https://static.lovebilibili.com/mybatis_mode_1.png)

在执行查询的时候你就会看到真的是千层饼一样，一层一层的。看CachingExecutor类的query方法。

![](https://static.lovebilibili.com/mybatis_mode_2.png)

最后在调用Cache的putObject方法时就会一层一层从外到内地调用，实现为对象动态扩展功能的装饰器模式。

![](https://static.lovebilibili.com/mybatis_mode_3.png)

装饰器模式在Mybatis中的应用就讲到这里了，有什么不懂的，可以关注公众号`java技术爱好者`，加我微信提问。

# 总结

这篇文章就介绍了Mybatis中用到的6种设计模式，分别是**工厂模式，单例模式，模板模式，建造者模式，代理模式，还有装饰器模式**。实际上Mybatis除了我讲的这些之外，还有很多我没有提到的，比如组合模式，适配器模式等等，有兴趣自己去研究一下吧。

因为现在很多面试动不动就问有看过什么框架的源码，实际上看源码是好的，但是不能盲目入手，因为很多框架是运用了大量的设计模式，如果对设计模式没有一定的认识，很容易看不懂，看懵。所以对于还没有设计模式基础的同学，建议先看设计模式，然后再去学习源码，这样才能循序渐进地提升自身实力。

这篇文章就讲到这里了，感谢大家的阅读。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！