# 前言

其实在开发中经常会看到泛型的使用，但是很多人对其也是一知半解，大概知道这是一个类似标签的东西。比如最常见的给集合定义泛型。

```java
List<String> list = new ArrayList<>();
Map<String,Object> map = new HashMap<>();
```

那么什么是泛型，为什么使用泛型，怎么使用泛型，接着往下看。

# 什么是泛型

> Java泛型是J2SE1.5中引入的一个新特性，其本质是参数化类型，也就是说所操作的数据类型被指定为一个参数（type parameter），这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口、泛型方法。-- 百度百科

这句话读起来有点拗口，但是我们要抓住他说的关键，`参数化类型`和`可以用在类、接口和方法的创建中`，我们知道泛型是在什么地方使用。

# 为什么使用泛型

一般我在思考这种问题时，会反过来思考，假如没有泛型会怎么样？

我们以最简单的`List`集合为例子，假如没有泛型：

```java
public static void main(String[] args) {
    List list = new ArrayList();
    list.add("good");
    list.add(100);
    list.add('a');
    for(int i = 0; i < list.size(); i++){
        String val = (String) list.get(i);
        System.out.println("val:" + val);
    }
}
```

很显然在没有泛型的时候，List默认是Object类型，所以List里的元素可以是任意的，看起来集合里装着任意类型的参数是“挺不错”，但是任意的类型的缺点也是很明显的，就是要开发者对集合中的元素类型在预知的情况下进行操作，否则编译时不会提示错误，但是运行时很容易出现类型转换异常(ClassCastException)。

如果没有泛型，第二个小问题是，我们把一个对象放进了集合中，但是集合并不会记住这个对象的类型，再次取出时统统都会变成Object类，但是在运行时仍然为其本身的类型。

所以引入泛型就可以解决以上两个问题：

- 类型安全问题。使用泛型，则会在编译期就能发现类型转换异常的错误。
- 消除类型强转。泛型可以消除源代码中的许多强转类型的操作，这样可以使代码更加可读，并减少出错的机会。

# 泛型的特性

泛型只有在编译阶段有效，在运行阶段会被擦除。

下面做个试验，请看代码：

```java
List<String> stringArrayList = new ArrayList<String>();
List<Integer> integerArrayList = new ArrayList<Integer>();

Class classStringArrayList = stringArrayList.getClass();
Class classIntegerArrayList = integerArrayList.getClass();
System.out.println(classStringArrayList == classIntegerArrayList);
```

结果是true，由此可看出，在运行时`ArrayList<String>`和`ArrayList<Integer>`都会被擦除成`ArrayList`。

Java 泛型擦除是 Java 泛型中的一个重要特性，其目的是避免过多的创建类而造成的运行时的过度消耗。

# 泛型的使用方式

在上文也提到泛型有三种使用方式：泛型类、泛型接口、泛型方法。

## 泛型类

基本语法：

```java
public class 类名<泛型标识, 泛型标识, ...> {
    private 泛型标识 变量名;
}
```

示例代码：

```java
public class GenericClass<T> {
    private T t;
}
```

在泛型类里面，泛型形参T可用在返回值和方法参数上，例如：

```java
public class GenericClass<T> {

    private T t;

    public GenericClass() {
    }

    public void setValue(T t) {//作为参数
        this.t = t;
    }

    public T getValue() {//作为返回值
        return t;
    }
}
```

当我们创建类实例时，就可以传入类型实参：

```java
public static void main(String[] args) throws Exception {
    //泛型传入了String类型
    GenericClass<String> generic = new GenericClass<String>();
    //这里就限制了setValue()方法只能传入String类型
    generic.setValue("abc");
    //限制了getValue()方法得到的也是String类型
    String value = generic.getValue();
    System.out.println(value);
}
```

这里与普通类创建实例不同的地方在于，泛型类的构造需要在类名后面添加上<泛型>，这个尖括号传的是什么类型，T就代表什么类型。泛型类的作用就体现在这里，他限制了这个类的使用了泛型标识的方法的返回值和参数。

## 泛型方法

基本语法：

```java
修饰符 <T, E, ...> 返回值类型 方法名(形参列表){
}
```

示例：

```java
//静态方法
public static <E> E getObject1(Class<E> clazz) throws Exception {
    return clazz.newInstance();
}

//实例方法
public <E> E getObject2(Class<E> clazz) throws Exception {
    return clazz.newInstance();
}
```

使用示例：

