---

title: 装饰者模式与IO流
date: 2020-05-04 22:10:01
index_img: https://static.lovebilibili.com/decorator_index.jpg
tags:
	- java
	- 设计模式
---

# 装饰者模式

## 定义

装饰者模式是一种**对象结构型**模式。**动态**地给一个对象添加一些**额外的**职责，就增加功能来说，装饰者模式比生成子类更为灵活。

<!-- more -->

## 通俗解释

上面的定义在网上是随处可见的描述，怎么解释呢。比如：我前几天和女朋友去买戒指，珠宝店的销售给我推荐了一种**自由搭配**的原创戒指。他跟我介绍戒指的元素需要选择材质(黄金，铂金，彩金)、表面工艺(拉丝，磨砂，光滑，铸造)、镶钻(内嵌，外嵌)、指环大小等等，然后组成一个戒指。这种就是装饰者模式的应用，原型是一个戒指，不断地给对象添加额外的职责，然后得到最终想要的产品。这样就可以通过不同的搭配产生很多不同类型的戒指。

后面那句**装饰者模式比生成子类更为灵活**怎么理解。如果用子类去描述的话，要把每一种搭配的结果都变成一个子类，也就是要穷举，就会产生很多子类，也就是造成**“类爆炸”**。所以就会说装饰者模式更加灵活。

## 来个例子

现在有一个需求，要求做一个加密的工具类，对传入的字符串加密。加密的算法有很多，有**MD5、AES、DES等等**，一般加密都不是单独使用一种加密算法，而是多种混合一起使用，这样可以提高安全性。

现在有三种算法：`MD5、AES、DES`。做一个工具类，给系统提供加密的服务，要求可以自由搭配使用。

## 使用继承的方式实现

我们就创建一个抽象类`EncryptionBase`，每一种组合方式就创建一个子类继承`EncryptionBase`，现在有三种加密方式，很容易我们可以穷举完，总共有6种组合。请看以下代码：

首先创建一个抽象类`EncryptionBase`：

```java
public abstract class EncryptionBase {
    public abstract String encrypt(String string,String password);
}
```

接着创建子类继承抽象类，并且实现其方法。以其中一个为例，其他实现类都类似：

```java
public class AESandDESandMD5Encryption extends EncryptionBase {
    @Override
    public String encrypt(String string, String password) {
        //网上可以找具体加密的代码，我这里篇幅受限就不展示了
        //AES加密
        byte[] encryptByAES = AESUtil.encrypt(string, password);
        //DES加密
        byte[] encryptByDES = DESUtil.encrypt(encryptByAES, password);
        //MD5加密
        return MD5Util.encryptByMD5(new String(encryptByDES) + password);
    }
}
```

我们就可以实现以下效果，有6个实现类分别实现了3种加密算法的不同顺序。

```java
public static void main(String[] args) {
        String string = "需要加密的字符串";
    	//秘钥
        String password = "12345678";

        //第一种加密顺序：AES->DES->MD5
        EncryptionBase AESandDESandMD5 = new AESandDESandMD5Encryption();
        //第二种加密顺序：AES->MD5->DES
        EncryptionBase AESandMD5andDES = new AESandMD5andDESEncryption();
        //第三种加密顺序：DES->AES->MD5
        EncryptionBase DESandAESandMD5 = new DESandAESandMD5Encryption();
        //第四种加密顺序：DES->MD5->AES
        EncryptionBase DESandMD5andAES = new DESandMD5andAESEncryption();
        //第五种加密顺序：MD5->DES->AES
        EncryptionBase MD5andDESandAES = new MD5andDESandAESEncryption();
        //第六种加密顺序：MD5->AES->DES
        EncryptionBase MD5andAESandDES = new MD5andAESandDESEncryption();
    }
```

以上就是使用继承的方式来完成这个需求。看起来没什么问题，但是仔细思考你会发现几个问题。

1. **会创建很多子类。**为什么3种算法是6个类呢？这是根据数学的排列组合`3*2*1=6`，假设再多两种算法呢？那就是`5*4*3*2*1=120`，那就是120个类了！这就是**“类爆炸”**。
2. **不符合开闭原则。**假设增加了新的算法，那就要修改原来的类，不利于代码的维护。
3. 假如其中一种加密算法要用两次，比如双重`MD5`加密，那也是很难扩展的。

如果你不会装饰者模式，那估计要加班加点去写代码，创建很多类。如果你会装饰者模式，那问题就很简单了，那怎么做呢？请继续看下去。

