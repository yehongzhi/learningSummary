---
title: 从秒杀聊到ZooKeeper分布式锁
date: 2020-07-26 14:03:52
index_img: https://static.lovebilibili.com/zookeeper_lock_index.jpg
tags:
	- zookeeper
	- 分布式锁
---

# 思维导图

![](https://user-gold-cdn.xitu.io/2020/7/19/17365f514c2871a0?w=846&h=397&f=png&s=36190)

# 前言

经过[《ZooKeeper入门》](https://juejin.im/post/5f05e96c5188252e5e22d8f4)后，我们学会了ZooKeeper的基本用法。

实际上ZooKeeper的应用是非常广泛的，实现分布式锁只是其中一种。接下来我们就ZooKeeper实现分布式锁解决**秒杀超卖问题**进行展开。

# 一、什么是秒杀超卖问题

秒杀活动应该都不陌生，不用过多解释。

不难想象，在这种"秒杀"的场景中，实际上会出现多个用户争抢"资源"的情况，**也就是多个线程同时并发，这种情况是很容易出现数据不准确，也就是超卖问题**。

## 1.1 项目演示
下面使用程序演示，我使用了**SpringBoot2.0、Mybatis、Mybatis-Plus、SpringMVC**搭建了一个简单的项目，github地址：
> https://github.com/yehongzhi/mall

创建一个商品信息表：
```sql
CREATE TABLE `tb_commodity_info` (
  `id` varchar(32) NOT NULL,
  `commodity_name` varchar(512) DEFAULT NULL COMMENT '商品名称',
  `commodity_price` varchar(36) DEFAULT '0' COMMENT '商品价格',
  `number` int(10) DEFAULT '0' COMMENT '商品数量',
  `description` varchar(2048) DEFAULT '' COMMENT '商品描述',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品信息表';
```
添加一个商品[**叉烧包**]进去：
![](https://user-gold-cdn.xitu.io/2020/7/15/1734e32e530bc75a?w=770&h=45&f=png&s=6596)

核心的代码逻辑是这样的：
```java
    @Override
    public boolean purchaseCommodityInfo(String commodityId, Integer number) throws Exception {
        //1.先查询数据库中商品的数量
        TbCommodityInfo commodityInfo = commodityInfoMapper.selectById(commodityId);
        //2.判断商品数量是否大于0，或者购买的数量大于库存
        Integer count = commodityInfo.getNumber();
        if (count <= 0 || number > count) {
            //商品数量小于或者等于0，或者购买的数量大于库存，则返回false
            return false;
        }
        //3.如果库存数量大于0，并且购买的数量小于或者等于库存。则更新商品数量
        count -= number;
        commodityInfo.setNumber(count);
        boolean bool = commodityInfoMapper.updateById(commodityInfo) == 1;
        if (bool) {
            //如果更新成功，则打印购买商品成功
            System.out.println("购买商品[ " + commodityInfo.getCommodityName() + " ]成功,数量为：" + number);
        }
        return bool;
    }
```

逻辑示意图如下：

![](https://user-gold-cdn.xitu.io/2020/7/15/1734e422a298bdef?w=719&h=411&f=png&s=35372)

上面这个逻辑，如果单线程请求的话是没有问题的。

但是多线程的话就出现问题了。现在我就创建多个线程，通过HttpClient进行请求，看会发生什么：
```java
    public static void main(String[] args) throws Exception {
        //请求地址
        String url = "http://localhost:8080/mall/commodity/purchase";
        //请求参数，商品ID，数量
        Map<String, String> map = new HashMap<>();
        map.put("commodityId", "4f863bb5266b9508e0c1f28c61ea8de1");
        map.put("number", "1");
        //创建10个线程通过HttpClient进行发送请求，测试
        for (int i = 0; i < 10; i++) {
            //这个线程的逻辑仅仅是发送请求
            CommodityThread commodityThread = new CommodityThread(url, map);
            commodityThread.start();
        }
    }
```
说明一下，叉烧包的数量是100，这里有10个线程同时去购买，假设都购买成功的话，库存数量应该是90。

实际上，10个线程的确都购买成功了：

![](https://user-gold-cdn.xitu.io/2020/7/15/1734e52bcd8c8410?w=348&h=182&f=png&s=10183)

但是数据库的商品库存，却不准确：

![](https://user-gold-cdn.xitu.io/2020/7/15/1734e5372723db05?w=777&h=47&f=png&s=7006)

# 二、尝试使用本地锁

上面的场景，大概流程如下所示：

![](https://user-gold-cdn.xitu.io/2020/7/16/173576192aaa3672?w=827&h=458&f=png&s=71783)

可以看出问题的**关键在于两个线程"同时"去查询剩余的库存，然后更新库存导致的**。要解决这个问题，其实**只要保证多个线程在这段逻辑是顺序执行即可，也就是加锁**。

本地锁JDK提供有两种：synchronized和Lock锁。

两种方式都可以，我这里为了简便，使用synchronized：
```java
    //使用synchronized修饰方法
    @Override
    public synchronized boolean purchaseCommodityInfo(String commodityId, Integer number) throws Exception {
        //省略...
    }
```

然后再测试刚刚多线程并发抢购的情况，看看结果：

![](https://user-gold-cdn.xitu.io/2020/7/15/17352d90c1ca872c?w=779&h=50&f=png&s=6771)

问题得到解决！！！

你以为事情就这样结束了吗，看了看进度条，发现事情并不简单。

我们知道在实际项目中，往往不会只部署一台服务器，所以不妨我们启动两台服务器，端口号分别是8080、8081，模拟实际项目的场景：

![](https://user-gold-cdn.xitu.io/2020/7/15/17352e08465b09fb?w=592&h=391&f=png&s=38702)

写一个交替请求的测试脚本，模拟多台服务器分别处理请求，用户秒杀抢购的场景：
```java
    public static void main(String[] args) throws Exception {
        //请求地址
        String url = "http://localhost:%s/mall/commodity/purchase";
        //请求参数，商品ID，数量
        Map<String, String> map = new HashMap<>();
        map.put("commodityId", "4f863bb5266b9508e0c1f28c61ea8de1");
        map.put("number", "1");
        //创建10个线程通过HttpClient进行发送请求，测试
        for (int i = 0; i < 10; i++) {
            //8080、8081交替请求，每个服务器处理5个请求
            String port = "808" + (i % 2);
            CommodityThread commodityThread = new CommodityThread(String.format(url, port), map);
            commodityThread.start();
        }
    }
```

首先看购买的情况，肯定都是购买成功的:

![](https://user-gold-cdn.xitu.io/2020/7/15/17352ecdc9af8253?w=615&h=302&f=png&s=36798)

关键是库存数量是否正确：

![](https://user-gold-cdn.xitu.io/2020/7/15/17352edee5171303?w=786&h=50&f=png&s=7086)

有10个请求购买成功，库存应该是90才对，这里库存是95。事实证明**本地锁是不能解决多台服务器秒杀抢购出现超卖的问题**。

为什么会这样呢，请看示意图：

![](https://user-gold-cdn.xitu.io/2020/7/15/17352fc7c52787c9?w=650&h=458&f=png&s=63889)

其实和多线程问题是差不多的原因，**多个服务器去查询数据库，获取到相同的库存，然后更新库存，导致数据不正确**。要保证库存的数量正确，**关键在于多台服务器要保证只能一台服务器在执行这段逻辑**，也就是要加分布式锁。

这也体现出分布式锁的作用，就是要保证多台服务器只能有一台服务器执行。

分布式锁有三种实现方式，分别是redis、ZooKeeper、数据库(比如mysql)。

# 三、使用ZooKeeper实现分布式锁

## 3.1 原理

实际上是利用ZooKeeper的临时顺序节点的特性实现分布式锁。怎么实现呢？

假设现在有一个客户端A，需要加锁，那么就在"/Lock"路径下创建一个临时顺序节点。然后获取"/Lock"下的节点列表，判断自己的序号是否是最小的，如果是最小的序号，则加锁成功！

![](https://user-gold-cdn.xitu.io/2020/7/15/17353229075ad04f?w=698&h=276&f=png&s=23589)

现在又有另一个客户端，客户端B需要加锁，那么也是在"/Lock"路径下创建临时顺序节点。依然获取"/Lock"下的节点列表，判断自己的节点序号是否最小的。发现不是最小的，加锁失败，接着对自己的上一个节点进行监听。

![](https://user-gold-cdn.xitu.io/2020/7/15/173532ccde8ed6b4?w=736&h=328&f=png&s=29464)

怎么释放锁呢，其实就是把临时节点删除。假设客户端A释放锁，把节点01删除了。那就会触发节点02的监听事件，客户端就再次获取节点列表，然后判断自己是否是最小的序号，如果是最小序号则加锁。

![](https://user-gold-cdn.xitu.io/2020/7/16/1735337578204cc5?w=788&h=314&f=png&s=33283)

如果多个客户端其实也是一样，一上来就会创建一个临时节点，然后开始判断自己是否是最小的序号，如果不是就监听上一个节点，形成一种排队的机制。也就形成了锁的效果，保证了多台服务器只有一台执行。

**假设其中有一个客户端宕机了，根据临时节点的特点，ZooKeeper会自动删除对应的临时节点**，相当于自动释放了锁。

## 3.2 手写代码实现分布式锁

首先加入Maven依赖

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.6</version>
</dependency>
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.4</version>
</dependency>
```
接着按照上面分析的思路敲代码，创建ZkLock类：
```java
public class ZkLock implements Lock {
    //计数器，用于加锁失败时，阻塞
    private static CountDownLatch cdl = new CountDownLatch(1);
    //ZooKeeper服务器的IP端口
    private static final String IP_PORT = "127.0.0.1:2181";
    //锁的根路径
    private static final String ROOT_NODE = "/Lock";
    //上一个节点的路径
    private volatile String beforePath;
    //当前上锁的节点路径
    private volatile String currPath;
    //创建ZooKeeper客户端
    private ZkClient zkClient = new ZkClient(IP_PORT);

    public ZkLock() {
        //判断是否存在根节点
        if (!zkClient.exists(ROOT_NODE)) {
            //不存在则创建
            zkClient.createPersistent(ROOT_NODE);
        }
    }
    
    //加锁
    public void lock() {
        if (tryLock()) {
            System.out.println("加锁成功！！");
        } else {
            // 尝试加锁失败，进入等待 监听
            waitForLock();
            // 再次尝试加锁
            lock();
        }

    }
    
    //尝试加锁
    public synchronized boolean tryLock() {
        // 第一次就进来创建自己的临时节点
        if (StringUtils.isBlank(currPath)) {
            currPath = zkClient.createEphemeralSequential(ROOT_NODE + "/", "lock");
        }
        // 对节点排序
        List<String> children = zkClient.getChildren(ROOT_NODE);
        Collections.sort(children);

        // 当前的是最小节点就返回加锁成功
        if (currPath.equals(ROOT_NODE + "/" + children.get(0))) {
            return true;
        } else {
            // 不是最小节点 就找到自己的前一个 依次类推 释放也是一样
            int beforePathIndex = Collections.binarySearch(children, currPath.substring(ROOT_NODE.length() + 1)) - 1;
            beforePath = ROOT_NODE + "/" + children.get(beforePathIndex);
            //返回加锁失败
            return false;
        }
    }
    
    //解锁
    public void unlock() {
        //删除节点并关闭客户端
        zkClient.delete(currPath);
        zkClient.close();
    }
    
    //等待上锁，加锁失败进入阻塞，监听上一个节点
    private void waitForLock() {
        IZkDataListener listener = new IZkDataListener() {
            //监听节点更新事件
            public void handleDataChange(String s, Object o) throws Exception {
            }

            //监听节点被删除事件
            public void handleDataDeleted(String s) throws Exception {
                //解除阻塞
                cdl.countDown();
            }
        };
        // 监听上一个节点
        this.zkClient.subscribeDataChanges(beforePath, listener);
        //判断上一个节点是否存在
        if (zkClient.exists(beforePath)) {
            //上一个节点存在
            try {
                System.out.println("加锁失败 等待");
                //加锁失败，阻塞等待
                cdl.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        // 释放监听
        zkClient.unsubscribeDataChanges(beforePath, listener);
    }

    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return false;
    }

    public void lockInterruptibly() throws InterruptedException {
    }

    public Condition newCondition() {
        return null;
    }
}
```
在Controller层加上锁：
```java
    @PostMapping("/purchase")
    public boolean purchaseCommodityInfo(@RequestParam(name = "commodityId") String commodityId, @RequestParam(name = "number") Integer number) throws Exception {
        boolean bool;
        //获取ZooKeeper分布式锁
        ZkLock zkLock = new ZkLock();
        try {
            //上锁
            zkLock.lock();
            //调用秒杀抢购的service方法
            bool = commodityInfoService.purchaseCommodityInfo(commodityId, number);
        } catch (Exception e) {
            e.printStackTrace();
            bool = false;
        } finally {
            //解锁
            zkLock.unlock();
        }
        return bool;
    }
```
测试，依然起两台服务器，8080、8081。然后跑测试脚本：
```java
    public static void main(String[] args) throws Exception {
        //请求地址
        String url = "http://localhost:%s/mall/commodity/purchase";
        //请求参数，商品ID，数量
        Map<String, String> map = new HashMap<>();
        map.put("commodityId", "4f863bb5266b9508e0c1f28c61ea8de1");
        map.put("number", "1");
        //创建10个线程通过HttpClient进行发送请求，测试
        for (int i = 0; i < 10; i++) {
            //8080、8081交替请求
            String port = "808" + (i % 2);
            CommodityThread commodityThread = new CommodityThread(String.format(url, port), map);
            commodityThread.start();
        }
    }
```
结果正确：

![](https://user-gold-cdn.xitu.io/2020/7/18/173623e9dd3cc00b?w=776&h=47&f=png&s=7044)

## 3.3 造好的轮子
Curator是Apache开源的一个操作ZooKeeper的框架。其中就有实现ZooKeeper分布式锁的功能。

当然分布式锁的实现只是这个框架的其中一个很小的部分，除此之外还有很多用途，大家可以到[官网](http://curator.apache.org/)去学习。

首先添加Maven依赖：
```java
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>4.3.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>4.3.0</version>
    </dependency>
```
还是一样在需要加锁的地方进行加锁：

```java
    @PostMapping("/purchase")
    public boolean purchaseCommodityInfo(@RequestParam(name = "commodityId") String commodityId,
                                         @RequestParam(name = "number") Integer number) throws Exception {
        boolean bool = false;
        //设置重试策略
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", retryPolicy);
        // 启动客户端
        client.start();
        InterProcessMutex mutex = new InterProcessMutex(client, "/locks");
        try {
            //加锁
            if (mutex.acquire(3, TimeUnit.SECONDS)) {
                //调用抢购秒杀service方法
                bool = commodityInfoService.purchaseCommodityInfo(commodityId, number);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            //解锁
            mutex.release();
            client.close();
        }
        return bool;
    }
```

# 四、遇到的坑

我尝试用原生的ZooKeeper写分布式锁，有点炸裂。遇到不少坑，最终放弃了，用zkclient的API。可能我太菜了不太会用。

下面我分享我遇到的一些问题，希望你们在遇到同类型的异常时能迅速定位问题。

## 4.1 Session expired

这个错误是使用原生ZooKeeper的API出现的错误。主要是我在进入debug模式进行调试出现的。

因为原生的ZooKeeper需要设定一个会话超时时间，一般debug模式我们都会卡在一个地方去调试，肯定就超出了设置的会话时间~

![](https://user-gold-cdn.xitu.io/2020/7/18/1736258e60ae7778?w=742&h=434&f=png&s=58282)

## 4.2 KeeperErrorCode = ConnectionLoss

这个也是原生ZooKeeper的API的错误，怎么出现的呢？

主要是创建的ZooKeeper客户端连接服务器时是异步的，由于连接需要时间，还没连接成功，代码已经开始执行create()或者exists()，然后就报这个错误。

解决方法：使用CountDownLatch计数器阻塞，连接成功后再停止阻塞，然后执行create()或者exists()等操作。

## 4.3 并发查询更新出现数据不一致

这个错误真的太炸裂了~

一开始我是把分布式锁加在service层，然后以为搞定了。接着启动8080、8081进行并发测试。10个线程都是购买成功，结果居然是不正确！

![](https://user-gold-cdn.xitu.io/2020/7/18/173627032c88e91b?w=876&h=184&f=png&s=26788)

第一反应觉得自己实现的代码有问题，于是换成curator框架实现的分布式锁，开源框架应该没问题了吧。没想到还是不行~

既然不是锁本身的问题，是不是事务问题。**上一个事务更新库存的操作还没提交，然后下一个请求就进来查询。于是我就把加锁的范围放大一点，放在Controller层**。居然成功了！

你可能已经注意到，我在上面的例子就是把分布式锁加在Controller层，其实我不太喜欢在Controller层写太多代码。

也许有更加优雅的方式，可惜本人能力不足，如果你有更好的实现方式，可以分享一下~

补充：下面评论有位大佬说，在原来的方法外再包裹一层，亲测是可以的。这应该是事务的问题。

上面放在Controller层可以成功是不是因为Controller层没有事务，原来写在service我是写了一个@Transactional注解在类上，所以整个类里面的都有事务，所以解锁后还没提交事务去更新数据库，然后下一个请求进来就查到了没更新的数据。

为了优雅一点，就把@Transactional注解放在抢购的service方法上
![](https://user-gold-cdn.xitu.io/2020/7/20/1736c9d98626282d?w=863&h=230&f=png&s=49347)

然后再包裹一个没有事务的方法，用于上锁。
![](https://user-gold-cdn.xitu.io/2020/7/20/1736c993c04ea9e5?w=873&h=378&f=png&s=62335)

![](https://user-gold-cdn.xitu.io/2020/7/20/1736c9a2d2881ae3?w=846&h=248&f=png&s=44367)

# 五、总结

最后，我们回顾总结一下吧：

- 首先我们模拟单机多线程的秒杀场景，单机的话可以使用本地锁解决问题。
- 接着模拟多服务器多线程的场景，思路是使用ZooKeeper实现分布式锁解决。
- 图解ZooKeeper实现分布式锁的原理。
- 然后动手写代码，实现分布式锁。
- 最后总结遇到的坑。

**希望这篇文章对你有用**

想第一时间看到我更新的文章，可以微信搜索公众号「`java技术爱好者`」，**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**
![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/6/30/17305cc08a7ed5d7?w=1180&h=528&f=png&s=152520)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！