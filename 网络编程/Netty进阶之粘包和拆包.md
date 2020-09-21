---
title: Netty进阶之粘包和拆包
date: 2020-07-08 00:20:05
index_img: https://static.lovebilibili.com/nianbao_index.jpg
tags:
	- java
	- Netty
---

# 思维导图

![](https://user-gold-cdn.xitu.io/2020/7/7/17329e0e4be94869?w=895&h=340&f=png&s=33550)

# 一、什么是粘包和拆包
TCP是一种**面向连接的、可靠的、基于字节流**的传输层通信协议。(来自百度百科)

发送端为了将多个发给接收端的数据包，更有效地发送到接收端，会使用**Nagle算法**。Nagle算法会**将多次时间间隔较小且数据量小的数据合并成一个大的数据块**进行发送。虽然这样的确提高了效率，但是**因为面向流通信，数据是无消息保护边界的**，就会**导致接收端难以分辨出完整的数据包**了。

所谓的粘包和拆包问题，就是因为TCP消息无保护边界导致的。

<!-- more -->

## 1.1 图解粘包和拆包

![](https://user-gold-cdn.xitu.io/2020/7/5/1731e5cfafa34ed7?w=774&h=456&f=png&s=36343)
正常发送消息是三次发送三个数据包，这种情况没有问题。

粘包，则是其中有多个数据包合并成一个数据包进行发送，也就是上图的第二种情况。

拆包，则是其中一个数据包被拆成了多段，发送的数据包只包含了一个完整数据包的一部分。也就是上图的第三种情况。

## 1.2 程序演示
首先准备客户端负责发送消息，连续发送5次消息，代码如下：
```java
    for (int i = 1; i <= 5; i++) {
        ByteBuf byteBuf = Unpooled.copiedBuffer("msg No" + i + " ", Charset.forName("utf-8"));
        ctx.writeAndFlush(byteBuf);
    }
```
然后服务端作为接收方，接收并且打印结果：
```java
//count变量，用于计数
private int count = 0;

@Override
protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
    byte[] bytes = new byte[msg.readableBytes()];
    //把ByteBuf的数据读到bytes数组中
    msg.readBytes(bytes);
    String message = new String(bytes, Charset.forName("utf-8"));
    System.out.println("服务器接收到数据：" + message);
    //打印接收的次数
    System.out.println("接收到的数据量是：" + (++this.count));
}
```
启动服务端，再启动两个客户端发送消息,服务端的控制台可以看到这样：

![](https://user-gold-cdn.xitu.io/2020/7/5/1731f27303054a8f?w=719&h=199&f=png&s=28205)

粘包的问题其实是随机的，所以每次结果都不太一样。

# 二、解决方案
总体思路可以分为三种：

- 在数据的末尾添加特殊的符号标识数据包的边界。通常会加\n\r、\t或者其他的符号。
- 在数据的头部声明数据的长度，按长度获取数据。
- 规定报文的长度，不足则补空位。读取时按规定好的长度来读取。

## 2.1 使用LineBasedFrameDecoder
这是Netty内置的一个解码器，对应的编码器是LineEncoder。

原理是上面讲的第一种思路，在数据末尾加上特殊符号以标识边界。默认是使用换行符\n。

用法很简单，发送方加上编码器：
```java
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        //添加编码器，使用默认的符号\n，字符集是UTF-8
        ch.pipeline().addLast(new LineEncoder(LineSeparator.DEFAULT, CharsetUtil.UTF_8));
        ch.pipeline().addLast(new TcpClientHandler());
    }
```
接收方加上解码器：
```java
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        //解码器需要设置数据的最大长度，我这里设置成1024
        ch.pipeline().addLast(new LineBasedFrameDecoder(1024));
        //给pipeline管道设置业务处理器
        ch.pipeline().addLast(new TcpServerHandler());
    }
```
然后在发送方，发送消息时在末尾加上标识符：
```java
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        for (int i = 1; i <= 5; i++) {
            //在末尾加上默认的标识符\n
            ByteBuf byteBuf = Unpooled.copiedBuffer("msg No" + i + StringUtil.LINE_FEED, Charset.forName("utf-8"));
            ctx.writeAndFlush(byteBuf);
        }
    }
```
于是我们再次启动服务端和客户端，在服务端的控制台可以看到：

![](https://user-gold-cdn.xitu.io/2020/7/5/1731f4a496a5469d?w=505&h=231&f=png&s=42918)
粘包、拆包的问题就轻松得到解决。

注意点：**数据末尾一定是分隔符，分隔符后面不要再加上数据**，否则会当做下一条数据的开始部分。下面是错误演示：
```java
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        for (int i = 1; i <= 5; i++) {
            //在分隔符后面加上一段字符串
            ByteBuf byteBuf = Unpooled.copiedBuffer("msg No" + i + StringUtil.LINE_FEED + "[我是分隔符后面的字符串]", Charset.forName("utf-8"));
            ctx.writeAndFlush(byteBuf);
        }
    }
```
服务端的控制台就会看到这样的打印信息：

![](https://user-gold-cdn.xitu.io/2020/7/5/1731f7fca99184e9?w=446&h=208&f=png&s=24270)

## 2.2 使用自定义长度帧解码器
使用这个解码器解决粘包问题的原理是上面讲的第二种，在数据的头部声明数据的长度，按长度获取数据。这个解码器构造器需要定义5个参数，相对较为复杂一点，先看参数的解释：
- maxFrameLength  发送数据包的最大长度
- lengthFieldOffset  长度域的偏移量。长度域位于整个数据包字节数组中的开始下标。
- lengthFieldLength  长度域的字节数长度。长度域的字节数长度。
- lengthAdjustment  长度域的偏移量矫正。如果长度域的值，除了包含有效数据域的长度外，还包含了其他域（如长度域自身）长度，那么，就需要进行矫正。矫正的值为：包长 - 长度域的值 – 长度域偏移 – 长度域长。
- initialBytesToStrip  丢弃的起始字节数。丢弃处于此索引值前面的字节。

前面三个参数比较简单，可以用下面这张图进行演示：

![](https://user-gold-cdn.xitu.io/2020/7/6/173249f1dcce734e?w=557&h=342&f=png&s=28445)
矫正偏移量是什么意思呢？意思是假设你的长度域设置的值除了包括有效数据的长度还有其他域的长度包含在里面，那么就要设置这个值进行矫正，否则解码器拿不到有效数据。矫正值的公式就是上面写着了。

丢弃的起始字节数。这个比较简单，就是在这个索引值前面的数据都丢弃，只要后面的数据。一般都是丢弃长度域的数据。当然如果你希望得到全部数据，那就设置为0。

下面就在消息接收端使用自定义长度帧解码器，解决粘包的问题：
```java
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        //数据包最大长度是1024
        //长度域的起始索引是0
        //长度域的数据长度是4
        //矫正值为0，因为长度域只有 有效数据的长度的值
        //丢弃数据起始值是4，因为长度域长度为4，我要把长度域丢弃，才能得到有效数据
        ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4));
        ch.pipeline().addLast(new TcpClientHandler());
    }
```
接着编写发送端代码，根据解码器的设置，进行发送：
```java
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        for (int i = 1; i <= 5; i++) {
            String str = "msg No" + i;
            ByteBuf byteBuf = Unpooled.buffer(1024);
            byte[] bytes = str.getBytes(Charset.forName("utf-8"));
            //设置长度域的值，为有效数据的长度
            byteBuf.writeInt(bytes.length);
            //设置有效数据
            byteBuf.writeBytes(bytes);
            ctx.writeAndFlush(byteBuf);
        }
    }
```
然后启动服务端，客户端，我们可以看到控制台打印结果：

![](https://user-gold-cdn.xitu.io/2020/7/6/17324aa254af3f3f?w=343&h=199&f=png&s=15892)
可以看到，利用自定义长度帧解码器解决了粘包问题。

## 2.3 使用Google Protobuf编解码器
Netty[官网](https://netty.io/)上是明显写着支持Google Protobuf的，如图所示：

![](https://user-gold-cdn.xitu.io/2020/7/6/17322cc3e98146fe?w=599&h=343&f=png&s=56992)

### 2.3.1 Google Protobuf是什么

> 摘自官网的原话：
> Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data – think XML, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages.

翻译一下：Protocol buffers是Google公司的**与语言无关、平台无关、可扩展的序列化数据的机制**，类似XML，但是**更小、更快、更简单**。您只需**定义一次数据的结构化方式**，然后就可以使用**特殊生成的源代码**，轻松地**将结构化数据写入和读取到各种数据流中，并支持多种语言**。

[Google Protobuf官网](https://developers.google.cn/protocol-buffers/)

### 2.3.2 使用Google Protobuf
首先先下载编译器，我使用的是win系统，所以下载的是win版本。[下载编译器链接，版本是v3.6.1](https://github.com/protocolbuffers/protobuf/releases/tag/v3.6.1)

![](https://user-gold-cdn.xitu.io/2020/7/7/17324e5a84010099?w=1041&h=240&f=png&s=31916)

如果官网下载慢的话，我已经下载了一个，并且上传到百度网盘，[网盘链接](https://pan.baidu.com/s/11yckiP4uWXR9I0bKyRBzOQ)，提取码：8b1r。公众号什么的随缘关注吧，哈哈~

![](https://user-gold-cdn.xitu.io/2020/7/7/17329eb724fd70a4?w=741&h=364&f=png&s=19605)

以下步骤参考Google Protobuf的github项目的[指南](https://github.com/protocolbuffers/protobuf/tree/master/java)。

#### 第一步：添加maven依赖
```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.6.1</version>
</dependency>
```
#### 第二步：编写proto文件Message.proto

如何编写.proto文件的相关文档说明，可以去[官网查看](https://developers.google.cn/protocol-buffers/docs/proto3#scalar)

下面我写一个例子，请看示范：
```proto
syntax = "proto3"; //版本
option java_outer_classname = "MessagePojo";//生成的外部类名，同时也是文件名

message Message {
    int32 id = 1;//Message类的一个属性，属性名称是id，序号为1
    string content = 2;//Message类的一个属性，属性名称是content，序号为2
}
```
#### 第三步：使用编译器，通过.proto文件生成代码

解压前面下载下来的压缩包protoc-3.6.1-win32.zip,然后打开\protoc-3.6.1-win32\bin目录下，可以看到有一个protoc.exe程序。如图所示：

![](https://user-gold-cdn.xitu.io/2020/7/7/1732508b9c9007ee?w=635&h=108&f=png&s=6967)

然后复制前面写好的Message.proto文件到此目录下，如图所示：

![](https://user-gold-cdn.xitu.io/2020/7/7/173250997e5b223c?w=638&h=136&f=png&s=10706)

接着在此目录下打开命令行cmd，输入命令：protoc.exe --java_out=. Message.proto

![](https://user-gold-cdn.xitu.io/2020/7/7/173250bea472e4c7?w=521&h=107&f=png&s=3797)

然后就可以看到生成的MessagePojo.java文件。最后把文件复制到IDEA项目中。

![](https://user-gold-cdn.xitu.io/2020/7/7/173250d5605c8188?w=242&h=179&f=png&s=7294)

#### 第四步：在发送端添加编码器，在接收端添加解码器

客户端添加编码器，对消息进行编码。
```java
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        //在发送端添加Protobuf编码器
        ch.pipeline().addLast(new ProtobufEncoder());
        ch.pipeline().addLast(new TcpClientHandler());
    }
```
服务端添加解码器，对消息进行解码。
```java
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        //添加Protobuf解码器，构造器需要指定解码具体的对象实例
        ch.pipeline().addLast(new ProtobufDecoder(MessagePojo.Message.getDefaultInstance()));
        //给pipeline管道设置处理器
        ch.pipeline().addLast(new TcpServerHandler());
    }
```
#### 第五步：发送消息

客户端发送消息，代码如下：
```java
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        //使用的是构建者模式进行创建对象
        MessagePojo.Message message = MessagePojo
                .Message
                .newBuilder()
                .setId(1)
                .setContent("芜湖大司马，起飞~")
                .build();
        ctx.writeAndFlush(message);
    }
```
服务端接收到数据，并且打印：
```java
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, MessagePojo.Message messagePojo) throws Exception {
        System.out.println("id:" + messagePojo.getId());
        System.out.println("content:" + messagePojo.getContent());
    }
```
测试结果正确：
![](https://user-gold-cdn.xitu.io/2020/7/7/17325145051d6e3a?w=364&h=68&f=png&s=2580)
### 2.3.3 分析Protocol的粘包、拆包
实际上直接使用Protocol编解码器还是存在粘包问题的。

证明一下，发送端循环一百次发送100条"大司马，起飞"的消息，请看发送端代码演示：

```java
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        for (int i = 1; i <= 100; i++) {
            MessagePojo.Message message = MessagePojo
                    .Message
                    .newBuilder()
                    .setId(i)
                    .setContent(i + "号大司马，起飞~")
                    .build();
            ctx.writeAndFlush(message);
        }
    }
```
这时，启动服务端，客户端后，你会在控制台看到如下错误：
> com.google.protobuf.InvalidProtocolBufferException: While parsing a protocol message, the input ended unexpectedly in the middle of a field.  This could mean either that the input has been truncated or that an embedded message misreported its own length.

意思是：分析protocol消息时，输入意外地在字段中间结束。这可能意味着输入被截断，或者嵌入的消息误报了自己的长度。

其实就是粘包问题，多条数据合并成一条数据了，导致解析出现异常。

### 2.3.4 解决Protocol的粘包、拆包问题

只需要在发送端加上编码器**ProtobufVarint32LengthFieldPrepender**

```java
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());
        ch.pipeline().addLast(new ProtobufEncoder());
        ch.pipeline().addLast(new TcpClientHandler());
    }
```

接收方加上解码器**ProtobufVarint32FrameDecoder**
```java
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());
        ch.pipeline().addLast(new ProtobufDecoder(MessagePojo.Message.getDefaultInstance()));
        //给pipeline管道设置处理器
        ch.pipeline().addLast(new TcpServerHandler());
    }
```
然后再启动服务端和客户端，我们可以看到**马老师成功地起飞了**~~~

![](https://user-gold-cdn.xitu.io/2020/7/7/17329b2d05572fb7?w=261&h=284&f=png&s=15543)

ProtobufVarint32LengthFieldPrepender编码器的工作如下：

```java
 * BEFORE ENCODE (300 bytes)       AFTER ENCODE (302 bytes)
 * +---------------+               +--------+---------------+
 * | Protobuf Data |-------------->| Length | Protobuf Data |
 * |  (300 bytes)  |               | 0xAC02 |  (300 bytes)  |
 * +---------------+               +--------+---------------+
@Sharable
public class ProtobufVarint32LengthFieldPrepender extends MessageToByteEncoder<ByteBuf> {
    @Override
    protected void encode(ChannelHandlerContext ctx, ByteBuf msg, ByteBuf out) throws Exception {
        int bodyLen = msg.readableBytes();
        int headerLen = computeRawVarint32Size(bodyLen);
        //写入请求头，消息长度
        out.ensureWritable(headerLen + bodyLen);
        writeRawVarint32(out, bodyLen);
        //写入数据
        out.writeBytes(msg, msg.readerIndex(), bodyLen);
    }
}
```
ProtobufVarint32FrameDecoder解码器的工作如下：
```java
 * BEFORE DECODE (302 bytes)       AFTER DECODE (300 bytes)
 * +--------+---------------+      +---------------+
 * | Length | Protobuf Data |----->| Protobuf Data |
 * | 0xAC02 |  (300 bytes)  |      |  (300 bytes)  |
 * +--------+---------------+      +---------------+
public class ProtobufVarint32FrameDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        //标记读取的下标位置
        in.markReaderIndex();
        //获取读取的下标位置
        int preIndex = in.readerIndex();
        //解码，获取消息的长度,并且移动读取的下标位置
        int length = readRawVarint32(in);
        //比较解码前和解码后的下标位置，如果相等。表示字节数不够读取，跳到下一轮
        if (preIndex == in.readerIndex()) {
            return;
        }
        //如果消息的长度小于0，抛出异常
        if (length < 0) {
            throw new CorruptedFrameException("negative length: " + length);
        }
        //如果不够读取一个完整的数据，reset还原下标位置。
        if (in.readableBytes() < length) {
            in.resetReaderIndex();
        } else {
            //否则，把数据写入到out，接收端就拿到了完整的数据了
            out.add(in.readRetainedSlice(length));
        }
 }
```
总结一下：

发送端通过编码器在发送的时候在**消息体前面加上一个描述数据长度的数据块**。

接收方通过**解码器先获取描述数据长度的数据块**，知道完整数据的长度，**然后根据数据长度获取一条完整的数据**。

# 总结

**创作不易**，觉得有用就**关注一下**吧。

想第一时间看到我更新的文章，可以微信搜索公众号「`java技术爱好者`」，**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**
![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/6/30/17305cc08a7ed5d7?w=1180&h=528&f=png&s=152520)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！