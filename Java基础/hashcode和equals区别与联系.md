> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 写在前面

其实很早我就注意到阿里巴巴Java开发规范有一句话：`只要重写 equals，就必须重写 hashCode`。

![](https://static.lovebilibili.com/hashcode_equals_1.png)

我想很多人都会问为什么，所谓`知其然知其所以然`，对待知识不单止知道结论还得知道原因。

# hashCode方法

hashCode()方法的作用是获取哈希码，返回的是一个int整数

![](https://static.lovebilibili.com/hashcode_equals_2.png)

学过数据结构的都知道，哈希码的作用是确定对象在哈希表的索引下标。比如HashSet和HashMap就是使用了hashCode方法确定索引下标。如果两个对象返回的hashCode相同，就被称为“哈希冲突”。

# equals方法

equals()方法的作用很简单，就是判断两个对象是否相等，equals()方法是定义在Object类中，而所有的类的父类都是Object，所以如果不重写equals方法则会调用Object类的equals方法。

![](https://static.lovebilibili.com/hashcode_equals_3.png)

Object类的equals方法是用“==”号进行比较，在很多时候，因为==号比较的是两个对象的内存地址而不是实际的值，所以不是很符合业务要求。所以很多时候我们需要重写equals方法，去比较对象中每一个成员变量的值是否相等。

# 问题来了

>  重写equals()方法就可以比较两个对象是否相等，为什么还要重写hashcode()方法呢？

因为HashSet、HashMap底层在添加元素时，会先判断对象的hashCode是否相等，如果hashCode相等才会用equals()方法比较是否相等。换句话说，HashSet和HashMap在判断两个元素是否相等时，**会先判断hashCode，如果两个对象的hashCode不同则必定不相等**。

![](https://static.lovebilibili.com/hashcode_equals_8.png)

下面我们做一个试验，有一个User类，只重写equals()方法，然后放到Set集合中去重。

```java
public class User {

    private String id;

    private String name;

    private Integer age;
    
    public User(String id, String name, Integer age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(id, user.id) &&
            Objects.equals(name, user.name) &&
            Objects.equals(age, user.age);
    }
    
    //getter、setter、toString方法
}
```

然后我们循环创建10个成员变量的值都是一样的User对象，最后放到Set集合中去重。

```java
public static void main(String[] args) {
    List<User> list = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        User user = new User("1", "张三", 18);
        list.add(user);
    }
    Set<User> set = new HashSet<>(list);
    for (User user : set) {
        System.out.println(user);
    }
    List<User> users = list.stream().distinct().collect(Collectors.toList());
    System.out.println(users);
}
```

按道理我们预期会去重，只剩下一个“张三”的user，但实际上因为没有重写hashCode方法，所以没有去重。

![](https://static.lovebilibili.com/hashcode_equals_4.png)

接着我们在User类里面重写一些hashCode方法再试试，其他不变。

```java
public class User {
    //其他不变
    
    //重写hashCode方法
    @Override
    public int hashCode() {
        return Objects.hash(id, name, age);
    }
}
```

再运行，结果正确。

![](https://static.lovebilibili.com/hashcode_equals_5.png)

究其原因在于HashSet会先判断hashCode是否相等，如果hashCode不相等就直接认为两个对象不相等，不会再用equals()比较了。我们不妨看看重写hashCode方法和不重写hashCode方法的哈希码。

这是不重写hashCode方法的情况，每个user对象的哈希码都不一样，所以HashSet会认为都不相等。

![](https://static.lovebilibili.com/hashcode_equals_6.png)

这是重写hashCode方法的情况，因为是用对象所有的成员变量的值计算出的哈希码，所以只要两个对象的成员变量都是相等的，则生成的哈希码是相同的。

![](https://static.lovebilibili.com/hashcode_equals_7.png)

那么有些人看到这里，就会问，如果两个对象返回的哈希码都是一样的话，是不是就**一定相等**？

答案是不一定的，因为HashSet、HashMap判断哈希码相等后还会再用equals()方法判断。

总而言之：

- 哈希码不相等，则两个对象一定不相同。
- 哈希码相等，两个对象不一定相同。
- 两个对象相同，则哈希码和值都一定相等。

# 总结

所以回到开头讲的那句，`只要重写 equals，就必须重写 hashCode`，这是一个很重要的细节，如果不注意的话，很容易发生业务上的错误。

特别是有时候我们明明用了HashSet，distinct()去重，但是就是不生效，这时应该回头看看重写了equals()和hashCode()方法了吗？

那么这篇文章就写到这里了，感谢大家的阅读。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！