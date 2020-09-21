---
title: 记一次高级java面试
date: 2020-05-24 20:20:49
index_img: https://static.lovebilibili.com/interview_summary_1.jpg
tags:
	- java
	- 面试
---

# 记录一次高级JAVA开发面试题目
面试时间大概40多分钟，问了有十几个问题，回忆一下记录下来，总结经验，以供参考。

<!-- more -->

**1、 static关键字的作用，平时开发用在什么地方？**
答：主要有三种用法。
①修饰成员变量，用static修饰的成员变量就成为静态变量，静态变量只会存在一份，在类被加载时会初始化，且只会加载一次，通过类名访问。一般可以用static和final定义一些String类型，boolean类型，int类型的变量作为常量，可以减少资源的消耗。
②static修饰方法，该方法就被定义为静态方法，静态方法是不能被方法重写的，通过类名调用。一般用static定义一些工具类的方法。
③用static修饰代码块，该代码块就被定义为静态代码块，静态代码块在类初始化时被执行，且执行一次。一般用于初始化一些静态的成员变量的值。

**2、static修饰的成员变量和非static修饰的成员变量有什么区别？分别存在什么区域？**
答：静态成员变量在内存中只会存在一份，是通过类名访问，存在于静态区中。非静态成员变量是随着对象的创建而存在的，可以有多份，通过创建的对象访问，存在于堆内存中。

**3、说一下类初始化的顺序。**
答：静态成员变量、静态代码块、实例成员变量，实例代码块，构造器，实例方法。

**4、常用的集合类型有哪些？**
答：有Map、Set、List是比较常用的。

**5、List常用的实现类有哪些？ArrayList和LinkedList底层实现原理是什么？**
答：List常用的实现类有ArrayList和LinkedList。ArrayList底层原理是数组+动态扩容机制实现的，LinkedList底层原理是用Node结点形成的链表实现的。

**6、在开发中如何选择使用ArrayList和LinkedList？**
答：ArrayList是数组实现，所以通过下标访问效率最快，但是缺点是如果增删比较频繁的情况下，需要经常扩容，性能不是很好。LinkedList在增删的情况下，效率较高，但是访问集合中的元素时都需要从第一个元素开始遍历，效率较低。所以如果增删的情况较多的时候，可以使用LinkedList。查询较多时使用ArrayList。

 **7、List集合如果要排序有哪些实现方式？**
①使用List接口定义的sort()方法。
```java
list.sort(Comparator.comparingInt(User::getAge));
```
②使用Collections的sort()方法，排序的对象需要实现Comparable接口，重写compareTo()方法。
```java
//实现Comparable接口
public class User implements Comparable<User> {
	//重写compareTo方法
	@Override
	public int compareTo(User user) {
    		return Integer.compare(this.getAge(), user.getAge());
	}
}
```
使用Collections的sort()方法
```java
Collections.sort(list);
//如果不想实现Comparable接口，也可以使用这个方法
Collections.sort(list,Comparator.comparingInt(User::getAge));
```
③使用Stream流操作的sort()方法，传入一个Comparator接口。
```java
list.stream().sorted(Comparator.comparingInt(User::getAge)).collect(Collectors.toList());
```

 **8、ArrayList是线程安全的吗？有什么方式可以让ArrayList变成线程安全的？**
答：不是线程安全的。
使用Collections的synchronizedList()方法包装可获得线程安全的ArrayList。
```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
```

**9、你是怎么在项目中使用redis的？**
答：这其实是考了“redis常用的应用场景”这个问题。
①利用redis读写速度快的特点，可以做热点数据的储存，降低数据库查询的压力。
②利用redis键值设置有效期的特性，做一些限时的业务。比如手机验证码。
③利用setnx命令的特性，可以实现分布式锁。

**10、使用Redis实现分布式锁的原理是什么？**
答： 利用setnx命令的特性。使用setnx一个lockKey字符串作为键，当前的时间+上锁时间作为value。如果返回是0，表示已经被上锁了，需要等待锁持有者释放锁；如果返回1，则表示获得了锁。客户端释放锁的话执行del命令删除lockKey对应的键值。

**11、如果使用分布式锁加锁后，由于一些异常的原因没有执行解锁的操作，怎么办？**
答：一般解锁操作会放在finally代码块中执行。如果有极端情况下没有执行到解锁的操作，可以通过key对应的时间戳判断是否超时，然后使用GETSET命令去进行解锁，通过判断返回的时间戳是否是超时的key对应的时间戳，确认是否成功上锁。

**12、如果加分布式锁的时候，业务操作时间比较长，造成长时间的阻塞，有什么解决方案？**
答：可以在加锁时启动一个watch dog(看门狗)线程，每隔10秒检查一下，如果客户端还持有锁则加长lockKey的生存时间。或者可以考虑用zookeeper实现的分布式锁，因为zk实现原理是基于事件监听的方式来实现。

**13、MySQL性能优化的策略有哪些？**
①复杂的多表查询可以拆成多句简单查询。
②返回尽量少的列，按需返回，严禁使用select *。
③尽量使用索引列做查询条件和排序条件。
④使用复合索引要遵循最左匹配原则。

**14、MySQL索引创建的原则是什么？**
①对于查询频率高的字段，创建索引。
②对排序、分组、联合查询频率高的字段创建索引。
③如果多个列都需要设置索引，可以考虑创建复合索引。
④尽量选择数据量较少的列作为索引。
⑤一个表的索引数量不宜过多，会降低查询的效率。


**15、雪花算法是什么原理？**
答：使用一个 64 bit 的 long 型的数字作为全局唯一 id。是由时间戳、机房id、机器id、序号组成的。结合了UUID的全局唯一的特点，又具有自增有顺序的特点。

**16、为什么雪花算法生成的主键有字符串类型和long类型两种类型？**
答：因为后端返回给前端一个long类型时，会有可能产生丢失精度的问题，所以会有字符串的类型，弥补这个问题。

**17、谈一谈MySQL锁机制。**
主要有以下几种锁：
表锁。开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低。
行锁。开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高。
在MySQL中只有InnoDB存储引擎可以使用行锁。行锁又分为以下两种形式：
读锁(共享锁)：当读取一条数据时，会加上读锁，其他事务如果要读取是可以的，如果要修改则要等事务释放才可以。
写锁(排他锁)：这个比较简单，当有一个事务要修改数据时，就会给这些行加上写锁。在加锁期间，不允许其他事务加上任何的锁，只有当这个事务释放了，才可以加锁操作。

在这次面试中，其实也不是特别难，大部分都回答得不错，但是有两个问题不是很好。雪花算法为什么主键生成有两种类型这个问题没有答出来，还有分布式锁长时间阻塞的解决方案没有详细展开讲。

<img src="https://me.lovebilibili.com/img/wechat.jpg-slim" alt="100" style="zoom:50%;" />

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！