## 使用装饰者模式实现

首先创建三种算法的基础类，继承`EncryptionBase`，实现三种加密算法。

MD5加密

```java
public class MD5Encryption extends EncryptionBase {
    @Override
    public String encrypt(String string, String password) {
        System.out.println("使用MD5加密，得到基础密文");
        return MD5Util.encryptByMD5(string + password);
    }
}
```

AES加密

```java
public class AESEncryption extends EncryptionBase {
    @Override
    public String encrypt(String string, String password) {
        System.out.println("使用AES加密，得到基础密文");
        return new String(AESUtil.encrypt(string, password));
    }
}
```

DES加密

```java
public class DESEncryption extends EncryptionBase {
    @Override
    public String encrypt(String string, String password) {
        System.out.println("使用DES加密，得到基础密文");
        return new String(DESUtil.encrypt(string.getBytes(), password));
    }
}
```

接着创建一个装饰抽象类`EncryptionDecorator`，需要继承`EncryptionBase`

```java
public abstract class EncryptionDecorator extends EncryptionBase {
	
    //定义一个父类的成员变量，用来存储其他装饰类，或者基础加密类
    private EncryptionBase encryption;

    public EncryptionDecorator(EncryptionBase encryption) {
        this.encryption = encryption;
    }

    @Override
    public String encrypt(String string, String password) throws Exception{
        return encryption.encrypt(string, password);
    }
}
```

然后实现三种加密的装饰者实现类，需要继承抽象装饰者类`EncryptionDecorator`。

MD5加密装饰者实现类`MD5EncryptionDecorator`

```java
public class MD5EncryptionDecorator extends EncryptionDecorator {

    public MD5EncryptionDecorator(EncryptionBase encryption) {
        //有参构造器获取到参数，调用父类的有参构造器，
        //当下面encrypt()方法里调用父类的加密算法就会调用传入的算法实现类的加密算法
        super(encryption);
    }

    @Override
    public String encrypt(String string, String password) throws Exception{
        //首先调用父类的加密方法，得到父类的算法加密后的结果
        String encrypt = super.encrypt(string, password);
        System.out.println("使用MD5加密");
        //得到的密文，再用MD5算法加密，返回
        return MD5Util.encryptByMD5(encrypt + password);
    }
}
```

AES加密装饰者实现类`AESEncryptionDecorator`

```java
public class AESEncryptionDecorator extends EncryptionDecorator {

    public AESEncryptionDecorator(EncryptionBase encryption) {
        super(encryption);
    }

    @Override
    public String encrypt(String string, String password) throws Exception{
        //首先调用父类的加密方法，得到父类的算法加密后的结果
        String encrypt = super.encrypt(string, password);
        System.out.println("使用AES加密");
        //得到的密文，再用AES算法加密，返回
        return new String(AESUtil.encrypt(encrypt, password),"UTF-8");
    }
}
```

DES加密装饰者实现类`DESEncryptionDecorator`

```java
public class DESEncryptionDecorator extends EncryptionDecorator {
    public DESEncryptionDecorator(EncryptionBase encryption) {
        super(encryption);
    }

    @Override
    public String encrypt(String string, String password) throws Exception{
        //首先调用父类的加密方法，得到父类的算法加密后的结果
        String encrypt = super.encrypt(string, password);
        System.out.println("使用DES加密");
        //得到的密文，再用DES算法加密，返回
        return new String(DESUtil.encrypt(encrypt.getBytes(), password),"UTF-8");
    }
}
```

大功告成！我们用`main()`方法测试一下：

```java
public class Main {
    public static void main(String[] args) throws Exception{
        String string = "需要加密的字符串";
        String password = "12345678";
        //第一种加密顺序：AES->DES->MD5
        EncryptionBase encryptionBase = new MD5EncryptionDecorator(new DESEncryptionDecorator(new AESEncryption()));
        encryptionBase.encrypt(string, password);
    }
}
```

控制台打印结果：

```java
/**
使用AES加密，得到基础密文
使用DES加密
使用MD5加密
*/
```

我们可以看到结果是很完美地实现了，你可以任意搭配加密算法，即使加多N种算法，我们也不会呈指数增加类的数量，只需要增加M*N个类即可，M是基础构件数量，N是具体装饰类数量。

原理是什么呢？我们不能说只学到形式，而不明白原理。接下来看类图。

在IDEA可以选中类名，然后右键，选中“Diagrams”，再选中“show Diagrams...”，就可以打开类图。

