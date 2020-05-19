---
layout: post
title: "Zookeeper集群搭建"
subtitle:   ""
description: ""
date: 2019-11-20T11:44:13+08:00
author: "FangXiangMeng"
tags:
  - Zookeeper
---

#### 搭建zookeeper3.4.12集群
ZooKeeper 是 Apache 的一个顶级项目，为分布式应用提供高效、高可用的分布式协调服务，提供了诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知和分布式锁等分布式基础服务。\
由于 ZooKeeper 便捷的使用方式、卓越的性能和良好的稳定性，被广泛地应用于诸如 Hadoop、HBase、Kafka 和 Dubbo 等大型分布式系统中。
Zookeeper 有三种运行模式：单机模式、伪集群模式和集群模式。这里我们以集群模式为例进行介绍。\
集群模式最重要的一点是，只要集群中存在超过一半的机器能够正常工作，那么整个集群就能够正常对外服务。正是基于这个特性，要将 ZK 集群的节点数量要为奇数（2n+1：如 3、5、7 个节点）较为合适。

```
主机信息：
zookeeper1  192.168.95.85
zookeeper2  192.168.95.86
zookeeper3 192.168.95.87
```

#### 前提条件
- 环境具备JAVA环境


#### 安装
**1、下载官方zk安装包并解压**
```bash
$ wget https://archive.apache.org/dist/zookeeper/zookeeper-3.4.12/zookeeper-3.4.12.tar.gz
$ mkdir /zk
$ tar xvf zookeeper-3.4.12.tar.gz -d /zk
```

**2、创建zk数据目录和日志目录**
```bash
$ mkdir /opt/zookeeper/data
$ mkdir /opt/zookeeper/logs
```

**3、创建myid文件**\
在各服务器节点(zk1、zk2、zk3)的 dataDir 目录下创建名为 myid 的文件，在文件第一行写上对应的 Server id。这个id必须在集群环境中服务器标识中是唯一的，且大小在1～255之间。
```bash
zk1
$ cd /opt/zookeeper/data
$ echo "1" > myid
zk2
$ cd /opt/zookeeper/data
$ echo "2" > myid
zk3
$ cd /opt/zookeeper/data
$ echo "3" > myid
```

**4、修改配置文件** \
将 zookeeper-3.4.12/conf 目录下的 zoo_sample.cfg 文件拷贝一份，命名为 zoo.cfg
```bash
$ cd /zk/zookeeper-3.4.12/conf
$ cp zoo_sample.cfg zoo.cfg
```
修改zoo.cfg配置文件 (3台机器都需要，可以先配置好一台，然后通过 scp 等命令复制到其它服务器节点)
```conf
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/opt/zookeeper
clientPort=2182
server.1 = 192.168.95.85:2888:3888
server.2 = 192.168.95.86:2888:3888
server.3 = 192.168.95.87:2888:3888
```

**参数说明：**
- tickTime=2000
Zookeeper最小时间单元，单位为ms，默认值为3000。也就是Leader与Follower每隔tickTime时间就会发送一个心跳。


- initLimit=10
Leader服务器等待Follower启动并完成数据同步的时间，默认值10。
当已经超过10*tickTime后，Leader还没有收到Follower的返回信息,那么表明这个Follower连接或同步失败。


- syncLimit=5
Leader服务器和Follower之间进行心跳检测的最大延时时间，默认值5，最长不能超过5*tickTime


- dataDir=/home/{$user}/zookeeper/zookeeper-3.4.12/data
存放内存数据结构的snapshot，便于快速恢复，默认情况下，事务日志也会存储在这里。建议同时配置参数dataLogDir, 事务日志的写性能直接影响zk性能。


- dataLogDir=/home/{$user}/zookeeper/zookeeper-3.4.12/data
dataLogDir事务日志输出目录。为了达到性能最大化，一般建议把dataDir和dataLogDir分到不同的磁盘上，这样就可以充分利用磁盘顺序写的特性。


- autopurge.purgeInterval, autopurge.snapRetainCount
客户端在与zookeeper交互过程中会产生非常多的日志，而且zookeeper也会将内存中的数据作为snapshot保存下来，这些数据是不会被自动删除的，这样磁盘中这样的数据就会越来越多。不过可以通过这两个参数来设置，让zookeeper自动删除数据。
autopurge.purgeInterval：指定自动清理快照文件和事务日志文件的时间，单位为h，默认为0表示不自动清理，这个时候可以使用脚本zkCleanup.sh手动清理。如果不清理则磁盘空间占用越来越大。
autopurge.snapRetainCount：用于指定保留快照文件和事务日志文件的个数，默认为3。
不过如果你的集群是一个非常繁忙的集群，然后又碰上这个删除操作，可能会影响zookeeper集群的性能，所以一般会让这个过程在访问低谷的时候进行，但是遗憾的是zookeeper并没有设置在哪个时间点运行的设置，所以有的时候我们会禁用这个自动删除的功能，而在服务器上配置一个cron，然后在凌晨来干这件事。


- clientPort=2181
顾名思义，就是客户端连接zookeeper服务的端口。这是一个TCP port。