```java
public static void main(String[] args) throws Exception {
    GenericClass<String> generic = new GenericClass<>();
    Object object1 = GenericClass.getObject1(Object.class);
    System.out.println(object1);
    Object object2 = generic.getObject2(Object.class);
    System.out.println(object2);
}
```

其实泛型方法比较简单，就是在返回值前加上尖括号<泛型标识>来表示泛型变量。不过对于初学者来说，很容易会跟泛型类的泛型方法混淆，特别是泛型类里定义了泛型方法的情况。

```java
public class GenericClass<T> {
    private T t;

    public GenericClass() {
    }
	//泛型类的方法
    public void setValue(T t) {
        this.t = t;
    }
	//泛型类的方法
    public T getValue() {
        return t;
    }
	//静态泛型方法，区别在于在返回值前需要加上尖括号<泛型标识>
    public static <E> E getObject1(Class<E> clazz) throws Exception {
        return clazz.newInstance();
    }
	//实例泛型方法，区别在于在返回值前需要加上尖括号<泛型标识>
    public <E> E getObject2(Class<E> clazz) throws Exception {
        return clazz.newInstance();
    }
}
```

## 泛型接口

基本语法：

```java
public 接口名称 <泛型标识, 泛型标识, ...>{
    泛型标识 方法名();
}
```

示例：

```java
public interface Generator<T> {
    T next();
}
```

当实现泛型接口不传入实参时，与泛型类定义相同，需要将泛型的声明加在类名后面：

```java
public class ObjectGenerator<T> implements Generator<T> {
    @Override
    public T next() {
        return null;
    }
}
```

当实现泛型接口传入实参时，当泛型参数传入了具体的实参后，则所有使用到泛型的地方都会被替换成传入的参数类型。比如下面的例子中，next()方法的返回值就变成了Integer类型：

```java
public class IntegerGenerator implements Generator<Integer> {
    @Override
    public Integer next() {
        Random random = new Random();
        return random.nextInt(10);
    }
}
```

## 泛型通配符

类型通配符一般是使用 **?** 代替具体的类型实参，这里的 **?** 是具体的类型实参，而不是类型形参。它跟Integer，Double，String这些类型一样都是一种实际的类型，我们可以把 **?** 看作是所有类型的父类。

类型通配符又可以分为类型通配符上限和类型通配符下限。

### 类型通配符上限

基本语法：

```java
类/接口<? extends 实参类型>
```

要求该泛型的类型，只能是实参类型，或实参类型的 **子类** 类型。

比如：

```java
public class ListUtils<T extends List> {
    public void doing(T t) {
        System.out.println("size: " + t.size());
        for (Object o : t) {
            System.out.println(o);
        }
    }
}
```

这样就限制了传入的泛型只能是List的子类，如下所示：

```java
public static void main(String[] args) throws Exception {
    ListUtils<List> utils = new ListUtils<>();
    List<String> list = Arrays.asList("1", "2", "3");
    utils.doing(list);
}
```

### 类型通配符下限

基本语法：

```java
类/接口<? super 实参类型>
```

要求该泛型的类型，只能是实参类型，或实参类型的 **父类** 类型

比如：

```java
public class ListUtils {
    //静态泛型方法，使用super关键字定义通配符下限
    public static <E> void copy(Collection<? super E> parentList, Collection<E> childList) {
        for (E e : childList) {
            parentList.add(e);
        }
    }
}
```

限制了传入的List类型：

```java
public static void main(String[] args) throws Exception {
    Integer i1 = new Integer(10);
    Integer i2 = new Integer(20);
    List<Integer> integers = new ArrayList<>();
    integers.add(i1);
    integers.add(i2);
    List<Number> numbers = new ArrayList<>();
    //childList传入的是List<Integer>，所以parentList传入的只能是Integer本身或者其父类的List
    ListUtils.copy(numbers, integers);
    System.out.println(numbers);//[10, 20]
}
```

## 泛型使用注意点

不能在静态字段使用泛型。

![](https://static.lovebilibili.com/fanxing_01.png)

不能和基本类型一起使用。

![](https://static.lovebilibili.com/fanxing_02.png)

异常类不能使用泛型。

![](https://static.lovebilibili.com/fanxing_03.png)

# 总结

最后总结一下泛型的作用：

- 提高了代码的可读性，这点毋庸置疑。
- 解决类型安全的问题，在编译期就能发现类型转换异常的错误。
- 可拓展性强，可以使用泛型通配符定义，不需要定义实际的数据类型。
- 提高了代码的重用性。可以重用数据处理算法，不需要为每一种类型都提供特定的代码。

非常感谢你的阅读，希望这篇文章能给到你帮助和启发。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！