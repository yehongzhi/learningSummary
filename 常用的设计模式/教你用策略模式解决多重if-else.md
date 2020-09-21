---

title: 教你用策略模式解决多重if-else
date: 2020-04-05 14:12:40
index_img: https://static.lovebilibili.com/strategy_index.jpg
tags:
	- 设计模式
	- java
---

# 写在前面

很多人可能在公司就是做普通的CRUD的业务，对于设计模式，即使学了好像也用处不大，顶多就在面试的时候能说上几种常见的单例模式，工厂模式。而在实际开发中，设计模式似乎很难用起来。

在现在的环境下，程序员的竞争已经非常激烈了，要体现出自身的价值，最直接的体现当然是差异化。这无需多说，我认为在实际开发中能运用设计模式，是很能体现差异化的。设计模式是一些前人总结的较好的方法，使程序能有更好的扩展性，可读性，维护性。

下面举个例子，使用策略模式解决多重if-else的代码结构。想学习更多的设计模式的实战经验，那就点个关注吧，谢谢大佬。

# 使用if-else

假设我们要开发一个支付接口，要对接多种支付方式，通过渠道码区分各种的支付方式。于是定义一个枚举`PayEnum`，如下：

```java
public enum PayEnum {
    ALI_PAY("ali","支付宝支付"),
    WECHAT_PAY("wechat","微信支付"),
    UNION_PAY("union","银联支付"),
    XIAO_MI_PAY("xiaomi","小米支付");
    /**渠道*/
    private String channel;
    /**描述*/
    private String description;
    PayEnum(String channel, String description) {
        this.channel = channel;
        this.description = description;
    }
    /**以下省略字段的get、set方法*/
```

创建一个`PayController`类，代码如下：

```java
@RestController
@RequestMapping("/xiaoniu")
public class PayController {
    @Resource(name = "payService")
    private PayService payService;
    /**
    * 支付接口
    * @param channel 渠道
    * @param amount  消费金额
    * @return String 返回消费结果
    * @author Ye hongzhi
    * @date 2020/4/5
    */
    @RequestMapping("/pay")
    public String pay(@RequestParam(name = "channel") String channel,
                      @RequestParam(name = "amount") String amount
    )throws Exception{
        return payService.pay(channel,amount);
    }
}
```

再创建一个`PayService`接口以及实现类`PayServiceImpl`

```java
public interface PayService {
    /**
    * 支付接口
    * @param channel 渠道
    * @param amount  金额
    * @return String
    * @author Ye hongzhi
    * @date 2020/4/5
    */
    String pay(String channel,String amount)throws Exception;
}
```

```java
@Service("payService")
public class PayServiceImpl implements PayService {
    private static String MSG = "使用 %s ,消费了 %s 元";
    @Override
    public String pay(String channel, String amount) throws Exception {
        if (PayEnum.ALI_PAY.getChannel().equals(channel)) {
            //支付宝
            //业务代码...
            return String.format(MSG,PayEnum.ALI_PAY.getDescription(),amount);
        }else if(PayEnum.WECHAT_PAY.getChannel().equals(channel)){
            //微信支付
            //业务代码...
            return String.format(MSG,PayEnum.WECHAT_PAY.getDescription(),amount);
        }else if(PayEnum.UNION_PAY.getChannel().equals(channel)){
            //银联支付
            //业务代码...
            return 		String.format(MSG,PayEnum.UNION_PAY.getDescription(),amount);
        }else if(PayEnum.XIAO_MI_PAY.getChannel().equals(channel)){
            //小米支付
            //业务代码...
            return String.format(MSG,PayEnum.XIAO_MI_PAY.getDescription(),amount);
        }else{
            return "输入渠道码有误";
        }
    }
}
```

然后通过浏览器，我们可以看到效果

