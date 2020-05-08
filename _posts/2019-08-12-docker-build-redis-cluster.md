---
layout: post
title: Docker快速拉起redis-cluster
subtitle:   ""
description: "Docker拉起redis-cluster"
date: 2019-08-12T11:25:13+08:00
author: "FangXiangMeng"
published: true  
tags:
  - Docker
categories: [ Docs ]
---

使用Docker拉起redis-cluster

## docker拉起redis集群版

```
机器两台，集群6个。不同端口7000-7005。
192.168.193.65 192.167.193.66
```

前提条件
- 容器使用网络模式必须为hosts模式

1、redis1-6得配置文件.(bind ip更改为实际得主机ip)

**192.168.193.65**
```
$ mkdir /data/redis_data/{7001,7002,7003}
redis-7000.conf
port 7000
dir /data
pidfile /var/run/redis-7000.pid
dbfilename dump-7000.rdb
appendfilename "appendonly-7000.aof"
cluster-config-file nodes-7000.conf
cluster-enabled yes
cluster-node-timeout 5000
appendonly yes
#daemonize yes
masterauth jidian123
requirepass jidian123
bind 192.168.193.65

redis-7001.conf
port 7001
dir /data
pidfile /var/run/redis-7001.pid
dbfilename dump-7001.rdb
appendfilename "appendonly-7001.aof"
cluster-config-file nodes-7001.conf
cluster-enabled yes
cluster-node-timeout 5000
appendonly yes
masterauth jidian123
requirepass jidian123
bind 192.168.193.65

redis-7002.conf
port 7002
dir /data
pidfile /var/run/redis-7002.pid
dbfilename dump-7002.rdb
appendfilename "appendonly-7002.aof"
cluster-config-file nodes-7002.conf
cluster-enabled yes
cluster-node-timeout 5000
appendonly yes
masterauth jidian123
requirepass jidian123
bind 192.168.193.65
```

**192.167.193.66**
```
$ mkdir /data/redis_data/{7003,7004,7005}

redis-7003.conf
port 7003
dir /data
pidfile /var/run/redis-7003.pid
dbfilename dump-7003.rdb
appendfilename "appendonly-7003.aof"
cluster-config-file nodes-7003.conf
cluster-enabled yes
cluster-node-timeout 5000
appendonly yes
masterauth jidian123
requirepass jidian123
bind 192.168.193.66

redis-7004.conf
port 7004
dir /data
pidfile /var/run/redis-7004.pid
dbfilename dump-7004.rdb
appendfilename "appendonly-7004.aof"
cluster-config-file nodes-7004.conf
cluster-enabled yes
cluster-node-timeout 5000
appendonly yes
masterauth jidian123
requirepass jidian123
bind 192.168.193.66

redis-7005.conf
port 7005
dir /data
pidfile /var/run/redis-7005.pid
dbfilename dump-7005.rdb
appendfilename "appendonly-7005.aof"
cluster-config-file nodes-7005.conf
cluster-enabled yes
cluster-node-timeout 5000
appendonly yes
masterauth jidian123
requirepass jidian123
bind 192.168.193.66

```

3、拉起容器：
**192.168.193.65**
```
$ mkdir /home/redis_data/{7000,7001,7002}
$ docker run -d -p 7000:7000  --restart=always --net=host -v /home/redis_data/7000/redis-7000.conf:/data/redis-7000.conf --name redis-7000 redis:latest redis-server /data/redis-7000.conf
$ docker run -d -p 7001:7001 --restart=always --net=host -v /home/redis_data/7001/redis-7001.conf:/data/redis-7001.conf --name redis-7001 redis:latest redis-server /data/redis-7001.conf
$ docker run -d -p 7002:7002 --restart=always --net=host -v /home/redis_data/7002/redis-7002.conf:/data/redis-7002.conf --name redis-7002 redis:latest redis-server /data/redis-7002.conf
```
**192.168.193.66**
```
$ mkdir /home/redis_data/{7003,7004,7005}
$ docker run -d -p 7003:7003 --restart=always --net=host -v /home/redis_data/7003/redis-7003.conf:/data/redis-7003.conf --name redis-7003 redis:latest redis-server /data/redis-7003.conf
$ docker run -d -p 7004:7004 --restart=always --net=host -v /home/redis_data/7004/redis-7004.conf:/data/redis-7004.conf --name redis-7004 redis:latest redis-server /data/redis-7004.conf
$ docker run -d -p 7005:7005 --restart=always --net=host -v /home/redis_data/7005/redis-7005.conf:/data/redis-7005.conf --name redis-7005 redis:latest redis-server /data/redis-7005.conf
```

4、创建集群
```
$ redis-cli --cluster create -a jidian123 192.168.193.65:7000 192.168.193.65:7001 192.168.193.65:7002 192.168.193.66:7003 192.168.193.66:7004 192.168.193.66:7005 --cluster-replicas 1
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 2730
Master[1] -> Slots 2731 - 5460
Master[2] -> Slots 5461 - 8191
Master[3] -> Slots 8192 - 10922
Master[4] -> Slots 10923 - 13652
Master[5] -> Slots 13653 - 16383
M: 9c5443caba618210d5920f271b8a506bdcd894d2 192.168.193.65:7000
   slots:[0-2730] (2731 slots) master
M: aa2a8fc31730661dc437e7837654e1d521fcb33b 192.168.193.65:7001
   slots:[5461-8191] (2731 slots) master
M: acb686224dea0e120ad347408014112799c10c83 192.168.193.65:7002
   slots:[10923-13652] (2730 slots) master
M: 5053882fac63272eefbb42df44011efc5f18b2d8 192.168.193.66:7003
   slots:[2731-5460] (2730 slots) master
M: 02b123989e462f491ebde8aed46191f375c4e7f8 192.168.193.66:7004
   slots:[8192-10922] (2731 slots) master
M: 96b36384720ac31196a288895f3ab3c1792ba538 192.168.193.66:7005
   slots:[13653-16383] (2731 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 192.168.193.65:7000)
M: 9c5443caba618210d5920f271b8a506bdcd894d2 192.168.193.65:7000
   slots:[0-2730] (2731 slots) master
M: acb686224dea0e120ad347408014112799c10c83 192.168.193.65:7002
   slots:[10923-13652] (2730 slots) master
M: 96b36384720ac31196a288895f3ab3c1792ba538 192.168.193.66:7005
   slots:[13653-16383] (2731 slots) master
M: 5053882fac63272eefbb42df44011efc5f18b2d8 192.168.193.66:7003
   slots:[2731-5460] (2730 slots) master
M: aa2a8fc31730661dc437e7837654e1d521fcb33b 192.168.193.65:7001
   slots:[5461-8191] (2731 slots) master
M: 02b123989e462f491ebde8aed46191f375c4e7f8 192.168.193.66:7004
   slots:[8192-10922] (2731 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

```
