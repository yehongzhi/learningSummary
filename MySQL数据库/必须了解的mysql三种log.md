> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 前言

大家有没有想过为什么MySQL数据库可以实现主从复制，实现持久化，实现回滚的呢？其实关键在于MySQL里的三种`log`，分别是：

- binlog
- redo log
- undo log

这三种log也是面试经常会问的问题，下面我们一起来探讨一下吧。

# 一、binlog

binlog应该是日常中听的最多的关于mysql中的log。

> 那么什么是binlog呢？

binlog是用于**记录数据库表结构和表数据变更的二进制日志**，比如insert、update、delete、create、truncate等等操作，不会记录select、show操作，因为没有对数据本身发生变更。

> binlog文件长什么样子呢？

使用`mysqlbinlog`命令可以查看。

![](https://static.lovebilibili.com/mysql_log_1.png)

会记录下每条变更的sql语句，还有执行开始时间，结束时间，事务id等等信息。

> 如何查看binlog是否打开，如果没打开怎么设置？

使用命令`show variables like '%log_bin%';`查看binlog是否打开。

![](https://static.lovebilibili.com/mysql_log_2.png)

如果像上图一样，没有开启binlog，那怎么开启呢？

找到`my.cnf`配置文件，增加下面配置(mysql版本5.7.31)：

```cnf
# 打开binlog
log-bin=mysql-bin
# 选择ROW(行)模式
binlog-format=ROW
```

修改后，重启mysql，配置生效。

执行`SHOW MASTER STATUS;`可以查看当前写入的binlog文件名。

![](https://static.lovebilibili.com/mysql_log_3.png)

> binlog用来干嘛的呢？

第一，用于主从复制。一般在公司中做一主二从的结构时，就需要master节点打开binlog日志，从机订阅binlog日志的信息，因为binlog日志记录了数据库数据的变更，所以当master发生数据变更时，从机也能随着master节点的数据变更而变更，做到主从复制的效果。

![](https://static.lovebilibili.com/mysql_log_5.jpg)

第二，用于数据恢复。因为binlog记录了数据库的变更，所以可以用于数据恢复。我们看到上面图中有个字段叫Position，这个参数是用于记录binlog日志的指针。当我们需要恢复数据时，只要指定**--start-position**和**--stop-position**，或者指定**--start-datetime**和**--stop-datetime**，那么就可以恢复指定区间的数据。

# 二、redo log

假设有一条update语句：

```sql
UPDATE `user` SET `name`='刘德华' WHERE `id`='1';
```

我们想象一下mysql修改数据的步骤，肯定是先把`id`='1'的数据查出来，然后修改名称为'刘德华'。再深层一点，mysql是使用页作为存储结构，所以MySQL会先把这条记录所在的页加载到内存中，然后对记录进行修改。但是我们都知道mysql支持持久化，最终数据都是存在于磁盘中。

假设需要修改的数据加载到内存中，并且修改成功了，但是还没来得及刷到磁盘中，这时数据库宕机了，那么这次修改成功后的数据就丢失了。

为了避免出现这种问题，MySQL引入了redo log。

![](https://static.lovebilibili.com/mysql_log_4.png)

如图所示，当执行数据变更操作时，首先把数据也加载到内存中，然后在内存中进行更新，更新完成后写入到redo log buffer中，然后由redo log buffer在写入到redo log file中。

redo log file记录着xxx页做了xxx修改，所以即使mysql发生宕机，也可以通过redo log进行数据恢复，也就是说在内存中更新成功后，即使没有刷新到磁盘中，但也不会因为宕机而导致数据丢失。

> redo log与事务机制是如何配合工作的？



![](https://static.lovebilibili.com/mysql_log_7.png)

如图所示：

第1-3步骤就是把数据变更，然后写入到内存中。

第4步记录到redo log中，然后把记录置为prepare(准备)状态。

第5，6步提交事务，提交事务之后，第7步把记录状态改成commit(提交)状态。

保证了事务与redo log的一致性。

> binlog和redo log都可以数据恢复，有什么区别？

redo log是恢复在内存更新后，还没来得及刷到磁盘的数据。

binlog是存储所有数据变更的情况，理论上只要记录在binlog上的数据，都可以恢复。

举个例子，**假如不小心整个数据库的数据被删除了，能使用redo log文件恢复数据吗**？

不可以使用redo log文件恢复，只能使用binlog文件恢复。因为redo log文件不会存储历史所有的数据的变更，当内存数据刷新到磁盘中，redo log的数据就失效了，也就是redo log文件内容是会被覆盖的。

> binlog又是在什么时候记录的呢？

答，在提交事务的时候。

![](https://static.lovebilibili.com/mysql_log_8.png)

# 三、undo log

undo log的作用主要**用于回滚**，mysql数据库的事务的原子性就是通过undo log实现的。我们都知道原子性是指对数据库的一系列操作，要么全部成功，要么全部失败。

undo log主要存储的是数据的逻辑变化日志，比如说我们要`insert`一条数据，那么undo log就会生成一条对应的delete日志。简单点说，undo log记录的是数据修改之前的数据，因为需要支持回滚。

那么当需要回滚时，只需要利用undo log的日志就可以恢复到修改前的数据。

undo log另一个作用是**实现多版本控制(MVCC)**，undo记录中包含了记录更改前的镜像，**如果更改数据的事务未提交**，对于隔离级别大于等于read commit的事务而言，**不应该返回更改后数据，而应该返回老版本的数据**。

# 总结

学完之后，我们知道这三种日志在mysql中都有着重要的作用，再回顾一下：

- binlog主要用于复制和数据恢复。
- redo log用于恢复在内存更新后，还没来得及刷到磁盘的数据。
- undo log用于实现回滚和多版本控制。

这篇文章就讲到这里了，感谢大家的阅读，希望看完大家能有所收获！

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！