<img src="https://static.lovebilibili.com/MD5EncryptionDecorator.png"/>

```java
//MD5(DES(AES))，最顶层的父类是AES，所以先执行，第二层是DES，第二执行，最外层是MD5第三执行
EncryptionBase encryptionBase = new MD5EncryptionDecorator(new DESEncryptionDecorator(new AESEncryption()));
encryptionBase.encrypt(string, password);
```

以上面这句代码为例，那么调用顺序就是：AES->DES->MD5

<img src="https://static.lovebilibili.com/decorator1.png" style="width:100%;"/>

这就是装饰者模式的原理，其实很简单的，很容易就可以看清楚。

## 装饰者模式与I/O流

看了上面的代码，很容易我们能联想到IO流也有类似的创建方式，比如我们要用文件缓冲输入流，那就要这样创建：

```java
InputStream inputStream 
    = new BufferedInputStream(new FileInputStream(new File("/D:abc.text")));
```

可以看出IO流使用了装饰者模式。

如果我们打开源码，查看`BufferedInputStream`，我们可以看到：

```java
public class BufferedInputStream extends FilterInputStream {
    //有参构造器
    public BufferedInputStream(InputStream in, int size) {
        //调用父类构造器，这是关键
        //通过上面我们学过的例子，可以知道BufferedInputStream是装饰实现类
        super(in);
        if (size <= 0) {
            throw new IllegalArgumentException("Buffer size <= 0");
        }
        buf = new byte[size];
    }
}
```

关键在`FilterInputStream`这个类，这是装饰者模式的基类。查看源码：

```java
public class FilterInputStream extends InputStream {
    /**
     * The input stream to be filtered.
     */
    protected volatile InputStream in;
    
    protected FilterInputStream(InputStream in) {
        this.in = in;
    }
    
    public int read() throws IOException {
        return in.read();
    }
}
```

`FilterInputStream`类似于加密算法例子的`EncryptionDecorator`类。我们可以通过加密算法的例子和这个作对比，就可以很容易地看出他们的关系。类图如下：

<img src="https://static.lovebilibili.com/FilterInputStream.png" style="width:100%;"/>

`FileInputStream`就是基础构件类，可以通过`FilterInputStream`的子类去做扩展，增加额外的功能，比如可以使用`BufferedInputStream`增加缓冲的作用。

接着我们真正理解了IO流的装饰者模式的应用后，我们可以写一个扩展类，实现一个功能：读取磁盘的文件，把所有字母变成大写的字母。代码如下：

```java
public class CapitalizaInputStream extends FilterInputStream {

    public CapitalizaInputStream(InputStream in) {
        super(in);
    }

    @Override
    public int read(byte[] b, int off, int len) throws IOException {
        int result = super.read(b, off, len);
        for (int i = off; i < off + result; i++){
            //如果是小写字母，转成大写，其他不是小写字母的不变
            if(Character.isLetter((char)b[i])){
                b[i] = (byte) Character.toUpperCase((char) b[i]);
            }
        }
        return result;
    }
}
```

abc.txt文件内容：

```yaml
abcdefghijklmnopqrstuvwxyz
```

Main方法测试代码：

```java
public static void main(String[] args) throws Exception {
        InputStream inputStream 
            = new CapitalizaInputStream(new FileInputStream(new File("D://abc.txt")));
        byte[] bytes = new byte[1024 * 2];
        int c;
        while ((c = inputStream.read(bytes, 0, bytes.length)) != -1) {
            System.out.println(new String(bytes, 0, c));
        }
        inputStream.close();
    }
```

控制台打印结果：

```yaml
ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

以上就是IO流关于装饰者模式的扩展，能够加深我们对装饰者模式的理解。很多博客写不清楚，讲得很复杂，或者讲得很简单，很大原因是我们只看，而没有动手去做，动手去自己写，自己琢磨，就很容易能理解。这是学习方法，不是关注了公众号，看几篇文章就能轻松学会的，学习总是要自己动手才会理解深刻。看我的文章可以提供一些思路，更容易去上手。

## 总结

装饰者模式的优点：

1. 可以动态地扩展类的功能，不会相互耦合。
2. 符合开闭原则，利于代码维护。
3. 比继承扩展的方式要更加灵活。

缺点：多层装饰，代码结构变得复杂。

更多的java技术分享，就关注java技术爱好者吧！

<img src="https://me.lovebilibili.com/img/wechat.jpg-slim" alt="100" style="zoom:50%;" />

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！