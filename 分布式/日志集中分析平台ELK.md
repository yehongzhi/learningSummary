# 思维导图

![](https://static.lovebilibili.com/elk_swdt.png)

> **文章已收录Github精选，欢迎Star**：https://github.com/yehongzhi/learningSummary

# 概述

我们都知道，在生产环境中经常会遇到很多异常，报错信息，需要查看日志信息排查错误。现在的系统大多比较复杂，即使是一个服务背后也是一个集群的机器在运行，**如果逐台机器去查看日志显然是很费力的，也不现实**。

如果能把日志全部收集到一个平台，然后像百度，谷歌一样**通过关键字搜索出相关的日志**，岂不快哉。于是就有了**集中式日志系统**。ELK就是其中一款使用最多的开源产品。

# 一、什么是ELK

ELK其实是Elasticsearch，Logstash 和 Kibana三个产品的首字母缩写，这三款都是开源产品。

**ElasticSearch**(简称ES)，是一个实时的分布式搜索和分析引擎，它可以用于全文搜索，结构化搜索以及分析。

**Logstash**，是一个数据收集引擎，主要用于进行数据收集、解析，并将数据发送给ES。支持的数据源包括本地文件、ElasticSearch、MySQL、Kafka等等。

**Kibana**，为 Elasticsearch 提供了分析和 Web 可视化界面，并生成各种维度表格、图形。

![](https://static.lovebilibili.com/elk_2.png)

# 二、搭建ELK

环境依赖：CentOS7.5，JDK1.8，ElasticSearch7.9.3，Logstash 7.9.3，Kibana7.9.3。

## 2.1 安装ElasticSearch

首先，到[官网](https://www.elastic.co/cn/downloads/elasticsearch)下载安装包，然后使用`tar -zxvf`命令解压。

![](https://static.lovebilibili.com/elk_3.png)

找到config目录下的elasticsearch.yml文件，修改配置：

```yaml
cluster.name: es-application
node.name: node-1
#对所有IP开放
network.host: 0.0.0.0
#HTTP端口号
http.port: 9200
#elasticsearch数据文件存放目录
path.data: /usr/elasticsearch-7.9.3/data
#elasticsearch日志文件存放目录
path.logs: /usr/elasticsearch-7.9.3/logs
```

配置完之后，因为ElasticSearch使用非root用户启动，所以创建一个用户。

```java
# 创建用户
useradd yehongzhi
# 设置密码
passwd yehongzhi
# 赋予用户权限
chown -R yehongzhi:yehongzhi /usr/elasticsearch-7.9.3/
```

然后切换用户，启动：

```
# 切换用户
su yehongzhi
# 启动 -d表示后台启动
./bin/elasticsearch -d
```

使用命令`netstat -nltp`查看端口号：

![](https://static.lovebilibili.com/elk_4.png)

访问http://192.168.0.109:9200/可以看到如下信息，表示安装成功。

![](https://static.lovebilibili.com/elk_5.png)

## 2.2 安装Logstash

首先在官网下载安装压缩包，然后解压，找到/config目录下的logstash-sample.conf文件，修改配置：

```yaml
input {
  file{
    path => ['/usr/local/user/*.log']
    type => 'user_log'
    start_position => "beginning"
  }
}

output {
  elasticsearch {
    hosts => ["http://192.168.0.109:9200"]
    index => "user-%{+YYYY.MM.dd}"
  }
}
```

input表示输入源，output表示输出，还可以配置filter过滤，架构如下：

![](https://static.lovebilibili.com/elk_11.png)

配置完之后，要有数据源，也就是日志文件，准备一个user.jar应用程序，然后后台启动，并且输出到日志文件user.log中，命令如下：

```shell
nohup java -jar user.jar >/usr/local/user/user.log &
```

接着再后台启动Logstash，命令如下：

```shell
nohup ./bin/logstash -f /usr/logstash-7.9.3/config/logstash-sample.conf &
```

启动完之后，使用`jps`命令，可以看到两个进程在运行：

![](https://static.lovebilibili.com/elk_8.png)

## 2.3 安装Kibana

首先还是到[官网](https://www.elastic.co/cn/downloads/kibana)下载压缩包，然后解压，找到/config目录下的kibana.yml文件，修改配置：

```yaml
server.port: 5601
server.host: "192.168.0.111"
elasticsearch.hosts: ["http://192.168.0.109:9200"]
```

和elasticSearch一样，不能使用root用户启动，需要创建一个用户：

```
# 创建用户
useradd kibana
# 设置密码
passwd kibana
# 赋予用户权限
chown -R kibana:kibana /usr/kibana/
```

然后使用命令启动：

```
#切换用户
su kibana
#非后台启动，关闭shell窗口即退出
./bin/kibana
#后台启动
nohup ./bin/kibana &
```

启动后在浏览器打开http://192.168.0.111:5601，可以看到kibana的web交互界面：

![](https://static.lovebilibili.com/elk_6.png)

## 2.4 效果展示

全部启动成功后，整个过程应该是这样，我们看一下：

![](https://static.lovebilibili.com/elk_12.png)

浏览器打开http://192.168.0.111:5601，到管理界面，点击“Index Management”可以看到，有一个`user-2020.10.31`的索引。

![](https://static.lovebilibili.com/elk_10.png)

点击`Index Patterns`菜单栏，然后创建，命名为user-*。

![](https://static.lovebilibili.com/elk_9.png)

最后，就可以到Discover栏进行选择，选择user-*的Index Pattern，然后搜索关键字，就找到相关的日志了！

![](https://static.lovebilibili.com/elk_7.png)

# 三、改进优化

上面只是用到了核心的三个组件简单搭建的ELK，实际上是有缺陷的。如果Logstash需要添加插件，那就全部服务器的Logstash都要添加插件，扩展性差。所以就有了**FileBeat**，占用资源少，只负责采集日志，不做其他的事情，这样就轻量级，把Logstash抽出来，做一些滤处理之类的工作。

![](https://static.lovebilibili.com/elk_13.png)

FileBeat也是官方推荐用的日志采集器，首先下载Linux安装压缩包：

```
https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.9.3-linux-x86_64.tar.gz
```

下载完成后，解压。然后修改filebeat.yml配置文件：

```yaml
#输入源
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /usr/local/user/*.log
#输出，Logstash的服务器地址
output.logstash:
  hosts: ["192.168.0.110:5044"]
#输出，如果直接输出到ElasticSearch则填写这个
#output.elasticsearch:
  #hosts: ["localhost:9200"]
  #protocol: "https"
```

然后Logstash的配置文件logstash-sample.conf，也要改一下：

```
#输入源改成beats
input {
  beats {
    port => 5044
    codec => "json"
  }
}
```

然后启动FileBeat：

```shell
#后台启动命令
nohup ./filebeat -e -c filebeat.yml >/dev/null 2>&1 &
```

再启动Logstash：

```shell
#后台启动命令
nohup ./bin/logstash -f /usr/logstash-7.9.3/config/logstash-sample.conf &
```

怎么判断启动成功呢，看Logstash应用的/logs目录下的logstash-plain.log日志文件：

![](https://static.lovebilibili.com/elk_14.png)

# 写在最后

目前，很多互联网公司都是采用ELK来做日志集中式系统，原因很简单：**开源、插件多、易扩展、支持数据源多、社区活跃、开箱即用**等等。我见过有一个公司在上面的架构中还会加多一个Kafka的集群，主要是基于日志数据量比较大的考虑。但是呢，基本的三大组件ElasticSearch，Logstash，Kibana是不能少的。

希望这篇文章能帮助大家对ELK有一些初步的认识，感谢大家的阅读。

**觉得有用就点个赞吧，你的点赞是我创作的最大动力**~

**拒绝做一条咸鱼，我是一个努力让大家记住的程序员。我们下期再见！！！**

![](https://static.lovebilibili.com/dashacha.png)

> 能力有限，如果有什么错误或者不当之处，请大家批评指正，一起学习交流！