![](https://static.lovebilibili.com/01.png)

![](https://static.lovebilibili.com/02.png)

这样看，以上代码的确可以实现需求，通过渠道码区分支付方式，可是看到上面那么多达4个的`if-else`的代码结构，已经开始显示出问题了。假设有更多的支付方式，那么这段代码就要写更多的`else if`去判断，这显然会不利于代码的扩展，这样会导致这个支付的方法越写越长。

在设计模式六大原则中，其中一个原则叫做`开闭原则`，对扩展开放，对修改关闭，应尽量在不修改原有代码的情况下进行扩展。

基于上面提到的`开闭原则`，我们可以使用策略模式进行重构。

# 使用策略模式重构代码

定义一个策略接口类`PayStrategy`

```java
public interface PayStrategy {
    String MSG = "使用 %s ,消费了 %s 元";
    String pay(String channel,String amount)throws Exception;
}
```

然后再创建四种策略实现类实现接口

```java
@Component("aliPayStrategy")
public class AliPayStrategyImpl implements PayStrategy{
    @Override
    public String pay(String channel, String amount) throws Exception {
        return String.format(MSG, PayEnum.ALI_PAY.getDescription(),amount);
    }
}
```

```java
@Component("wechatPayStrategy")
public class WechatPayStrategyImpl implements PayStrategy{
    @Override
    public String pay(String channel, String amount) throws Exception {
        return String.format(MSG, PayEnum.WECHAT_PAY.getDescription(),amount);
    }
}
```

```java
@Component("unionPayStrategy")
public class UnionPayStrategyImpl implements PayStrategy{
    @Override
    public String pay(String channel, String amount) throws Exception {
        return String.format(MSG, PayEnum.UNION_PAY.getDescription(),amount);
    }
}
```

```java
@Component("xiaomiPayStrategy")
public class XiaomiPayStrategyImpl implements PayStrategy{
    @Override
    public String pay(String channel, String amount) throws Exception {
        return String.format(MSG, PayEnum.XIAO_MI_PAY.getDescription(),amount);
    }
}
```

看到这里实际上已经很清晰了，思路就是通过渠道码，动态获取到具体的实现类，这样就可以实现不需要`if else`判断。怎么通过渠道码获取实现类呢？

在`PayEnum`枚举加上`BeanName`字段，然后增加一个通过渠道码获取`BeanName`的方法

 ```java
	ALI_PAY("ali","支付宝支付","aliPayStrategy"),
    WECHAT_PAY("wechat","微信支付","wechatPayStrategy"),
    UNION_PAY("union","银联支付","unionPayStrategy"),
    XIAO_MI_PAY("xiaomi","小米支付","xiaomiPayStrategy");
	/**策略实现类对应的 beanName*/
    private String beanName;
	/**
     * 通过渠道码获取枚举
     * */
    public static PayEnum findPayEnumBychannel(String channel){
        PayEnum[] enums = PayEnum.values();
        for (PayEnum payEnum : enums){
            if(payEnum.getChannel().equals(channel)){
                return payEnum;
            }
        }
        return null;
    }
	//构造器
    PayEnum(String channel, String description, String beanName) {
        this.channel = channel;
        this.description = description;
        this.beanName = beanName;
    }
 ```

这时候还差一个获取Spring上下文对象的工具类，于是我们创建一个`SpringContextUtil`类

```java
@Component
public class SpringContextUtil implements ApplicationContextAware {
	/**
     * 上下文对象实例
     */
    private static ApplicationContext applicationContext;
    /**
     * 获取applicationContext
     */
    private static ApplicationContext getApplicationContext() {
        return applicationContext;
    }
    /**
     * 通过name获取Bean
     * */
    public static Object getBean(String name){
        return getApplicationContext().getBean(name);
    }
    /**
     * 通过name,以及Clazz返回指定的Bean
     * */
    public static <T> T getBean(String name,Class<T> clazz){
        return getApplicationContext().getBean(name,clazz);
    }
    @Override
    @Autowired
    public void setApplicationContext(ApplicationContext applicationContext) throws 		BeansException {
        SpringContextUtil.applicationContext = applicationContext;
    }
```

接着定义一个工厂类，通过渠道码获取对应的策略实现类

```java
public class PayStrategyFactory {
    /**
     * 通过渠道码获取支付策略具体实现类
     * */
    public static PayStrategy getPayStrategy(String channel){
        PayEnum payEnum = PayEnum.findPayEnumBychannel(channel);
        if(payEnum == null){
            return null;
        }
        return SpringContextUtil.getBean(payEnum.getBeanName(),PayStrategy.class);
    }
}
```

最后我们再改造一下原来的`PayServiceImpl`的`pay`方法

```java
@Override
public String pay(String channel, String amount) throws Exception {
    PayStrategy payStrategy = PayStrategyFactory.getPayStrategy(channel);
    if(payStrategy == null){
        return "输入渠道码有误";
    }
    return payStrategy.pay(channel,amount);
}
```

哇喔！突然间代码就显得清爽很多了！

小伙伴们看到这里，get到新的技能了吗？

> 假设需要增加新的支付方式，就不需要再使用else if 去判断，而是在枚举中定义一个新的枚举对象，然后再增加一个策略实现类，实现对应的方法，那就可以很轻松地扩展。也实现了开闭原则。

# 写在最后

设计模式运用得熟练的话，很多代码可以写得很优雅。更多的设计模式实战经验的分享，就关注java技术小牛吧。

<img src="https://me.lovebilibili.com/img/wechat.jpg-slim" alt="100" style="zoom:50%;" />

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！