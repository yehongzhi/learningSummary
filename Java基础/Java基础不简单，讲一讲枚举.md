> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 什么是枚举

枚举是JDK1.5新增的一种数据类型，是一种特殊的类，常用于表示一组常量，比如一年四季，12个月份，星期一到星期天，服务返回的错误码，结算支付的方式等等。枚举是使用enum关键字来定义。

# 枚举的使用

在使用枚举之前我们先探讨一个问题，为什么要使用枚举。

现在有个业务场景是结算支付，有支付宝和微信支付两种方式，1表示支付宝，2表示微信支付，还需要根据编码(1或2)获取相应的英文名，如果不用枚举，我们就要这样写。

```java
public class PayTypeUtil {
    //支付宝
    private static final int ALI_PAY = 1;
    //微信支付
    private static final int WECHAT_PAY = 2;

    //根据编码获取支付方式的名称
    public String getPayName(int code) {
        if (ALI_PAY == code) {
            return "Ali_Pay";
        }
        if (WECHAT_PAY == code) {
            return "Wechat_Pay";
        }
        return null;
    }
}
```

如果这时，产品经理说要增加一个银联支付，就要加多if的判断，就会造成有多少种支付方式，就有多少个`if`，非常难看。

如果使用枚举，就变得很优雅，先看代码：

```java
public enum PayTypeEnum {
    /** 支付宝*/
    ALI_PAY(1, "ALI_PAY"),
    /** 微信支付*/
    WECHAT_PAY(2, "WECHAT_PAY");

    private int code;

    private String describe;

    PayTypeEnum(int code, String describe) {
        this.code = code;
        this.describe = describe;
    }
    //根据编码获取支付方式
    public PayTypeEnum find(int code) {
        for (PayTypeEnum payTypeEnum : values()) {
            if (payTypeEnum.getCode() == code) {
                return payTypeEnum;
            }
        }
        return null;
    }
    //getter、setter方法
}
```

当我们需要扩展，只需要定义多一个实例即可，其他代码都不用动，比如加多一个银联支付。

```java
/** 支付宝*/
ALI_PAY(1, "ALI_PAY"),
/** 微信支付*/
WECHAT_PAY(2, "WECHAT_PAY"),
//只需要加多一行代码即可完成扩展
/** 银联支付*/
UNION_PAY(3,"UNION_PAY");
```

一般在实际项目中，最多的写法就是这样，主要是简单明了，易于扩展。

第二种常见的用法是结合switch-case使用，比如我定义一个一年四季的枚举。

```java
public enum Season {
    //春
    SPRING,
    //夏
    SUMMER, 
    //秋
    AUTUMN, 
    //冬
    WINTER;
}
```

然后结合switch使用。

```java
public static void main(String[] args) throws Exception{
    doSomething(Season.SPRING);
}

private static void doSomething(Season season){
    switch (season){
        case SPRING:
            System.out.println("不知细叶谁裁出，二月春风似剪刀");
            break;
        case SUMMER:
            System.out.println("接天莲叶无穷碧，映日荷花别样红");
            break;
        case AUTUMN:
            System.out.println("停车坐爱枫林晚，霜叶红于二月花");
            break;
        case WINTER:
            System.out.println("梅花香自苦寒来，宝剑锋从磨砺出");
            break;
        default:
            System.out.println("垂死病中惊坐起，笑问客从何处来");
    }
}
```

可能很多人觉得直接用int，String类型配合switch使用就够了，为什么还要支持枚举，这样的设计是不是显得很多余，其实非也。

不妨反过来想，假如用1到4代表四季，接收的参数类型就是int，在没有提示的情况下，我们仅仅只知道数int类型是很难猜到需要传入数字的范围，字符串也是一样，如果不用枚举你是很难一眼看出需要传入什么参数，这才是最关键的。

如果使用枚举，那么问题就迎刃而解，当你调用doSomething()方法时，一看到枚举就知道传入的是哪几个参数，因为已经在枚举类里面定义好了。**这对于项目交接，还有代码的可读性都是非常有利的**。

这种限制不单止限制了调用方，也限制了传入的参数只能是定义好的枚举，不用担心传入的参数错误导致的程序错误。

所以枚举类使用得恰当，对于项目的可维护性是有很大提升的。

# 枚举本身的方法

首先我们先以上面的支付类型枚举PayTypeEnum为例子，看看有哪些自带的方法。

## valueOf()方法

这是一个静态方法，传入一个字符串(枚举的名称)，获取枚举类。如果传入的名称不存在，则报错。

```java
public static void main(String[] args) throws Exception{
    System.out.println(PayTypeEnum.valueOf("ALI_PAY"));
    System.out.println(PayTypeEnum.valueOf("HUAWEI_PAY"));
}
```

