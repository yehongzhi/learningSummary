---
title: ZooKeeper入门
date: 2020-07-26 14:06:28
index_img: https://static.lovebilibili.com/ZooKeeper_index.jpg
tags:
	- zookeeper
	- java
---

# 思维导图

![](https://user-gold-cdn.xitu.io/2020/7/12/17343787ae8f3533?w=538&h=629&f=png&s=42414)

# 前言
在很多时候，我们都可以在各种框架应用中看到ZooKeeper的身影，比如Kafka中间件，Dubbo框架，Hadoop等等。为什么到处都看到ZooKeeper？

# 一、什么是ZooKeeper

**ZooKeeper是一个分布式服务协调框架**，提供了分布式数据一致性的解决方案，基于ZooKeeper的**数据结构，Watcher，选举机制**等特点，可以**实现数据的发布/订阅，软负载均衡，命名服务，统一配置管理，分布式锁，集群管理**等等。

# 二、为什么使用ZooKeeper

ZooKeeper能保证：

- 更新请求顺序进行。来自同一个client的更新请求按其发送顺序依次执行
- 数据更新原子性。一次数据更新要么成功，要么失败
- **全局唯一数据视图**。client无论连接到哪个server，数据视图都是一致的
- **实时性**。在一定时间范围内，client读到的数据是最新的

# 三、数据结构
ZooKeeper的数据结构和Unix文件系统很类似，总体上可以看做是一棵树，每一个节点称之为一个ZNode，每一个ZNode**默认能存储1M的数据**。每一个ZNode可**通过唯一的路径标识**。如下图所示：

![](https://user-gold-cdn.xitu.io/2020/7/9/173343f042cc21bf?w=450&h=317&f=png&s=19615)

创建ZNode时，可以指定以下四种类型，包括：

- **PERSISTENT，持久性ZNode**。创建后，即使客户端与服务端断开连接也不会删除，只有客户端主动删除才会消失。
- **PERSISTENT_SEQUENTIAL，持久性顺序编号ZNode**。和持久性节点一样不会因为断开连接后而删除，并且ZNode的编号会自动增加。
- **EPHEMERAL，临时性ZNode**。客户端与服务端断开连接，该ZNode会被删除。
- **EPEMERAL_SEQUENTIAL，临时性顺序编号ZNode**。和临时性节点一样，断开连接会被删除，并且ZNode的编号会自动增加。

# 四、监听通知机制

Watcher是基于**观察者模式**实现的一种机制。如果我们需要实现当某个ZNode节点发生变化时收到通知，就可以使用Watcher监听器。

**客户端通过设置监视点（watcher）向 ZooKeeper 注册需要接收通知的 znode，在 znode 发生变化时 ZooKeeper 就会向客户端发送消息**。

**这种通知机制是一次性的**。一旦watcher被触发，ZooKeeper就会从相应的存储中删除。如果需要不断监听ZNode的变化，可以在收到通知后再设置新的watcher注册到ZooKeeper。

监视点的类型有很多，如**监控ZNode数据变化、监控ZNode子节点变化、监控ZNode 创建或删除**。

# 五、选举机制
ZooKeeper是一个高可用的应用框架，因为ZooKeeper是支持集群的。ZooKeeper在集群状态下，配置文件是不会指定Master和Slave，而是在ZooKeeper服务器初始化时就在内部进行选举，产生一台做为Leader，多台做为Follower，并且遵守半数可用原则。

由于遵守半数可用原则，所以5台服务器和6台服务器，实际上最大允许宕机数量都是3台，所以为了节约成本，**集群的服务器数量一般设置为奇数**。

如果在运行时，**如果长时间无法和Leader保持连接的话，则会再次进行选举，产生新的Leader，以保证服务的可用**。

![](https://user-gold-cdn.xitu.io/2020/7/10/17339067a2c6c061?w=600&h=185&f=png&s=87756)

# 六、初の体验

首先在[官网](https://zookeeper.apache.org/releases.html)下载ZooKeeper，我这里用的是3.3.6版本。

然后解压，复制一下/conf目录下的zoo_sample.cfg文件，重命名为zoo.cfg。

修改zoo.cfg中dataDir的值，并创建对应的目录：

![](https://user-gold-cdn.xitu.io/2020/7/10/17339300106f99c6?w=515&h=245&f=png&s=18915)

最后到/bin目录下启动，我用的是window系统，所以启动zkServer.cmd，双击即可：

![](https://user-gold-cdn.xitu.io/2020/7/10/1733937df5bee4e3?w=663&h=180&f=png&s=28318)

启动成功的话就可以看到这个对话框：

![](https://user-gold-cdn.xitu.io/2020/7/10/17339369078c2cc2?w=961&h=407&f=png&s=51377)

可视化界面的话，我推荐使用`ZooInspector`，操作比较简便：

![](https://user-gold-cdn.xitu.io/2020/7/10/173393a9befbac48?w=767&h=156&f=png&s=10149)

## 6.1 使用java连接ZooKeeper

首先引入Maven依赖：
```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.6</version>
</dependency>
```
接着我们写一个Main方法，进行操作：

```java
    //连接地址及端口号
    private static final String SERVER_HOST = "127.0.0.1:2181";

    //会话超时时间
    private static final int SESSION_TIME_OUT = 2000;

    public static void main(String[] args) throws Exception {
        //参数一：服务端地址及端口号
        //参数二：超时时间
        //参数三：监听器
        ZooKeeper zooKeeper = new ZooKeeper(SERVER_HOST, SESSION_TIME_OUT, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                //获取事件的状态
                Event.KeeperState state = watchedEvent.getState();
                //判断是否是连接事件
                if (Event.KeeperState.SyncConnected == state) {
                    Event.EventType type = watchedEvent.getType();
                    if (Event.EventType.None == type) {
                        System.out.println("zk客户端已连接...");
                    }
                }
            }
        });
        zooKeeper.create("/java", "Hello World".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        System.out.println("新增ZNode成功");
        zooKeeper.close();
    }
```

创建一个持久性ZNode，路径是/java，值为"Hello World"：

![](https://user-gold-cdn.xitu.io/2020/7/10/17339630f8746e96?w=443&h=168&f=png&s=10357)

# 七、API概述

## 7.1 创建
```java
public String create(final String path, byte data[], List<ACL> acl, CreateMode createMode)
```
参数解释：

- path ZNode路径
- data ZNode存储的数据
- acl ACL权限控制
- createMode ZNode类型

ACL权限控制，有三个是ZooKeeper定义的常用权限，在ZooDefs.Ids类中：
```java
/**
 * This is a completely open ACL.
 * 完全开放的ACL，任何连接的客户端都可以操作该属性znode
 */
public final ArrayList<ACL> OPEN_ACL_UNSAFE = new ArrayList<ACL>(Collections.singletonList(new ACL(Perms.ALL, ANYONE_ID_UNSAFE)));

/**
 * This ACL gives the creators authentication id's all permissions.
 * 只有创建者才有ACL权限
 */
public final ArrayList<ACL> CREATOR_ALL_ACL = new ArrayList<ACL>(Collections.singletonList(new ACL(Perms.ALL, AUTH_IDS)));

/**
 * This ACL gives the world the ability to read.
 * 只能读取ACL
 */
public final ArrayList<ACL> READ_ACL_UNSAFE = new ArrayList<ACL>(Collections.singletonList(new ACL(Perms.READ, ANYONE_ID_UNSAFE)));
```

createMode就是前面讲过的四种ZNode类型：
```java
public enum CreateMode {
    /**
     * 持久性ZNode
     */
    PERSISTENT (0, false, false),
    /**
     * 持久性自动增加顺序号ZNode
     */
    PERSISTENT_SEQUENTIAL (2, false, true),
    /**
     * 临时性ZNode
     */
    EPHEMERAL (1, true, false),
    /**
     * 临时性自动增加顺序号ZNode
     */
    EPHEMERAL_SEQUENTIAL (3, true, true);
}
```

## 7.2 查询

```java
//同步获取节点数据
public byte[] getData(String path, boolean watch, Stat stat){
    ...
}

//异步获取节点数据
public void getData(final String path, Watcher watcher, DataCallback cb, Object ctx){
    ...
}
```
同步getData()方法中的stat参数是用于接收返回的节点描述信息：
```java
public byte[] getData(final String path, Watcher watcher, Stat stat){
    //省略...
    GetDataResponse response = new GetDataResponse();
    //发送请求到ZooKeeper服务器，获取到response
    ReplyHeader r = cnxn.submitRequest(h, request, response, wcb);
    if (stat != null) {
        //把response的Stat赋值到传入的stat中
        DataTree.copyStat(response.getStat(), stat);
    }
}
```

使用同步getData()获取数据：
```java
    //数据的描述信息，包括版本号，ACL权限，子节点信息等等
    Stat stat = new Stat();
    //返回结果是byte[]数据，getData()方法底层会把描述信息复制到stat对象中
    byte[] bytes = zooKeeper.getData("/java", false, stat);
    //打印结果
    System.out.println("ZNode的数据data:" + new String(bytes));//Hello World
    System.out.println("获取到dataVersion版本号:" + stat.getVersion());//默认数据版本号是0
```

## 7.3 更新

```java
public Stat setData(final String path, byte data[], int version){
    ...
}
```

值得注意的是第三个参数version，**使用CAS机制，这是为了防止多个客户端同时更新节点数据，所以需要在更新时传入版本号，每次更新都会使版本号+1**，如果服务端接收到版本号，对比发现不一致的话，则会抛出异常。

所以，在更新前需要先查询获取到版本号，否则你不知道当前版本号是多少，就没法更新：

```java
    //获取节点描述信息
    Stat stat = new Stat();
    zooKeeper.getData("/java", false, stat);
    System.out.println("更新ZNode数据...");
    //更新操作，传入路径，更新值，版本号三个参数,返回结果是新的描述信息
    Stat setData = zooKeeper.setData("/java", "fly!!!".getBytes(), stat.getVersion());
    System.out.println("更新后的版本号为：" + setData.getVersion());//更新后的版本号为：1
```

更新后，版本号增加了：

![](https://user-gold-cdn.xitu.io/2020/7/11/1733999126b62716?w=440&h=320&f=png&s=19876)

如果传入的版本参数是"-1"，就是告诉zookeeper服务器，客户端需要基于数据的最新版本进行更新操作。但是-1并不是一个合法的版本号，而是一个标识符。

## 7.4 删除

```java
public void delete(final String path, int version){
    ...
}
```
- path 删除节点的路径
- version 版本号

这里也需要传入版本号，调用getData()方法即可获取到版本号，很简单：

```java
Stat stat = new Stat();
zooKeeper.getData("/java", false, stat);
//删除ZNode
zooKeeper.delete("/java", stat.getVersion());
```

## 7.5 watcher机制

在上面第三点提到，ZooKeeper是可以使用通知监听机制，当ZNode发生变化会收到通知消息，进行处理。基于watcher机制，ZooKeeper能玩出很多花样。怎么使用？

ZooKeeper的通知监听机制，总的来说可以分为三个过程：

①客户端注册 Watcher
②服务器处理 Watcher
③客户端回调 Watcher客户端。

注册 watcher 有 4 种方法，new ZooKeeper()、getData()、exists()、getChildren()。下面演示一下使用exists()方法注册watcher：

首先**需要实现Watcher接口**，新建一个监听器：

```java
public class MyWatcher implements Watcher {
    @Override
    public void process(WatchedEvent event) {
        //获取事件类型
        Event.EventType eventType = event.getType();
        //通知状态
        Event.KeeperState eventState = event.getState();
        //节点路径
        String eventPath = event.getPath();
        System.out.println("监听到的事件类型:" + eventType.name());
        System.out.println("监听到的通知状态:" + eventState.name());
        System.out.println("监听到的ZNode路径:" + eventPath);
    }
}
```
然后调用exists()方法，注册监听器：
```java
zooKeeper.exists("/java", new MyWatcher());
//对ZNode进行更新数据的操作，触发监听器
zooKeeper.setData("/java", "fly".getBytes(), -1);
```

然后在控制台就可以看到打印的信息：

![](https://user-gold-cdn.xitu.io/2020/7/11/1733cd85f8b2bb10?w=682&h=200&f=png&s=48245)

这里我们可以看到**Event.EventType对象就是事件类型**，我们可以对事件类型进行判断，再配合**Event.KeeperState通知状态**，做相关的业务处理，事件类型有哪些？

打开EventType、KeeperState的源码查看：

```java
//事件类型是一个枚举
public enum EventType {
    None (-1),//无
    NodeCreated (1),//Watcher监听的数据节点被创建
    NodeDeleted (2),//Watcher监听的数据节点被删除
    NodeDataChanged (3),//Watcher监听的数据节点内容发生变更
    NodeChildrenChanged (4);//Watcher监听的数据节点的子节点列表发生变更
}

//通知状态也是一个枚举
public enum KeeperState {
    Unknown (-1),//属性过期
    Disconnected (0),//客户端与服务端断开连接
    NoSyncConnected (1),//属性过期
    SyncConnected (3),//客户端与服务端正常连接
    AuthFailed (4),//身份认证失败
    ConnectedReadOnly (5),//返回这个状态给客户端，客户端只能处理读请求
    SaslAuthenticated(6),//服务器采用SASL做校验时
    Expired (-112);//会话session失效
}
```

### 7.5.1 watcher的特性

- 一次性。一旦watcher被触发，ZK都会从相应的存储中移除。

```java
    zooKeeper.exists("/java", new Watcher() {
        @Override
        public void process(WatchedEvent event) {
            System.out.println("我是exists()方法的监听器");
        }
    });
    //对ZNode进行更新数据的操作，触发监听器
    zooKeeper.setData("/java", "fly".getBytes(), -1);
    //企图第二次触发监听器
    zooKeeper.setData("/java", "spring".getBytes(), -1);
```

![](https://user-gold-cdn.xitu.io/2020/7/11/1733cfc7501b447c?w=476&h=149&f=png&s=27289)

- 串行执行。客户端Watcher回调的过程是一个串行同步的过程，这是为了保证顺序。
```java
    zooKeeper.exists("/java", new Watcher() {
        @Override
        public void process(WatchedEvent event) {
            System.out.println("我是exists()方法的监听器");
        }
    });
    Stat stat = new Stat();
    zooKeeper.getData("/java", new Watcher() {
        @Override
        public void process(WatchedEvent event) {
            System.out.println("我是getData()方法的监听器");
        }
    }, stat);
    //对ZNode进行更新数据的操作，触发监听器
    zooKeeper.setData("/java", "fly".getBytes(), stat.getVersion());
```
打印结果，说明先调用exists()方法的监听器，再调用getData()方法的监听器。因为exists()方法先注册了。

![](https://user-gold-cdn.xitu.io/2020/7/11/1733cf5f77df73f1?w=541&h=140&f=png&s=27151)

- 轻量级。WatchedEvent是ZK整个Watcher通知机制的最小通知单元。WatchedEvent包含三部分：**通知状态，事件类型，节点路径**。Watcher通知仅仅告诉客户端发生了什么事情，而不会说明事件的具体内容。

# 写在最后

我记得B站的UP主李永乐说过，**只有你让更多的人生活变得更美好时，自己的生活才能变得更美好。**

![](https://user-gold-cdn.xitu.io/2020/7/11/1733d114535c1c80?w=631&h=127&f=png&s=23550)

这句话也是我今年开始写技术分享的一个动力源泉，希望这篇文章对你有用~

著名的飞行家**马老师**说过：**回城是收费的，而点赞是免费的~**

![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/6/30/17305cc08a7ed5d7?w=1180&h=528&f=png&s=152520)