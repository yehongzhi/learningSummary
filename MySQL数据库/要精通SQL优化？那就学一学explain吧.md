

> **文章已收录Github精选，欢迎Star**：[https://github.com/yehongzhi](https://github.com/yehongzhi/learningSummary)

# 前言

在MySQL中，我们知道加索引能提高查询效率，这基本上算是常识了。但是有时候，我们加了索引还是觉得SQL查询效率低下，我想看看**有没有使用到索引，扫描了多少行，表的加载顺序**等等，怎么查看呢？其实MySQL自带的SQL分析神器**Explain执行计划**就能完成以上的事情！

# Explain有哪些信息

先确认一下试验的MySQL版本，这里使用的是`5.7.31`版本。

![](https://static.lovebilibili.com/mysql_explain_01.png)

只需要在SQL语句前加上explain关键字就可以查看执行计划，执行计划包括以下信息：id、select_type、table、partitions、type、possible_keys、key、key_len、ref、rows、filtered、Extra，总共12个字段信息。

![](https://static.lovebilibili.com/mysql_explain_02.png)

然后创建三个表：

```sql
CREATE TABLE `tb_student` (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `name` varchar(36) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `index_name` (`name`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb4 COMMENT='学生表';

CREATE TABLE `tb_class` (
  `id` INT(10) primary key not null auto_increment,
  `name` VARCHAR(36) NOT NULL,
	`stu_id` INT(10) NOT NULL,
	`tea_id` INT(10) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='班级表';

CREATE TABLE `tb_teacher` (
  `id` INT(10) primary key not null auto_increment,
  `name` VARCHAR(36) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='教师表';
```

# Explain执行计划详解

explain的使用很简单，只需要在SQL语句前加上关键字`explain`即可，关键是怎么看explain执行后返回的字段信息，这才是重点。

## 一、id

SELECT识别符。这是SELECT的查询序列号。**SQL执行的顺序的标识，SQL从大到小的执行**。id列有以下几个注意点：

- id相同时，执行顺序由上至下。
- id不同时，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行。

```sql
EXPLAIN SELECT * FROM `tb_student` WHERE id IN (SELECT stu_id FROM tb_class WHERE tea_id IN(SELECT id FROM tb_teacher WHERE `name` = '马老师'));
```

![](https://static.lovebilibili.com/mysql_explain_03.png)

根据原则，当id不同时，SQL从大到小执行，id相同则从上到下执行。

## 二、select_type

表示select查询的类型，用于区分各种复杂的查询，例如普通查询，联合查询，子查询等等。

### SIMPLE

表示最简单的查询操作，也就是查询SQL语句中没有子查询、union等操作。

### PRIMARY

当查询语句中包含复杂查询的子部分，表示复杂查询中最外层的 select。

### SUBQUERY

当 `select` 或 `where` 中包含有子查询，该子查询被标记为SUBQUERY。

### DERIVED

在SQL语句中包含在`from`子句中的子查询。

### UNION

表示在union中的第二个和随后的select语句。

### UNION RESULT

代表从`union`的临时表中读取数据。

```sql
EXPLAIN SELECT u.`name` FROM ((SELECT s.id,s.`name` FROM `tb_student` s) UNION (SELECT t.id,t.`name` FROM tb_teacher t)) AS u;
```

`<union2,3>`代表是id为2和3的select查询的结果进行union操作。

![](https://static.lovebilibili.com/mysql_explain_04.png)

### MATERIALIZED

`MATERIALIZED`表示物化子查询，子查询来自视图。

## 三、table

表示输出结果集的表的表名，并不一定是真实存在的表，也有可能是别名，临时表等等。

## 四、partitions

表示SQL语句查询时匹配到的分区信息，对于非分区表值为NULL，当查询的是分区表则会显示分区表命中的分区情况。

## 五、type

需要重点关注的一个字段信息，表示查询使用了哪种类型，在 `SQL`优化中是一个非常重要的指标，依次从优到差分别是：**system > const > eq_ref > ref > range > index > ALL**。

### system和const 

**单表中最多有一条匹配行，查询效率最高，所以这个匹配行的其他列的值可以被优化器在当前查询中当作常量来处理**。通常出现在根据主键或者唯一索引进行的查询，system是const的特例，表里只有一条元组匹配时（系统表）为system。

![](https://static.lovebilibili.com/mysql_explain_05.png)

![](https://static.lovebilibili.com/mysql_explain_06.png)

### eq_ref 

primary key 或 unique key 索引的所有部分被连接使用 ，最多只会返回一条符合条件的记录，所以这种类型常出现在多表的join查询。

![](https://static.lovebilibili.com/mysql_explain_07.png)

### ref

相比**eq_ref**，不使用唯一索引，而是使用普通索引或者唯一性索引的部分前缀，可能会找到多个符合条件的行。

![](https://static.lovebilibili.com/mysql_explain_08.png)

### range

使用索引选择行，仅检索给定范围内的行。一般来说是针对一个有索引的字段，给定范围检索数据，通常出现在where语句中使用 `bettween...and`、`<`、`>`、`<=`、`in` 等条件查询 。

![](https://static.lovebilibili.com/mysql_explain_09.png)

### index

扫描全表索引，通常比ALL要快一些。

![](https://static.lovebilibili.com/mysql_explain_10.png)

### ALL

**全表扫描，MySQL遍历全表来找到匹配行**，性能最差。

![](https://static.lovebilibili.com/mysql_explain_11.png)

## 六、possible_keys

表示在查询中可能使用到的索引来查找，别列出的索引并不一定是最终查询数据所用到的索引。

## 七、key

跟possible_keys有所区别，key表示查询中实际使用到的索引，若没有使用到索引则显示为NULL。

## 八、key_len

表示查询用到的索引key的长度(字节数)。如果单列索引，那么就会把整个索引长度计算进去，如果是联合索引，不是所有的列都用到，那么就只计算实际用到的列，因此可以**根据key_len来判断联合索引是否生效**。

## 九、ref

显示了哪些列或常量被用于查找索引列上的值。常见的值有：`const`，`func`，`null`，字段名。

## 十、rows

mysql估算要找到我们所需的记录，需要读取的行数。可以通过这个数据很直观的显示 `SQL` 性能的好坏，一般情况下 `rows` 值越小越好。

## 十一、filtered

指返回结果的行占需要读到的行(rows列的值)的百分比，一般来说越大越好。

## 十二、Extra

表示额外的信息。此字段能够给出让我们深入理解执行计划进一步的细节信息。

### Using index

说明在select查询中使用了覆盖索引。覆盖索引的好处是一条SQL通过索引就可以返回我们需要的数据。

![](https://static.lovebilibili.com/mysql_explain_12.png)

### Using where

查询时没使用到索引，然后通过where条件过滤获取到所需的数据。

![](https://static.lovebilibili.com/mysql_explain_13.png)

### Using temporary

表示在查询时，MySQL需要创建一个临时表来保存结果。临时表一般会比较影响性能，应该尽量避免。

![](https://static.lovebilibili.com/mysql_explain_14.png)

有时候使用DISTINCT去重时也会产生Using temporary。

![](https://static.lovebilibili.com/mysql_explain_15.png)

### **Using filesort** 

我们知道索引除了查询中能起作用外，排序也是能起到作用的，所以当SQL中包含 ORDER BY 操作，而且**无法利用索引完成排序操作**的时候，MySQL不得不选择相应的排序算法来实现，这时就会出现**Using filesort**，应该尽量避免使用**Using filesort**。

![](https://static.lovebilibili.com/mysql_explain_16.png)

# 总结

一般优化SQL语句第一步是要知道这条SQL语句有哪些需要优化的，explain执行计划就相当于一面镜子，能把详细的执行情况给开发者列出来。所以说善用explain执行计划，能解决80%的SQL优化问题。

explain的信息中，一般我们要关心的是type，看是什么级别，如果是在互联网公司一般需要在range以上的级别，接着关心的是Extra，有没有出现filesort或者using template，一旦出现就要想办法避免，接着再看key使用的是什么索引，还有看filtered筛选比是多少。

这篇文章就讲到这里了，希望大家看完之后能对SQL优化有更深入的理解，感谢大家的阅读。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！