![](https://static.lovebilibili.com/enum_1.png)

## values()方法

返回包含枚举类中所有枚举数据的一个数组。

```java
public static void main(String[] args) throws Exception {
    PayTypeEnum[] payTypeEnums = PayTypeEnum.values();
    for (PayTypeEnum payTypeEnum : payTypeEnums) {
        System.out.println("code: " + payTypeEnum.getCode() + ",describe: " + payTypeEnum.getDescribe());
    }
}
```

![](https://static.lovebilibili.com/enum_2.png)

## ordinal()方法

默认情况下，枚举类会给定义的枚举提供一个默认的次序，ordinal()方法就可以返回枚举的次序。

```java
public static void main(String[] args) throws Exception {
    PayTypeEnum[] payTypeEnums = PayTypeEnum.values();
    for (PayTypeEnum payTypeEnum : payTypeEnums) {
        System.out.println("ordinal: " + payTypeEnum.ordinal() + ", Enum: " + payTypeEnum);
    }
}
/**
ordinal: 0, Enum: ALI_PAY
ordinal: 1, Enum: WECHAT_PAY
ordinal: 2, Enum: UNION_PAY
*/
```

## name()、toString()方法

返回定义枚举用的名称。

```java
public static void main(String[] args) throws Exception {
    for (Season season : Season.values()) {
        System.out.println(season.name());
    }
    for (Season season : Season.values()) {
        System.out.println(season.toString());
    }
}
```

输出结果都是一样的：

```java
SPRING
SUMMER
AUTUMN
WINTER
```

为什么？因为底层代码是一样，返回的是name。

```java
public abstract class Enum<E extends Enum<E>> implements Comparable<E>, Serializable {
    
    public final String name() {
        return name;
    }
    
    public String toString() {
        return name;
    }
}
	
```

区别在于toString()方法没有被final修饰，可以重写，name()方法不能重写。

## compareTo()方法

因为枚举类实现了Comparable接口，所以必须重写compareTo()方法，比较的是枚举的次序，也就是ordinal，源码如下：

```java
public final int compareTo(E o) {
    Enum<?> other = (Enum<?>)o;
    Enum<E> self = this;
    if (self.getClass() != other.getClass() && // optimization
        self.getDeclaringClass() != other.getDeclaringClass())
        throw new ClassCastException();
    return self.ordinal - other.ordinal;
}
```

因为实现Comparable接口，所以可以用来排序，比如这样：

```java
public static void main(String[] args) throws Exception {
    //这里是乱序的枚举数组
    Season[] seasons = new Season[]{Season.WINTER, Season.AUTUMN, Season.SPRING, Season.SUMMER};
    //调用sort方法排序，按默认次序排序
    Arrays.sort(seasons);
    for (Season season : seasons) {
        System.out.println(season);
    }
}
```

输出结果，按照默认次序排序：

```java
SPRING
SUMMER
AUTUMN
WINTER
```

# 原理

以枚举Season为例，分析一下枚举的底层。表面上看，一个枚举很简单：

```java
public enum Season {
    //春
    SPRING,
    //夏
    SUMMER,
    //秋
    AUTUMN,
    //冬
    WINTER;
}
```

实际上编译器在编译的时候做了很多动作，我们使用`javap -v`对Season.class文件反编译，可以看到很多细节。

首先我们看到枚举是继承了抽象类Enum的类。

```java
Season extends java.lang.Enum<Season>
```

第二，通过一段静态代码块初始化枚举。

```java
  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=4, locals=0, args_size=0
         0: new           #4                  // class io/github/yehongzhi/user/redisLock/Season
         3: dup
         4: ldc           #7                  // String SPRING
         6: iconst_0
         7: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
        10: putstatic     #9                  // Field SPRING:Lio/github/yehongzhi/user/redisLock/Season;
        13: new           #4                  // class io/github/yehongzhi/user/redisLock/Season
        16: dup
        17: ldc           #10                 // String SUMMER
        19: iconst_1
        20: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
        23: putstatic     #11                 // Field SUMMER:Lio/github/yehongzhi/user/redisLock/Season;
        26: new           #4                  // class io/github/yehongzhi/user/redisLock/Season
        29: dup
        30: ldc           #12                 // String AUTUMN
        32: iconst_2
        33: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
        36: putstatic     #13                 // Field AUTUMN:Lio/github/yehongzhi/user/redisLock/Season;
        39: new           #4                  // class io/github/yehongzhi/user/redisLock/Season
        42: dup
        43: ldc           #14                 // String WINTER
        45: iconst_3
        46: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
        49: putstatic     #15                 // Field WINTER:Lio/github/yehongzhi/user/redisLock/Season;
        52: iconst_4
        53: anewarray     #4                  // class io/github/yehongzhi/user/redisLock/Season
        56: dup
        57: iconst_0
        58: getstatic     #9                  // Field SPRING:Lio/github/yehongzhi/user/redisLock/Season;
        61: aastore
        62: dup
        63: iconst_1
        64: getstatic     #11                 // Field SUMMER:Lio/github/yehongzhi/user/redisLock/Season;
        67: aastore
        68: dup
        69: iconst_2
        70: getstatic     #13                 // Field AUTUMN:Lio/github/yehongzhi/user/redisLock/Season;
        73: aastore
        74: dup
        75: iconst_3
        76: getstatic     #15                 // Field WINTER:Lio/github/yehongzhi/user/redisLock/Season;
        79: aastore
        80: putstatic     #1                  // Field $VALUES:[Lio/github/yehongzhi/user/redisLock/Season;
        83: return
```

这段静态代码块的作用就是生成四个静态常量字段的值，还生成了$VALUES字段，用于保存枚举类定义的枚举常量。相当于执行了以下代码：

```java
Season SPRING = new Season1();
Season SUMMER = new Season2();
Season AUTUMN = new Season3();
Season WINTER = new Season4();
Season[] $VALUES = new Season[4];
$VALUES[0] = SPRING;
$VALUES[1] = SUMMER;
$VALUES[2] = AUTUMN;
$VALUES[3] = WINTER;
```

第三个，关于values()方法，这是一个静态方法，作用是返回该枚举类的数组，底层实现原理，其实是这样的。

```java
public static io.github.yehongzhi.user.redisLock.Season[] values();
    Code:
       0: getstatic     #1                  // Field $VALUES:[Lio/github/yehongzhi/user/redisLock/Season;
       3: invokevirtual #2                  // Method "[Lio/github/yehongzhi/user/redisLock/Season;".clone:()Ljava/lang/Object;
       6: checkcast     #3                  // class "[Lio/github/yehongzhi/user/redisLock/Season;"
       9: areturn
```

其实是将静态代码块初始化的$VALUES数组克隆一份，然后强转成Season[]返回。相当于这样：

```java
public static Season[] values(){
	return (Season[])$VALUES.clone();
}
```

所以表面上，只是加了一个enum关键字定义枚举，但是底层一旦确认是枚举类，则会由编译器对枚举类进行特殊处理，通过静态代码块初始化枚举，只要是枚举就一定会提供values()方法。

通过反编译我们也知道所有的枚举父类都是抽象类Enum，所以Enum有的成员变量，实现的接口，子类也会有。

所以只要是枚举都会有name，ordinal这两个字段，并且我们看Enum的构造器。

```java
/**
* Sole constructor.  Programmers cannot invoke this constructor.
* It is for use by code emitted by the compiler in response to
* enum type declarations.
*/
protected Enum(String name, int ordinal) {
    this.name = name;
    this.ordinal = ordinal;
}
```

翻译一下上面那段英文，意思大概是：唯一的构造器，程序员没法调用此构造器，它是供编译器响应枚举类型声明而使用的。得出结论，枚举实例的创建也是由编译器完成的。

# 枚举实现单例

很多人都说，枚举类是最好的实现单例的一种方式，因为枚举类的单例是线程安全，并且是唯一一种不会被破坏的单例模式实现。也就是不能通过反射的方式创建实例，保证了整个应用中只有一个实例，非常硬核的单例。

```java
public class SingletonObj {
    //内部类使用枚举
    private enum SingletonEnum {
        INSTANCE;

        private SingletonObj singletonObj;
		//在枚举类的构造器里初始化singletonObj
        SingletonEnum() {
            singletonObj = new SingletonObj();
        }

        private SingletonObj getSingletonObj() {
            return singletonObj;
        }
    }

    //对外部提供的获取单例的方法
    public static SingletonObj getInstance() {
        //获取单例对象，返回
        return SingletonEnum.INSTANCE.getSingletonObj();
    }

    //测试
    public static void main(String[] args) {
        SingletonObj a = SingletonObj.getInstance();
        SingletonObj b = SingletonObj.getInstance();
        System.out.println(a == b);//true
    }
}
```

假如有人想通过反射创建枚举类呢，我们以Season枚举为例。

```java
public static void main(String[] args) throws Exception {
    Constructor<Season> constructor = Season.class.getDeclaredConstructor(String.class, int.class);
    constructor.setAccessible(true);
    //通过反射调用构造器，创建枚举
    Season season = constructor.newInstance("NEW_SPRING", 4);
    System.out.println(season);
}
```

然后就会报错，因为不允许对枚举的构造器使用反射调用。

![](https://static.lovebilibili.com/enum_3.png)

查看源码，就可以看到，有个专门针对枚举的`if`判断。

```java
public T newInstance(Object ... initargs) throws InstantiationException, IllegalAccessException,IllegalArgumentException, InvocationTargetException {
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, null, modifiers);
        }
    }
    //判断是否是枚举，如果是枚举的话，报、抛出异常
    if ((clazz.getModifiers() & Modifier.ENUM) != 0)
        //抛出异常，不能通过反射创建枚举
        throw new IllegalArgumentException("Cannot reflectively create enum objects");
    ConstructorAccessor ca = constructorAccessor;   // read volatile
    if (ca == null) {
        ca = acquireConstructorAccessor();
    }
    @SuppressWarnings("unchecked")
    T inst = (T) ca.newInstance(initargs);
    return inst;
}
```

# 总结

枚举看起来好像是很小一部分的知识，其实深入挖掘的话，我们会发现还是有很多地方值得学习的。第一点使用枚举定义常量更容易扩展，而且代码可读性更强，维护性更好。接着第二点是需要了解枚举自带的方法。第三点通过反编译，探索编译器在编译阶段为枚举做了什么事情。最后再讲一下枚举实现单例模式的例子。

这篇文章讲到这里了，感谢大家的阅读，希望看完这篇文章能有所收获！

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！