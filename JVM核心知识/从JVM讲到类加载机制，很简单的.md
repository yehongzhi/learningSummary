# 思维导图

![](https://static.lovebilibili.com/jvm_siweidaotu.png)

> **文章已收录到Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 一、JVM介绍

在介绍JVM之前，先看一下.java文件从编码到执行的过程：

![](https://static.lovebilibili.com/jvm_02.png)

整个过程是，x.java文件需要编译成x.class文件，通过类加载器加载到内存中，然后通过解释器或者即时编译器进行解释和编译，最后交给执行引擎执行，执行引擎操作OS硬件。

从**类加载器到执行引擎这块内容就是JVM**。

<!-- more -->

**JVM是一个跨语言的平台**。从上面的图中可以看到，实际上JVM上运行的不是.java文件，而是.class文件。这就引出一个观点，JVM是一个跨语言的平台，他不仅仅能跑java程序，只要这种编程语言能编译成JVM可识别的.class文件都可以在上面运行。

所以除了java以外，能在JVM上运行的语言有很多，比如JRuby、Groovy、Scala、Kotlin等等。

从本质上讲JVM就是一台通过软件虚拟的计算机，它有它自身的指令集，有它自身的操作系统。

所以Oracle给JVM定了一套JVM规范，Oracle公司也给出了他的实现。基本上是目前最多人使用的java虚拟机实现，叫做Hotspot。使用java -version可以查看：

![](https://static.lovebilibili.com/jvm_03.png)

一些体量较大，有一定规模的公司，也会开发自己的JVM虚拟机，比如淘宝的TaobaoVM、IBM公司的J9-IBM、微软的MicrosoftVM等等。

# 二、JDK、JRE、JVM

![](https://static.lovebilibili.com/jvm_04.png)

JVM应该很清楚了，是运行.class文件的虚拟机。JRE则是运行时环境，包括JVM和java核心类库，没有核心的类库是跑不起来的。

![](https://static.lovebilibili.com/jvm_05.png)

JDK则包括JRE和一些开发使用的工具集。

所以总的关系是**JDK > JRE > JVM**。

# 三、Class加载过程

类加载是JVM工作的一个很重要的过程，我们知道.class是存在在硬盘上的一个文件，如何加载到内存工作的呢，面试中也经常问这个问题。所以你要和其他程序员拉开差距，体现差异化，这个问题要搞懂。

类加载的过程实际上分为三大步：**Loading(加载)、Linking(连接)、Initlalizing(初始化)**。

其中第二步Linking又分为三小步：**Verification(验证)、Preparation(准备)、Resolution(解析)**。

![](https://static.lovebilibili.com/jvm_06.png)

## 3.1 Loading

Loading是**把.class字节码文件加载到内存中，并将这些数据转换成方法区中的运行时数据，在堆中生成一个java.lang.Class类对象代表这个类，作为方法区这些类型数据的访问入口**。

## 3.2 Linking

Linking简单来说，就是把原始的类定义的信息合并到JVM运行状态之中。分为三小步进行。

### 3.2.1 Verification

验证加载的类信息是否符合class文件的标准，防止恶意信息或者不符合规范的字节信息。是JVM虚拟机运行安全的重要保障。

### 3.2.2 Preparation

创建类或者接口中的**静态变量**，并初始化**静态变量赋默认值**。赋默认值不是赋初始值，比如static int i = 5，这一步只是把i赋值为0，而不是赋值为5。赋值为5是在后面的步骤。

### 3.2.3 Resolution

把class文件常量池里面用到的符号引用转换成直接内存地址，直接可以访问到的内容。

## 3.3 Initlalizing

这一步真正去执行类初始化clinit()(类构造器)的代码逻辑，包括静态字段赋值的动作，以及执行类定义中的静态代码块内(static{})的逻辑。当初始化一个类时，发现父类还没有进行过初始化，则先初始化父类。虚拟机会保证一个类的clinit()方法在多线程环境中被正确加锁和同步。

# 四、类加载器

上面就是类加载的整个过程。而最后一步Initlalizing是通过类加载器加载类。类加载器这里我单独讲一下，因为这是一个重点。

Java中的类加载器由上到下分为：

- Bootstrap ClassLoader（启动类加载器）
- ExtClassLoader（扩展类加载器）
- AppClassLoader（应用程序类加载器）

从类图，可以看到**ExtClassLoader和AppClassLoader都是ClassLoader的子类**。

![](https://static.lovebilibili.com/jvm_07.png)

所以如果要自定义一个类加载器，可以继承ClassLoader抽象类，重写里面的方法。重写什么方法后面再讲。

# 五、双亲委派机制

讲完类加载器，这些类加载器是怎么工作的呢。对于双亲委派机制可能多多少少有听过，没听过也没关系，我正要讲。

上面说过有Bootstrap，ExtClassLoader，AppClassLoader三个类加载器。工作机制如下：

![](https://static.lovebilibili.com/jvm_08.png)

加载类的逻辑是怎么样的呢，核心代码是可以在JDK源码中找到的，在抽象类ClassLoader类的loadClass()，有兴趣可以源码看看：

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    //如果上层有类加载器，递归向上，往上层的类加载器寻找
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
			//如果上层的都找不到相应的class
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                //自己去加载
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

其实整个逻辑已经很清晰了，为了更好理解，我这里画张图给给大家，更好理解一点：

![](https://static.lovebilibili.com/jvm_09.png)

看到这里，应该都清楚了双亲委派机制的流程了。**重点来了，为什么要使用双亲委派机制呢？**

如果面试官问这个问题，一定要答出关键字：**安全性**。

反证法来辩证。假设不采用双亲委派机制，那我可以自定义一个类加载器，然后我写一个java.lang.String类用自定义的类加载器加载进去，原来java本身又有一个java.lang.String类，那么类的唯一性就没法保证，就不就给虚拟机的安全带来的隐患了吗。**所以要保证一个类只能由同一个类加载器加载，才能保证系统类的的安全**。

# 六、自定义类加载器

自定义类加载器，上面讲过可以有样学样，自定义一个类继承ClassLoader抽象类。重写哪个方法呢？loadClass()方法是加载类的方法，重写这个不就行了？

如果重写loadClass()那证明有思考过，但是不太对，因为重写loadClass()会破坏了双亲委派机制的逻辑。应该重写loadClass()方法里的findClass()方法。

findClass()方法才是自定义类加载器加载类的方法。

![](https://static.lovebilibili.com/jvm_10.png)

那findClass()方法源码是怎么样的呢？

![](https://static.lovebilibili.com/jvm_11.png)

明显这个方法是给子类重写用的，权限修饰符也是protected，如果不重写，那就会抛出找不到类的异常。如果学过设计模式的同学，应该看得出来这里用了**模板模式**的设计模式。所以我们自定义类加载器重写此方法即可。开始动手！

创建CustomerClassLoader类，继承ClassLoader抽象类的findClass()方法。

```java
public class CustomerClassLoader extends ClassLoader {
	//class文件在磁盘中的路径
    private String path;
	//通过构造器初始化class文件的路径
    public CustomerClassLoader(String path) {
        this.path = path;
    }

    /**
     * 加载类
     *
     * @param name 类的全路径
     * @return Class<?>
     * @author Ye hongzhi
     */
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class clazz = null;
        //获取class文件，转成字节码数组
        byte[] data = getData();
        if (data != null) {
            //将class的字节码数组转换成Class类的实例
            clazz = defineClass(name, data, 0, data.length);
        }
        //返回Class对象
        return clazz;
    }

    private byte[] getData() {
        File file = new File(path);
        if (file.exists()) {
            try (FileInputStream in = new FileInputStream(file);
                 ByteArrayOutputStream out = new ByteArrayOutputStream();) {
                byte[] buffer = new byte[1024];
                int size;
                while ((size = in.read(buffer)) != -1) {
                    out.write(buffer, 0, size);
                }
                return out.toByteArray();
            } catch (IOException e) {
                e.printStackTrace();
                return null;
            }
        } else {
            return null;
        }
    }
}
```

这样就完成了，接下来测试一下，定义一个Hello类。

```java
public class Hello {
    public void say() {
        System.out.println("hello.......java");
    }
}
```

使用javac命令编译成class文件，如下图：

![](https://static.lovebilibili.com/jvm_12.png)

最后写个main方法运行测试一把：

```java
public class Main {
    public static void main(String[] args) throws Exception {
        String path = "D:\\mall\\core\\src\\main\\java\\io\\github\\yehongzhi\\classloader\\Hello.class";
        CustomerClassLoader classLoader = new CustomerClassLoader(path);
        Class<?> clazz = classLoader.findClass("io.github.yehongzhi.classloader.Hello");
        System.out.println("使用类加载器：" + clazz.getClassLoader());
        Method method = clazz.getDeclaredMethod("say");
        Object obj = clazz.newInstance();
        method.invoke(obj);
    }
}
```

运行结果：

![](https://static.lovebilibili.com/jvm_13.png)

# 七、破坏双亲委派机制

看到这里，你肯定会很疑惑。上面不是才讲过双亲委派机制为了保证系统的安全性吗，为什么又要破坏双亲委派机制呢？

重温一下双亲委派机制，应该还记得，就是底层的类加载器一直委托上层的类加载器，如果上层的已经加载了，就无需加载，上层的类加载器没有加载则自己加载。这就**突出了双亲委派机制的一个缺陷，就是只能子的类加载器委托父的类加载器，不能反过来用父的类加载器委托子的类加载器**。

那你会问，什么情况会出现父的类加载器委托子的类加载器呢？

还真有这个场景，就是加载JDBC的数据库驱动。在JDK中有一个所有 JDBC 驱动程序需要实现的接口Java.sql.Driver。而Driver接口的实现类则是由各大数据库厂商提供。那问题就出现了，**DriverManager(JDK的rt.jar包中)要加载各个实现了Driver接口的实现类，然后进行统一管理，但是DriverManager是由Bootstrap类加载器加载的，只能加载JAVA_HOME下lib目录下的文件(可以看回上面双亲委派机制的第一张图)，但是实现类是服务商提供的，由AppClassLoader加载，这就需要Bootstrap(上层类加载器)委托AppClassLoader(下层类加载器)，也就破坏了双亲委派机制**。这只是其中一种场景，破坏双亲委派机制的例子还有很多。

那么怎么实现破坏双亲委派机制呢？

- 最简单就是自定义类加载器，前面讲过为了不破坏双亲委派机制重写findClass()方法，所以如果我要破坏双亲委派机制，那就重写loadClass()方法，直接把双亲委派机制的逻辑给改了。在JDK1.2后不提倡重写此方法。所以提供下面这种方式。
- 使用线程上下文件类加载器(Thread Context ClassLoader)。**这个类加载器可以通过java.lang.Thread类的setContextClassLoader()方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个；如果在应用程序的全局范围内都没有设置过，那么这个类加载器默认就是AppClassLoader类加载器**。

![](https://static.lovebilibili.com/jvm_14.png)

那么刚刚说的JDBC又是采用什么方式破坏双亲委派机制的呢？

当然是采用上下文文件类加载器，还有使用了SPI机制，下面一步一步分解。

第一步，Bootstrap加载DriverManager类，在DriverManager类的静态代码块调用初始化方法。

```java
public class DriverManager {
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
}
```

第二步，加载Driver接口的所有实现类，得到Driver实现类的集合，获取一个迭代器。

![](https://static.lovebilibili.com/jvm_15.png)

第三步，看ServiceLoader.load()方法。

![](https://static.lovebilibili.com/jvm_16.png)

![](https://static.lovebilibili.com/jvm_17.png)

![](https://static.lovebilibili.com/jvm_18.png)

第四步，看迭代器driversIterator。

![](https://static.lovebilibili.com/jvm_19.png)

接着一直找下去，就会看到一个很神奇的地方。

![](https://static.lovebilibili.com/jvm_21.png)

而这个常量值PREFIX则是：

```java
private static final String PREFIX = "META-INF/services/";
```

所以我们可以在mysql驱动包中找到这个文件：

![](https://static.lovebilibili.com/jvm_22.png)

通过文件名找接口的实现类，这是java的SPI机制。到此为止，破案了大人！

作为暖男的我，就画张图，总结一下整个过程吧：

![](https://static.lovebilibili.com/jvm_23.png)

# 总结

这篇文章主要介绍了JVM，然后讲到JVM的类加载机制的三大步骤，接着讲自定义类加载器以及双亲委派机制。最后再深入探讨了为什么要使用双亲委派机制，又为什么要破坏双亲委派机制的问题。可能讲得有点长，不过我相信应该都看懂了，因为我讲得比较通俗，而且图文并茂。

上面所有例子的代码都上传Github了：

> https://github.com/yehongzhi/mall

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！