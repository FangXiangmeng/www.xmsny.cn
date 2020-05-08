---
layout: post
title: harbor启用charts
subtitle: ""
description: "在 v1.6 版本的 harbor 中新增加了 helm charts 的管理功能,这样就可以利用 harbor 同时管理镜像和 helm charts 了，在部署 kubernetes 相关应用时就比较方便，本次尝试用 harbor 来管理 helm charts"
date: 2018-10-09T11:25:13+08:00
author: "FangXiangMeng"
published: true
tags:
  - Kubernetes
categories: [ Docs ]
---


在 v1.6 版本的 harbor 中新增加了 helm charts 的管理功能,这样就可以利用 harbor 同时管理镜像和 helm charts 了，在部署 kubernetes 相关应用时就比较方便，本次尝试用 harbor 来管理 helm charts。

### 启用Harbor的chart repository 服务
默认新版 harbor 不会启用 chart repository service，如果需要管理 helm，我们需要在安装时添加额外的参数，例如：
```SHELL
$ cd /srv/harbor
$ ./install.sh --with-chartmuseum
```

### 安装成功后可以把harbor做为helm charts的仓库
```shell
[root@caas1 /]# helm repo add --username=admin --password=Harbor12345 myrepo http://192.168.1.103/chartrepo/helmcharts
"myrepo" has been added to your repositories
[root@caas1 /]# helm repo list
NAME  	URL                                                   
local 	http://127.0.0.1:8879/charts                          
stable	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
myrepo	http://192.168.1.103/chartrepo/helmcharts

上面的helmcharts实际上是harbor的项目名。
也可以直接chartrepo(chartrepo是固定的不是项目名)
```

### 上传charts到harbor
因为我们需要用 helm push命令上传，该命令是通过 helm plugin实现的，但是默认 helm 没有安装此插件，需要安装：
```shell
helm plugin install https://github.com/chartmuseum/helm-push //安装helm push插件

helm fetch stable/redis-ha  //下载redis-ha 包

helm push --username=admin --password=Harbor12345 redis-ha-2.0.1.tgz myrepo  //上传redis-ha tar包到harbor
```

### 从Harbor下载charts并安装
```
helm install myrepo/redis-ha //安装myrepo仓库的redis
```

### 遇到问题
1.上传chart到harbor的Repoistory仓库，使用helm search myrepo查询不到。
解决方法：
```
helm repo updata //更新仓库即可
```

2.添加harbor成为helm的Repoistory仓库失败
解决方法 \
因为harbor的仓库名称固定的是chartrepo这个。开始当成项目名了所以一直没有上传成功。