- server.id=IP/Host : port1 : port2
id：用来配置ZK集群中的各节点，并建议id的值和myid保持一致。
IP/Host: 服务器的 IP 或者是与 IP 地址做了映射的主机名
port1：Leader和Follower或Observer交换数据使用
port2：用于Leader选举
注意：如果是伪集群的配置方式，不同的 Zookeeper 实例通信端口号不能一样，所以要给它们分配不同的端口号。


- maxClientCnxns
对于一个客户端的连接数限制，默认是60，这在大部分时候是足够了。
但是在我们实际使用中发现，在测试环境经常超过这个数，经过调查发现有的团队将几十个应用全部部署到一台机器上，以方便测试，于是这个数字就超过了


- minSessionTimeout, maxSessionTimeout
一般，客户端连接zookeeper的时候，都会设置一个session timeout，如果超过这个时间client没有与zookeeper server有联系，则这个session会被设置为过期(如果这个session上有临时节点，则会被全部删除，这就是实现集群感知的基础，后面的文章会介绍这一点)。
但是这个时间不是客户端可以无限制设置的，服务器可以设置这两个参数来限制客户端设置的范围。

**5、启动并测试(3台执行下面命令)**
```bash
zk1
$ cd /zk/zookeeper-3.4.12/bin
$ ./zkServer.sh start
$ ZooKeeper JMX enabled by default
Using config: /zk/zookeeper-3.4.12/bin/../conf/zoo.cfg
Mode: follower

zk2
$ cd /zk/zookeeper-3.4.12/bin
$ ./zkServer.sh start
$ ZooKeeper JMX enabled by default
Using config: /zk/zookeeper-3.4.12/bin/../conf/zoo.cfg
Mode: follower

zk3
zk1
$ cd /zk/zookeeper-3.4.12/bin
$ ./zkServer.sh start
$ ZooKeeper JMX enabled by default
Using config: /zk/zookeeper-3.4.12/bin/../conf/zoo.cfg
Mode: leader
```

**6、zk添加到开机自启中**
```bash
第一种方法：
$ echo "/zk/zookeeper-3.4.12/bin/zk.Server.sh start" >> /etc/rc.local

第二种方法(推荐，目录位置根据实际位置更改)：
$ vim /usr/lib/systemd/system/zookeeper.service
[Unit]
Description=zookeeper.service
After=network.target

[Service]
Type=forking
Environment=ZOO_LOG_DIR=/zk/zookeeper-3.4.12/log
ExecStart=/zk/zookeeper-3.4.12/bin/zkServer.sh start
ExecStop=/zk/zookeeper-3.4.12/bin/zkServer.sh stop
ExecReload=/zk/zookeeper-3.4.12/bin/zkServer.sh restart
Restart=always

[Install]
WantedBy=multi-user.target

$ systemctl daemon-reload  //重载配置
$ systemctl enable zookeeper //加入开机自启
$ systemctl start zookeeper //启动zookeeper

```

**7、基本命令** \
登录ZookeeperClient：
```log
$ ./zkCli.sh -server 192.168.95.86:2182
Connecting to 192.168.95.86:2182
2019-08-15 13:17:38,395 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.12-e5259e437540f349646870ea94dc2658c4e44b3b, built on 03/27/2018 03:55 GMT
2019-08-15 13:17:38,406 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=node1
2019-08-15 13:17:38,407 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.7.0_51
2019-08-15 13:17:38,443 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
2019-08-15 13:17:38,443 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.51-2.4.5.5.el7.x86_64/jre
2019-08-15 13:17:38,444 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/zk/zookeeper-3.4.12/bin/../build/classes:/zk/zookeeper-3.4.12/bin/../build/lib/*.jar:/zk/zookeeper-3.4.12/bin/../lib/slf4j-log4j12-1.7.25.jar:/zk/zookeeper-3.4.12/bin/../lib/slf4j-api-1.7.25.jar:/zk/zookeeper-3.4.12/bin/../lib/netty-3.10.6.Final.jar:/zk/zookeeper-3.4.12/bin/../lib/log4j-1.2.17.jar:/zk/zookeeper-3.4.12/bin/../lib/jline-0.9.94.jar:/zk/zookeeper-3.4.12/bin/../lib/audience-annotations-0.5.0.jar:/zk/zookeeper-3.4.12/bin/../zookeeper-3.4.12.jar:/zk/zookeeper-3.4.12/bin/../src/java/lib/*.jar:/zk/zookeeper-3.4.12/bin/../conf:
2019-08-15 13:17:38,445 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2019-08-15 13:17:38,445 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp
2019-08-15 13:17:38,446 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
2019-08-15 13:17:38,446 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux
2019-08-15 13:17:38,447 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd64
2019-08-15 13:17:38,447 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=3.10.0-123.el7.x86_64
2019-08-15 13:17:38,448 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=root
2019-08-15 13:17:38,448 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/root
2019-08-15 13:17:38,449 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/zk/zookeeper-3.4.12/bin
2019-08-15 13:17:38,453 [myid:] - INFO  [main:ZooKeeper@441] - Initiating client connection, connectString=192.168.95.86:2182 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@43e47e37
Welcome to ZooKeeper!
2019-08-15 13:17:38,619 [myid:] - INFO  [main-SendThread(192.168.95.86:2182):ClientCnxn$SendThread@1028] - Opening socket connection to server 192.168.95.86/192.168.95.86:2182. Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
2019-08-15 13:17:38,669 [myid:] - INFO  [main-SendThread(192.168.95.86:2182):ClientCnxn$SendThread@878] - Socket connection established to 192.168.95.86/192.168.95.86:2182, initiating session
2019-08-15 13:17:38,707 [myid:] - INFO  [main-SendThread(192.168.95.86:2182):ClientCnxn$SendThread@1302] - Session establishment complete on server 192.168.95.86/192.168.95.86:2182, sessionid = 0x2000ee31b6f0002, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: 192.168.95.86:2182(CONNECTED) 0] help
ZooKeeper -server host:port cmd args
	connect host:port
	get path [watch]
	ls path [watch]
	set path data [version]
	rmr path
	delquota [-n|-b] path
	quit
	printwatches on|off
	create [-s] [-e] path data acl
	stat path [watch]
	close
	ls2 path [watch]
	history
	listquota path
	setAcl path acl
	getAcl path
	sync path
	redo cmdno
	addauth scheme auth
	delete path [version]
	setquota -n|-b val path
[zk: 192.168.95.86:2182(CONNECTED) 1]
```

