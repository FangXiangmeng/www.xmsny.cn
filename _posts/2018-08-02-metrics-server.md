---
layout: post
title: 使用metric-server收集kubernetes资源使用指标
subtitle: ""
description: "从Kubernetes 1.8开始，资源使用指标（如容器CPU和内存使用率）可以通过Metrics API在Kubernetes中获取。"
date: 2018-08-02T11:25:13+08:00
author: "FangXiangMeng"
published: true
tags:
  - Kubernetes
categories: [ Docs ]
---



## 简介
从Kubernetes 1.8开始，资源使用指标（如容器CPU和内存使用率）可以通过Metrics API在Kubernetes中获取。这些指标可以直接被用户访问（例如通过使用kubectl top命令），或由集群中的控制器使用（例如，Horizo​​ntal Pod Autoscale可以使用这些指标作出决策)。


## 创建证书

Metrics API的URI是/apis/metrics.k8s.io/，扩展了Kubernetes的核心API，因此在往集群中部署metrics-server之前需要确认Kubernetes集群配置了Aggregation Layer(聚合层)。

1.创建聚合层API证书
如果想开启聚合层API，需要创建几个与聚合层API相关的证书。

安装cfssl
方式一：直接使用二进制源码包安装
```shell
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ chmod +x cfssl_linux-amd64
$ mv cfssl_linux-amd64 /usr/local/bin/cfssl

$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ chmod +x cfssljson_linux-amd64
$ mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
$ chmod +x cfssl-certinfo_linux-amd64
$ mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

$ export PATH=/usr/local/bin:$PATH
方式二：使用去命令安装

$ go get -u github.com/cloudflare/cfssl/cmd/...
$ ls $GOPATH/bin/cfssl*
cfssl cfssl-bundle cfssl-certinfo cfssljson cfssl-newkey cfssl-scan
在$GOPATH/bin目录下得到以cfssl开头的几个命令。
```
> 注意：以下文章中出现的cat的文件名如果不存在需要手工创建。


- 创建CA（证书颁发机构）
- 创建CA配置文件

```shell
$ mkdir /root/ssl
$ cd /root/ssl
$ cfssl print-defaults config > config.json
$ cfssl print-defaults csr > csr.json
```

