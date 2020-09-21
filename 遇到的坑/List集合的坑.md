---
title: List集合的坑
date: 2020-06-21 17:24:09
index_img: https://static.lovebilibili.com/List集合的坑.jpg
tags:
	- java
	- 集合
	- 经验总结
---

> 学如逆水行舟，不进则退

经过几年的工作经验，我发现`List`有很多坑，之前公司有些实习生一不小心就踩到了，所以我打算写一篇文章总结一下，希望看到这篇文章的人能不再踩到坑，代码没bug。做个快乐的程序员。

<!-- more -->

### 迭代时删除元素
使用`for-each`迭代遍历时，删除集合中的元素，会报错。
```java
	private static List<String> list = new ArrayList<>();

    static {
        //初始化集合
        for (int i = 1; i <= 10; i++) {
            list.add(String.valueOf(i));
        }
    }

    public static void main(String[] args) {
        //使用for-each迭代时删除元素
        for (String str : list) {
            if ("1".equals(str)) {
                list.remove(str);
            }
        }
    }
```
或者你使用迭代器`Iterator`遍历时，删除元素。
```java
	public static void main(String[] args) {
        //使用Iterator迭代器遍历时，删除元素
        Iterator<String> it = list.iterator();
        while (it.hasNext()) {
            String str = it.next();
            if ("1".equals(str)) {
                list.remove(str);
            }
        }
    }
```
以上两种情况都会报这个错：
```java
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:901)
	at java.util.ArrayList$Itr.next(ArrayList.java:851)
```
这就是不正确的删除姿势，那怎么删呢？

使用`for-i`循环遍历删除(亲测有效)：
```java
	public static void main(String[] args) {
        //使用Iterator迭代器遍历时，删除元素
        for (int i = 0; i < list.size(); i++) {
            String s = list.get(i);
            if ("1".equals(s)) {
                list.remove(s);
            }
        }
        list.forEach(System.out::println);//2 3 4 5 6 7 8 9 10
    }
```
使用`for-i`循环倒序遍历，删除元素。
```java
	public static void main(String[] args) {
        //使用for-i倒序遍历，删除元素
        for (int i = list.size() - 1; i >= 0; i--) {
            String str = list.get(i);
            if ("1".equals(str)) {
                list.remove(str);
            }
        }
        list.forEach(System.out::println);//2 3 4 5 6 7 8 9 10
    }
```
使用`Iterator`的`remove()`方法删除。
```java
	public static void main(String[] args) {
        //使用Iterator迭代器遍历时，删除元素
        Iterator<String> it = list.iterator();
        while (it.hasNext()) {
            String str = it.next();
            if ("1".equals(str)) {
                it.remove();
            }
        }
        list.forEach(System.out::println);//2 3 4 5 6 7 8 9 10
    }
```
要么潇洒一点，用`Lambda`表达式。在java8中，`List`增加了一个`removeIf()`方法用于删除。
```java
	public static void main(String[] args) {
        //使用removeIf()遍历时，删除元素。删除集合中为1的元素
        list.removeIf(str -> "1".equals(str));
        list.forEach(System.out::println);//2 3 4 5 6 7 8 9 10
    }
```
### 使用asList()获得集合删除/增加
看代码演示：
```java
	public static void main(String[] args) {
        List<Integer> nums = Arrays.asList(1, 2, 3, 4, 5, 6);
        nums.add(7);
    }
```
```java
	public static void main(String[] args) {
        List<Integer> nums = Arrays.asList(1, 2, 3, 4, 5, 6);
        nums.remove(1);
    }
```
如果你进行以上操作，就会看到报错：
```java
Exception in thread "main" java.lang.UnsupportedOperationException
	at java.util.AbstractList.remove(AbstractList.java:161)
```
为什么会报这个错，看一下源代码就知道了！
```java
private static class ArrayList<E> extends AbstractList<E> implements RandomAccess, java.io.Serializable {
    
}
```
`ArrayList`不是`util`包的`ArrayList`，而是`Arrays`的一个内部类。因为继承了`AbstractList`抽象类，但是又没有实现`add()`、`remove()`方法。所以会调用抽象类的`add()`和`remove()`。
你猜猜抽象类的`add()`怎么着？
```java
	public void add(int index, E element) {
        throw new UnsupportedOperationException();
    }

	public E remove(int index) {
        throw new UnsupportedOperationException();
    }
```
所以不能用`asList()`得到的集合去增删了！
### 通过subList()方法获得集合后增删
当使用`subList()`方法获得集合后删除，原(父)集合也会被删除。
```java
	public static void main(String[] args) {
        List<String> subList = list.subList(0, 5);
        System.out.println(list);//[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        System.out.println(subList);//[1, 2, 3, 4, 5]
        subList.remove("1");
        System.out.println(list);//[2, 3, 4, 5, 6, 7, 8, 9, 10]
        System.out.println(subList);//[2, 3, 4, 5]
    }
```
当使用`subList()`方法获得集合后增加元素，原(父)集合也会增加。
```java
	public static void main(String[] args) {
        List<String> subList = list.subList(0, 5);
        System.out.println(list);//[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        System.out.println(subList);//[1, 2, 3, 4, 5]
        subList.add("11");
        System.out.println(list);//[1, 2, 3, 4, 5, 11, 6, 7, 8, 9, 10]
        System.out.println(subList);//[1, 2, 3, 4, 5, 11]
    }
```
大家看一下源码就知道什么原因了。
```java
private class SubList extends AbstractList<E> implements RandomAccess {
		public void add(int index, E e) {
            rangeCheckForAdd(index);
            checkForComodification();
            //父集合添加元素
            parent.add(parentOffset + index, e);
            this.modCount = parent.modCount;
            this.size++;
        }

        public E remove(int index) {
            rangeCheck(index);
            checkForComodification();
            //父集合删除元素
            E result = parent.remove(parentOffset + index);
            this.modCount = parent.modCount;
            this.size--;
            return result;
        }
}
```
如果希望截取的集合是和原集合互不干扰的话，可以这样：
```java
List<String> subList = new ArrayList<>(list.subList(0, 5));
```
### 使用Collections.unmodifiableList()创建不可变集合也是可变的。
当不可变集合的原集合改变时，不可变集合也跟着改变。演示代码：
```java
	public static void main(String[] args) {
        List<String> unmodifiableList = Collections.unmodifiableList(list);
        System.out.println(unmodifiableList);//[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        //删除原集合元素
        list.remove("1");
        System.out.println(unmodifiableList);//[2, 3, 4, 5, 6, 7, 8, 9, 10]
    }
```
看源码就知道原因了：
```java
	UnmodifiableList(List<? extends E> list) {
    	super(list);
    	this.list = list;
    }
```
**因为不可变集合的成员变量的引用是指向原集合的，所以当原集合改变时，不可变集合也会随之改变**。

解决方式：使用`Guava`工具包的`ImmutableList.copyOf()`方法创建。
```java
	public static void main(String[] args) throws Exception {
        List<String> unmodifiableList = ImmutableList.copyOf(list);
        list.remove("1");
        System.out.println(unmodifiableList);//[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    }
```

创作不易，觉得有用就**点个赞**吧。

想第一时间看到我更新的文章，可以微信搜索公众号「`java技术爱好者`」，**拒绝做一条咸鱼，我是一个在互联网荒野求生的程序员。我们下期再见！！！**
![在这里插入图片描述](https://static.lovebilibili.com/erweimaguanzhu.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！