**8、zookeeper各类日志简介**

zookeeper服务器会产生三类日志：事务日志、快照日志和log4j日志。

在zookeeper默认配置文件zoo.cfg（可以修改文件名）中有一个配置项dataDir，该配置项用于配置zookeeper快照日志和事务日志的存储地址。

在官方提供的默认参考配置文件zoo_sample.cfg中，只有dataDir配置项。其实在实际应用中，还可以为事务日志专门配置存储地址，配置项名称为dataLogDir，在zoo_sample.cfg中并未体现出来。在没有dataLogDir配置项的时候，zookeeper默认将事务日志文件和快照日志文件都存储在dataDir对应的目录下。

建议将事务日志（dataLogDir）与快照日志（dataLog）单独配置，因为当zookeeper集群进行频繁的数据读写操作是，会产生大量的事务日志信息，将两类日志分开存储会提高系统性能，而且，可以允许将两类日志存在在不同的存储介质上，减少磁盘压力。

dataDir:存在一个文件夹version-2，该文件夹中保存着数据快照日志文件。如：acceptedEpoch、currentEpoch
dataLogDir：存在一个文件夹version-2，该文件夹中保存着事物日志文件。如：log.*

log4j用于记录zookeeper集群服务器运行日志，该日志的配置地址在conf/目录下的log4j.properties文件中，该文件中有一个配置项为“zookeeper.log.dir=.”，表示log4j日志文件在与执行程序zkServer.sh在同一目录下。当执行zkServer.sh时，在该文件夹下会产生zookeeper.out日志文件。

**9、修改 Zookeeper 日志 zookeeper.out 输出路径**

如果不做修改，默认zookeeper的日志输出信息都打印到了zookeeper.out文件中，这样输出路径和大小没法控制，因为日志文件没有轮转。所以需要修改日志输出方式。具体操作如下：
```
1、修改 ZOOKEEPER_HOME/bin/zkEnv.sh
将：
ZOO_LOG_DIR="."
改为：
ZOO_LOG_DIR="/home/{$user}/zookeeper/logs"

2、再修改下ZOO_LOG4J_PROP，让日志不是输出到zookeeper.out，而是写入到日志文件
将：
ZOO_LOG4J_PROP="INFO,CONSOLE"
改为：
ZOO_LOG4J_PROP="INFO,ROLLINGFILE"


3、修改: ZOOKEEPER_HOME/conf/log4j.properties


如果按日志文件大小轮转，只需执行一步
zookeeper.root.logger=INFO,ROLLINGFILE


如果按照天轮转，需执行以下两步
1） log4j.appender.ROLLINGFILE=org.apache.log4j.DailyRollingFileAppender
2） zkEnv.sh：注释掉# log4j.appender.ROLLINGFILE.MaxFileSize=10MB

4、修改 /zk/zookeeper-3.4.12/binzkServer.sh（可选）
可不修改该文件，只是zk的启动脚本默认用 nohup 启动会生成一个zookeeper.out的空文件
若修改则执行以下两步：
1） 注释以下行:
#_ZOO_DAEMON_OUT="$ZOO_LOG_DIR/zookeeper.out"

# nohup "$JAVA" "-Dzookeeper.log.dir.........
# -cp "$CLASSPATH" $JVMFLAGS $ZOOMAIN.........
添加该行:
"$JAVA" "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" -cp
```


### 报错
###### 1、启动失败
如果本机没安装java，需要安装java.如果安装了java，需要在bin/zkServer.sh中指定java路径
```bash
JAVA_HOME=/usr/local/java
```
