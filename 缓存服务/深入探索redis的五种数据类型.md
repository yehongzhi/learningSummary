> **文章已收录Github精选，欢迎Star**：[https://github.com/yehongzhi](https://github.com/yehongzhi/learningSummary)

# 前言

Redis是一个**开源**的使用C语言编写、支持网络、可**基于内存**亦**可持久化**的日志型、**Key-Value**的NoSQL数据库。

一般来说，我们都是使用关系型数据库MySQL来存储数据，但是面对着流量高峰，会对MySQL造成巨大的压力，导致数据库性能很差，这时就要使用缓存中间件来降低数据库的压力，这是Redis最常见的使用场景。除了作为缓存使用之外，Redis还有很多使用场景，比如分布式锁，计数，队列等等。

所以Redis对于程序员来说可以算得上是必修课。

# 安装Redis

安装Redis很简单，因为网上教程很多，这里就不再详细讲解，推荐看菜鸟教程：https://www.runoob.com/redis/redis-install.html

# Redis的特点

要用好Redis，首先要明白它的特点：

- **读写速度快**。redis官网测试读写能到10万左右每秒。速度快的原因这里简单说一下，第一是因为**数据存储在内存中**，我们知道机器访问内存的速度是远远大于访问磁盘的，其次是**Redis采用单线程的架构**，避免了上下文的切换和多线程带来的竞争，也就不存在加锁释放锁的操作，减少了CPU的消耗，第三点是**采用了非阻塞IO多路复用机制**。
- **数据结构丰富**。 Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构。这也是这篇文章要讲的。
- **支持持久化**。Redis提供了RDB和AOF两种持久化策略，能最大限度地保证Redis服务器宕机重启后数据不会丢失。
- **支持高可用**。可以使用主从复制，并且提供哨兵机制，保证服务器的高可用。
- **客户端语言多**。因为Redis受到社区和各大公司的广泛认可，所以客户端语言涵盖了所有的主流编程语言，比如Java，C，C++，PHP，NodeJS等等。

# Redis的数据结构

下面我们就学习Redis的数据结构，也是使用Redis要知道的最基础的知识。

Redis是一个Key-Value型的内存数据库，它所有的key都是字符串，而value常见的数据类型有五种：string，list，set，zset，hash。

![](https://static.lovebilibili.com/redis_kv_01.png)

Redis的这些数据结构，在底层都是使用redisObject来进行表示。redisObject中有三个重要的属性，分别是**type、 encoding 和 ptr**。

**type**表示保存的value的类型。通常有以下几种，也就是常见的五种数据结构：

- 字符串 REDIS_STRING
- 列表 REDIS_LIST
- 集合 REDIS_SET
- 有序集合 REDIS_ZSET
- 字典 REDIS_HASH

**encoding**表示保存的value的编码，通常有以下几种：

```c
#define REDIS_ENCODING_RAW 0            // 编码为字符串
#define REDIS_ENCODING_INT 1            // 编码为整数
#define REDIS_ENCODING_HT 2             // 编码为哈希表
#define REDIS_ENCODING_ZIPMAP 3         // 编码为 zipmap
#define REDIS_ENCODING_LINKEDLIST 4     // 编码为双端链表
#define REDIS_ENCODING_ZIPLIST 5        // 编码为压缩列表
#define REDIS_ENCODING_INTSET 6         // 编码为整数集合
#define REDIS_ENCODING_SKIPLIST 7       // 编码为跳跃表
```

**ptr**是一个指针，指向实际保存的value的数据结构。

这里要特别说明一下的是，数据类型和编码方式是有一定关系的，所以数据类型和编码方式是可以确定底层采用什么数据结构存储数据的。

![](https://static.lovebilibili.com/redis_kv_02.png)

# string(字符串)

这是redis中最常用的数据类型，字符串对象的 encoding 有三种，分别是：**int、raw、embstr**。

常用的命令有常用命令:  **set、get、decr、incr、mget** 等。

我们知道Redis是用C语言开发的，但是底层存储不是使用C语言的字符串类型，而是自己开发了一种数据类型SDS进行存储，SDS即*Simple Dynamic String* ，是一种动态字符串。我们可以在github找到源码。

```c
struct sdshdr{
 int len;/*字符串长度*/
 int free;/*未使用的字节长度*/
 char buf[];/*保存字符串的字节数组*/
}
```

![](https://static.lovebilibili.com/redis_kv_03.png)

SDS与C语言的字符串有什么区别呢？

- C语言获取字符串长度是从头到尾遍历，时间复杂度是O(n)，而SDS有len属性记录字符串长度，时间复杂度为O(1)。
- 避免缓冲区溢出。SDS在需要修改时，会先检查空间是否满足大小，如果不满足，则先扩展至所需大小再进行修改操作。
- 空间预分配。当SDS需要进行扩展时，Redis会为SDS分配好内存，并且根据特定的算法分配多余的free空间，避免了连续执行字符串添加带来的内存分配的消耗。
- 惰性释放。如果需要缩短字符串，不会立即回收多余的内存空间，而是用free记录剩余的空间，以备下次扩展时使用，避免了再次分配内存的消耗。
- 二进制安全。c语言在存储字符串时采用N+1的字符串数组，末尾使用'\0'标识字符串的结束，如果我们存储的字符串中间出现'\0'，那就会导致识别出错。而SDS因为记录了字符串的长度len，则没有这个问题。

字符串类型的应用是非常广泛的，比如可以把对象转成JSON字符串存储到Redis中作为缓存，也可以使用decr、incr命令用于计数器的实现，又或者是用setnx命令为基础实现分布式锁等等。

需要注意的是：**Redis 规定了字符串的长度不得超过 512 MB。**

# hash(字典)

哈希对象的编码有两种，分别是：**ziplist、hashtable**。

当哈希对象保存的键值对数量小于 512，并且所有键值对的长度都小于 64 字节时，使用ziplist(压缩列表)存储；否则使用 hashtable 存储。

Redis中的hashtable跟Java中的HashMap类似，都是通过"数组+链表"的实现方式解决部分的哈希冲突。直接看源码定义。

```c
typedf struct dict{
    dictType *type;//类型特定函数，包括一些自定义函数，这些函数使得key和value能够存储
    void *private;//私有数据
    dictht ht[2];//两张hash表 
    int rehashidx;//rehash索引，字典没有进行rehash时，此值为-1
    unsigned long iterators; //正在迭代的迭代器数量
}dict;

typedef struct dictht{
     //哈希表数组
     dictEntry **table;
     //哈希表大小
     unsigned long size;
     //哈希表大小掩码，用于计算索引值
     //总是等于 size-1
     unsigned long sizemask;
     //该哈希表已有节点的数量
     unsigned long used; 
}dictht;

typedf struct dictEntry{
    void *key;//键
    union{
        void val;
        unit64_t u64;
        int64_t s64;
        double d;
    }v;//值
    struct dictEntry *next；//指向下一个节点的指针
}dictEntry;
```

我们再看一个结构图就比较清楚了。

![](https://static.lovebilibili.com/redis_kv_04.webp)

下面讲一下扩容和收缩。当哈希表保存的键值太多或者太少时，就会通过rehash来进行相应的扩容和收缩。

**扩容和收缩的过程**：

1、如果执行扩展操作，会基于原哈希表创建一个大小等于 ht[0].used*2n 的哈希表（也就是每次扩展都是根据原哈希表已使用的空间扩大一倍创建另一个哈希表）。相反如果执行的是收缩操作，每次收缩是根据已使用空间缩小一倍创建一个新的哈希表。

2、重新利用哈希算法，计算索引值，然后将键值对放到新的哈希表位置上。

3、所有键值对都迁徙完毕后，释放原哈希表的内存空间。

在redis中执行扩容和收缩的规则是：

- 服务器目前没有执行 BGSAVE 命令或者 BGREWRITEAOF (持久化)命令，并且负载因子大于等于1。

- 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF (持久化)命令，并且负载因子大于等于5。

负载因子 = 哈希表已保存节点数量 / 哈希表大小。

**渐进式rehash**

什么是渐进式，也就是说扩容和收缩不是一次性，集中式地完成，而是通过多次逐渐地完成的。为什么要采用这种方式呢？如果是几十个键值，那么rehash当然可以瞬间完成，如果是几十万，几百万的键值要一次性进行rehash，势必会导致redis性能严重下降，自然而然地redis开发者就想到采用渐进式rehash。过程如下：

在rehash时，会使用rehashidx字段保存迁移的进度，rehashidx为0表示迁移开始。

在迁移过程中ht[0]和ht[1]会同时保存数据，ht[0]指向旧哈希表，ht[1]指向新哈希表，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]的元素迁移到ht[1]中。

随着字典操作的不断执行，最终会在某个时间节点，ht[0]的所有键值都会被迁移到ht[1]中，rehashidx设置为-1，代表迁移完成。如果没有执行字典操作，redis也会通过定时任务去判断rehash是否完成，没有完成则继续rehash。

rehash完成后，ht[0]指向的旧表会被释放, 之后会将新表的持有权转交给ht[0], 再重置ht[1]指向NULL。

**渐进式rehash的优缺点**：

优点是把rehash操作分散到每一个字典操作和定时函数上，避免了一次性集中式rehash带来的服务器压力。

缺点是在rehash期间需要使用两个hash表，占用内存稍大。

hash类型的常用命令有：hget、hset、hgetall 等。

# list(链表)

列表对象的编码有两种，分别是：ziplist、linkedlist。当列表的长度小于 512，并且所有元素的长度都小于 64 字节时，使用ziplist存储；否则使用 linkedlist 存储。

Redis中的linkedlist类似于Java中的LinkedList，是一个链表，底层的实现原理也和LinkedList类似。这意味着list的插入和删除操作效率会比较快，时间复杂度是O(1)。我们看源码：

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

我们可以看到，listNode就是链表中的节点元素，通过prev和next组成双向链表。

![](https://static.lovebilibili.com/redis_kv_05.png)

list则记录了头结点head，尾结点tail，还有链表长度len，match函数用于比较两个节点的值是否相等，操作起来更加方便。

![](https://static.lovebilibili.com/redis_kv_06.png)

list类型常用的命令有：lpush、rpush、lpop、rpop、lrange等。



# set(集合)

set类型的特点很简单，无序，不重复，跟Java的HashSet类似。它的编码有两种，分别是intset和hashtable。如果value可以转成整数值，并且长度不超过512的话就使用intset存储，否则采用hashtable。

hashtable在前面讲hash类型时已经讲过，这里的set集合采用的hashtable几乎一样，只是哈希表的value都是NULL。这个不难理解，比如用Java中的HashMap实现一个HashSet，我们只用HashMap的key就是了。

我们讲一讲intset，先看源码。

```c
typedef struct intset{
    uint32_t encoding;//编码方式

    uint32_t length;//集合包含的元素数量

    int8_t contents[];//保存元素的数组
}intset;
```

encoding有三种，分别是INTSET_ENC_INT16、INSET_ENC_INT32、INSET_ENC_INT64，代表着整数值的取值范围。Redis会根据添加进来的元素的大小，选择不同的类型进行存储，可以尽可能地节省内存空间。

length记录集合有多少个元素，这样获取元素个数的时间复杂度就是O(1)。

contents，存储数据的数组，数组按照从小到大有序排列，不包含任何重复项。

![](https://static.lovebilibili.com/redis_kv_07.png)

这里我们可能会提出疑问，如果一开始存的是INTSET_ENC_INT16(范围在-32,768~32,767)，如果这时添加了一个40000的数，怎么升级为INSET_ENC_INT32呢？

升级过程是这样的：

1、根据新元素的类型扩展数组contents的空间。

2、从尾部将数据插入。

3、根据新的编码格式重置之前的值，因为这时的contents存在着两种编码的值。从插入的数据的位置，也就是尾部，从后到前将之前的数据按照新的编码格式进行移动和设置。从后到前调整是为了防止数据被覆盖。

升级的优点在于，根据存储的数据大小选择合适的编码方式，节省了内存。

缺点在于，升级会消耗系统资源。而且升级是不可逆的，也就是一旦对数组进行升级，编码就会一直保持升级后的状态。

set数据类型常用的命令有：sadd、spop、smembers、sunion等等。

Redis为set类型提供了求交集，并集，差集的操作，可以非常方便地实现譬如共同关注、共同爱好、共同好友等功能。

# zset(有序集合)

zset是Redis中比较有特色的数据类型，它和set一样是不可重复的，区别在于多了score值，用来代表排序的权重。也就是当你需要一个有序的，不可重复的集合列表时，就可以考虑使用这种数据类型。

zset的编码有两种，分别是：ziplist、skiplist。当zset的长度小于 128，并且所有元素的长度都小于 64 字节时，使用ziplist存储；否则使用 skiplist 存储。

这里要讲一下skiplist，也就是跳跃表。它的底层实现比较复杂，这里简单地提一下。

![](https://static.lovebilibili.com/redis_kv_08.png)

跳跃表的数据结构如上图所示，为什么要设计成这样呢？好处在于查询的时候，可以减少时间复杂度，如果是一个链表，我们要插入并且保持有序的话，那就要从头结点开始遍历，遍历到合适的位置然后插入，如果这样性能肯定是不理想的。

所以问题的关键在于**能不能像使用二分查找一样定位到插入的点**，答案就是使用跳跃表。比如我们要插入38，那么查找的过程就是这样。

首先从L4层，查询87，需要查询1次。

然后到L3层，查询到在->24->87之间，需要查询2次。

然后到L2层，查询->48，需要查询1次。

然后到L1层，查询->37->48，查询2次。确定在37->48之间是插入点。

有没有发现经过L4，L3，L2层的查询后已经跳过了很多节点，当到了L1层遍历时已经把范围缩小了很多。这就是跳跃表的优势。这种方式有点类似于二分查找，所以他的时间复杂度为**O(logN)**。

其实生活中也有这种例子，类似于快递填写的地址是省->市->区->镇->街，当快递公司在送快递时就根据地址层层缩小范围，最终锁定在一个很小的区域去搜索，提高了效率。

zet常用的命令有：zadd、zrange、zrem、zcard等。

zset的特点非常适合应用于开发排行榜的功能，比如三天阅读排行榜，游戏排行榜等等。

# 总结

Redis能够受到社区的认可，并且在互联网中如此欢迎，除了速度快之外，很大原因也跟丰富的数据类型有关，而且很多数据类型的底层实现也是会考虑到内存空间的使用，尽可能地节省内存空间。

其实很多人是知道Redis常用的五种数据类型，但是对于底层的实现，就没有深入去研究，当然我以前也是没有深入的。那么在面试时就没有产生差异化，要从面试中脱颖而出最重要就是要跟普通程序员不一样，这样才能突出自身的价值。所以深入学习Redis的数据结构还是很有用的。

希望这篇文章能让大家对Redis有更深入的理解，感谢大家的阅读。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**我是一个努力让大家记住的程序员。我们下期再见！！！**

![img](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！

