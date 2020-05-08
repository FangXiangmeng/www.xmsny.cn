---
layout: post
title: 更改Docker存储驱动为overlay2
subtitle:   ""
description: "OverlayFS是一个现代的联合文件系统，类似于AUFS，但速度更快，实现更简单。Docker为OverlayFS提供了两个存储驱动程序：原始版本overlay，更新版本更稳定overlay2"
date: 2019-03-04T11:25:13+08:00
author: "FangXiangMeng"
tags:
  - Docker
categories: [ Docs ]
---

OverlayFS是一个现代的联合文件系统，类似于AUFS，但速度更快，实现更简单。Docker为OverlayFS提供了两个存储驱动程序：原始版本overlay，更新版本更稳定overlay2
<!--more-->


> 注意：如果使用OverlayFS，请使用overlay2驱动程序而不是 overlay驱动程序，因为它在inode利用率方面更有效。要使用新驱动程序，需要使用版本4.0或更高版本的Linux内核，或使用版本3.10.0-514及更高版本的RHEL或CentOS。 \



## 先决条件
**如果满足以下先决条件，则支持OverlayFS:**  

- overlay2 Docker EE 17.06.02-ee5及更高版本支持该驱动程序，并推荐用于Docker CE。
- overlay 允许使用驱动程序但不建议使用Docker CE。
- Linux内核的4.0或更高版本，或使用内核版本3.10.0-514或更高版本的RHEL或CentOS。 

**支持以下后备文件系统:**  

- ext4 （仅限RHEL 7.1）
- xfs（RHEL 7.2及更高版本），但仅d_type=true启用。使用 ```xfs_info```验证ftype选项设置为1。要xfs正确格式化 文件系统，请使用该标志```-n ftype=1```。 

> 警告：现在在没有d_type支持的XFS上运行会导致Docker跳过尝试使用overlay或overlay2驱动程序。而默认使用devicemapper，现有安装将继续运行，但会产生错误。这是为了允许用户迁移他们的数据。在将来的版本中，这将是一个致命的错误，这将阻止Docker启动。 \


## 使用overlay2存储驱动程序配置Docker 
> 注：更改存储驱动程序会使本地系统上的现有容器和镜像无法使用。使用docker save保存你已经建立的任何图像或他们推到你的私有仓库。

1.停止docker
```shell
$ systemctl stop docker
```

2.将之前docker数据备份
```shell
$ cp -au /var/lib/docker /var/lib/docker.bk
```

3.编辑/etc/docker/daemon.json。如果它尚不存在，请创建它。
```json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```
> overlay2.override_kernel_check: 禁用对Linux内核4.0或更高版本的检查。

4.启动docker
```shell
$ systemctl start docker
```

5.验证是否更改过来
```json
$ docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 18.09.2
Storage Driver: overlay2
 Backing Filesystem: xfs
 Supports d_type: true
 Native Overlay Diff: false
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: e6b3f5632f50dbc4e9cb6288d911bf4f5e95b18e
runc version: 6635b4f0c6af3810594d2770f662f34ddc15b40d
init version: fec3683
Security Options:
 seccomp
  Profile: default
Kernel Version: 3.10.0-514.el7.x86_64
Operating System: CentOS Linux 7 (Core)
OSType: linux
Architecture: x86_64
CPUs: 2
Total Memory: 976.5MiB
Name: localhost.localdomain
ID: Q3C3:HY5D:PEBS:UASF:WRAF:TG4O:DDWW:FKVY:CNIS:O242:LWS3:ILDU
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Labels:
Experimental: false
Insecure Registries:
 127.0.0.0/8
Live Restore Enabled: false
Product License: Community Engine
```


## 遇到问题 \
1.在Centos7.2操作系统做升级时，默认的```ftype```没开启，需要重新格式化磁盘。
```shell
$ mkfs.xfs -n ftype=1 /path/to/your/device
$  xfs_info /
meta-data=/dev/mapper/cl-root    isize=512    agcount=4, agsize=3079680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=12318720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=6015, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

ftype为1时代表开启，0为关闭
```

2.如果ftype没开启的情况下启动Docker会失败
```shell
dockerd: Error starting daemon: error initializing graphdriver: overlay2: the backing xfs filesystem is formatted without d_type support, which leads to incorrect behavior. Reformat the filesystem with ftype=1 to enable d_type support. Backing filesystems without d_type support are not supported.
```
**解决方法:**

1.备份数据，格式化文件系统
```shell
$ mkfs.xfs -n ftype=1 /path/to/your/device
```

2.换回devicemapper存储驱动