1 根据config.json文件的格式创建如下的ca-config.json文件
2 过期时间设置成了 87600h
```shell
$ cat > aggregator-ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "aggregator": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

### 字段说明：

- profiles ：可以定义多个个人资料，分别指定不同的过期时间，使用场景等参数;后续在签名证书时使用某个个人资料。
- signing：表示该证书可用于签名其它证书;生成的aggregator-ca.pem证书中CA=TRUE。
- server auth ：表示客户端可以用该CA对服务器提供的证书进行验证。
- client auth ：表示Server可以用该CA对Client提供的证书进行验证。

1 创建CA证书签名请求

2 创建³³ aggregator-ca-csr.json文件，内容如下：
```shell
{
  "CN": "aggregator",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shanghai",
      "L": "Shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ],
    "ca": {
       "expiry": "87600h"
    }
}
```

### 字段说明：

- “CN”：Common Name，kube-apiserver从证书中提取该字段作为请求的用户名（用户名）;浏览器使用该字段验证网站是否合法。
- “O”： Organization，KUBE-API 服务器从证书中提取该字段作为请求用户所属的组（组）;

1 生成CA证书和私
```shell
$ cfssl gencert -initca aggregator-ca-csr.json | cfssljson -bare aggregator-ca
$ ls aggregator-ca*
aggregator-ca-config.json  aggregator-ca.csr  aggregator-ca-csr.json  aggregator-ca-key.pem  aggregator-ca.pem
```
1 创建kubernetes证书
2 创建聚合器证书签名请求文件aggregator-csr.json：
```yaml
{
    "CN": "aggregator",
    "hosts": [
      "127.0.0.1",
      "192.168.123.250",
      "192.168.123.248",
      "192.168.123.249",
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```
- 如果主机字段不为空则需要指定授权使用该证书的IP或域名列表，由于该证书后续被etd集群和kubernetes主集群使用，所以上面分别指定了etcd集群，kubernetes master集群的主机IP和kubernetes服务的服务IP（一般是kube-apiserver指定的service-cluster-ip-range网段的第一个IP，如10.254.0.1）。
> 以上物理节点的IP也可以更换为主机名。

1 生成aggregator证书和私
```
$ cfssl gencert -ca=aggregator-ca.pem -ca-key=aggregator-ca-key.pem -config=aggregator-ca-config.json -profile=aggregator aggregator-csr.json | cfssljson -bare aggregator
$ ls aggregator*
aggregator.csr  aggregator-csr.json  aggregator-key.pem  aggregator.pem
```
2 分发证书
将生成的证书和秘钥文件（后缀名为.pem）拷贝到Master节点的/etc/kubernetes/ssl目录下备用。
```shell
$ cp *.pem /etc/kubernetes/ssl
```

3 开启聚合层API
kube-apiserver 增加以下配置：
- --requestheader-client-ca-file=/etc/kubernetes/ssl/aggregator-ca.pem
- --requestheader-allowed-names=aggregator
- --requestheader-extra-headers-prefix=X-Remote-Extra-
- --requestheader-group-headers=X-Remote-Group
- --requestheader-username-headers=X-Remote-User
- --proxy-client-cert-file=/etc/kubernetes/ssl/aggregator.pem
- --proxy-client-key-file=/etc/kubernetes/ssl/aggregator-key.pem

>注意
创建前面证书的的CN字段的值必须状语从句：参数--requestheader-allowed-names指定的值aggregator相同。

重启kube-apiserver：
```
$ systemctl daemon-reload
$ systemctl restart kube-apiserver
```
> 如果kube-proxy没有在Master上面运行，apiserver还需要添加配置：
```shell
--enable-aggregator-routing=true
```




## 安装metrics-server
https://github.com/kubernetes-incubator/metrics-server/tree/master/deploy  下载相关yaml文件。

![修改配置内容](D:/有道云存储/qqF92BCC0821E4E2EABBA59CC968555837/mdimage/kube-state-server配置.png)

1 修改metrics-server-deployment.yaml
增加上面红框内容
- --kubelet-insecure-tls：跳过验证Kubelet CA证书。不建议用于生产用途，但在具有自签名Kubelet服务证书的测试群集中非常有用。
- --kubelet-preferred-address-types：连接到Kubelet时考虑不同Kubelet地址节点类型的顺序功能类似于API服务器上的同名标志。这块使用的是主机ip，默认是主机hostname。

2 创建yaml
```shell
kubectl create -f aggregated-metrics-reader.yaml 
kubectl create -f auth-delegator.yaml  
kubectl create -f auth-reader.yaml  
kubectl create -f metrics-apiservice.yaml  
kubectl create -f metrics-server-deployment.yaml  
kubectl create -f metrics-server-service.yaml  
kubectl create -f resource-reader.yaml
```
3 部署完成后使用下面的命令查看node、pod相关的指标：
```shell
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods"
```
然后通过api:8080/apis/metrics.k8s.io/v1beta1/去看资源使用
![修改配置内容](D:/有道云存储/qqF92BCC0821E4E2EABBA59CC968555837/mdimage/metrics-api.png)



## 遇到问题
1. 开始没用--kubelet-preferred-address-types，容器去解析主机名称解析不到。持续报错。
当时解决方法：（给容器添加hosts解析）
![修改配置内容](D:/有道云存储/qqF92BCC0821E4E2EABBA59CC968555837/mdimage/pod-hostnames.png)

也可以在kube-dns或者coredns里面添加，kube-dns添加没成功

2. kubectl top node不可用。
解决方法：
目前1.10一下版本需要依赖heapster。当时使用的1.9.3版本所以kubectl top不可用。


3. 403报错
报错如下：
```shell
E0117 00:55:33.544214       1 manager.go:102] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:ctc-caas-node01: unable to fetch metrics from Kubelet ctc-caas-node01 (192.168.154.27): request failed - "403 Forbidden", response: "Forbidden (user=system:anonymous, verb=get, resource=nodes, subresource=stats)", unable to fully scrape metrics from source kubelet_summary:ctc-caas-matsr01: unable to fetch metrics from Kubelet ctc-caas-matsr01 (192.168.154.26): request failed - "403 Forbidden", response: "Forbidden (user=system:anonymous, verb=get, resource=nodes, subresource=stats)", unable to fully scrape metrics from source kubelet_summary:ctc-caas-node02: unable to fetch metrics from Kubelet ctc-caas-node02 (192.168.154.28): request failed - "403 Forbidden", response: "Forbidden (user=system:anonymous, verb=get, resource=nodes, subresource=stats)"]
```
**解决方法：**\

**第一种解决方法：**
在启动kubelet的时候修改--authorization-mode = Webhook为--authorization-mode = AlwaysAllow，可以避免403，但不安全; 

**第二种解决方法：**
使用kubeconfig，在启动指标 - 服务器容器的时候添加如下命令（在YAML中添加）：
```
command：
- / metrics-server 
- --kubelet-insecure-tls 
- --kubeconfig=/key/kubeconfig
```
--kubeconfig=/key/kubeconfig使用指定的kubeconfig，确保容器内部/key/ kubeconfig里面为kubeconfig内容，可以采用挂在卷的方式




3 metrics-server获取不到数据 


报错信息:
```shell
E0117 00:55:33.544214       1 manager.go:102] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:ctc-caas-node01: unable to fetch metrics from Kubelet ctc-caas-node01 (192.168.154.27): request failed - "403 Forbidden", response: "Forbidden (user=system:anonymous, verb=get, resource=nodes, subresource=stats)", unable to fully scrape metrics from source kubelet_summary:ctc-caas-matsr01: unable to fetch metrics from Kubelet ctc-caas-matsr01 (192.168.154.26): request failed - "403 Forbidden", response: "Forbidden (user=system:anonymous, verb=get, resource=nodes, subresource=stats)", unable to fully scrape metrics from source kubelet_summary:ctc-caas-node02: unable to fetch metrics from Kubelet ctc-caas-node02 (192.168.154.28): request failed - "403 Forbidden", response: "Forbidden (user=system:anonymous, verb=get, resource=nodes, subresource=stats)"]
```
**解决方法**\
临时解决方法： 
```shell
匿名帐号绑定一个cluster-admin的权限
kubectl create clusterrolebinding system:anonymous   --clusterrole=cluster-admin   --user=system:anonymous
```