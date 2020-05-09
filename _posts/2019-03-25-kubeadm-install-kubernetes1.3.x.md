---
layout: post
title: Kubeadm 安装1.13.4集群
subtitle: ""
description: "kubeadm是一个kubernetes官方提供的快速安装和初始化拥有最佳实践.目前已经GA."
date: 2019-03-25T11:25:13+08:00
author: "FangXiangMeng"
published: true
tags:
  - Kubernetes
categories: [ Docs ]
---

`kubeadm`是一个kubernetes官方提供的快速安装和初始化拥有最佳实践.目前已经GA。HA方案还没GA。

## Centos7.3基于kubeadm安装kubernetes1.13.4

### 系统环境
- Centos7 内核版本3.10.0-514
- 2GB或者以上的RAM(否则将没有足够空间留给app).
- 2核以上CPU.
- 集群的机器之间必须能通过网络互相通信.
- SWAP必须被关闭，否则kubelet会出错！

### 安装前准备
**1.关闭selinux**

```shell
$ setenforce 0
$ sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

**2.添加netfilter模块** 

> CentOS 7上的iptables被绕过而导致流量路由不正确的问题，需要把net.bridge.bridge-nf-call-iptables在sysctl配置中设置为1

```
$ cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

$ sysctl -p /etc/sysctl.d/k8s.conf
```
确保br_netfilter在此步骤之前加载了模块。这可以通过运行来完成```lsmod | grep br_netfilter```。要加载它显式调用```modprobe br_netfilter```。

**3.安装docker**

```
$ sudo yum install docker-ce docker-ce-cli containerd.io
```

**4.配置docker cgroupdriver**

```shell
$ systemctl cat docker
# /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
[Service]
Type=notify
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd --exec-opt native.cgroupdriver=systemd \
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
[Service]
EnvironmentFile=-/run/flannel/docker
```

**5.启动docker**

```
$ systemctl enabel docker
$ systemctl start docker
```


**6.安装kubeadm、kubectl、kubelet**

```
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
$ yum install -y kubelet kubeadm kubectl cri-tools --disableexcludes=kubernetes

// 启动kubelet
$ systemctl enable kubelet && systemctl start kubelet
```

> kubelet现在每隔几秒重新启动一次，因为它在一个crashloop中等待kubeadm告诉它该怎么做。


### 使用kubeadm安装k8s 
> 这里指定了版本和pod的ip范围段，把ttl设置为永不过期。

```
$ kubeadm init --kubernetes-version=v1.13.4 --pod-network-cidr=172.80.0.0/16 --token-ttl=0
[init] Using Kubernetes version: v1.13.4
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kube-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.193.64]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kube-master localhost] and IPs [192.168.193.64 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kube-master localhost] and IPs [192.168.193.64 127.0.0.1 ::1]
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 31.004730 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kube-master" as an annotation
[mark-control-plane] Marking the node kube-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kube-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 5w1erh.y9g8pqqrydfdvjtd
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.193.64:6443 --token 5w1erh.y9g8pqqrydfdvjtd --discovery-token-ca-cert-hash sha256:4b817b3481b0e2476979559f8afdcd8f68c1f8e64dd9473a8e3e28bab90816dd
```


这个时候，我们还不能通过kubectl来控制集群，要让kubectl可用，我们需要做：
```
# 对于非root用户
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 对于root用户
$ export KUBECONFIG=/etc/kubernetes/admin.conf
# 也可以直接放到~/.bash_profile
$ echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
```

查看集群状态
```
$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"} 
```

### 安装网络插件
比较常见的network addon有： Calico, Canal, Flannel, Kube-router, Romana, Weave Net等。这里我们使用Flannel。

**1.下载flannel所需yaml**
```
[root@kube-master ~]# wget https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

**2.修改flanne网段** 

> flannel网段修改成kubeadm init --pod-network-cidr这个网段相同的

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "173.80.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

**3.创建kube-flannel**

```
[root@kube-master ~]# kubectl create -f kube-flannel.yml
```

默认pod不会在master上调度pod，如果想在master上调度pod执行以下指令
```
[root@kube-master ~]# kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 加入node节点

**1.启动kubelet**

```
[root@kube-node1~]# systemctl enable kubelet && systemctl start kubelet
```

**2.加入集群**
```
[root@kube-node1 ~]#   kubeadm join 192.168.193.64:6443 --token 5w1erh.y9g8pqqrydfdvjtd --discovery-token-ca-cert-hash sha256:4b817b3481b0e2476979559f8afdcd8f68c1f8e64dd9473a8e3e28bab90816dd
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "kube-node1" could not be reached
	[WARNING Hostname]: hostname "kube-node1": lookup kube-node1 on 8.8.8.8:53: no such host
[discovery] Trying to connect to API Server "192.168.193.64:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.193.64:6443"
[discovery] Requesting info from "https://192.168.193.64:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.193.64:6443"
[discovery] Successfully established connection with API Server "192.168.193.64:6443"
[join] Reading configuration from the cluster...
[join] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.13" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kube-node1" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```


**4.查看节点加入是否成功**

```
[root@kube-master ~]# kubectl get nodes  --show-labels
NAME          STATUS   ROLES    AGE   VERSION   LABELS
kube-master   Ready    master   11h   v1.13.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=kube-master,node-role.kubernetes.io/master=
kube-node1    Ready    <none>   10h   v1.13.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=kube-node1
kube-node2    Ready    <none>   10h   v1.13.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=kube-node2

```

```
[root@kube-master ~]# kubectl get pod --all-namespaces  -o wide
NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
kube-system   coredns-86c58d9df4-bwtzw              1/1     Running   0          11h   172.80.0.2       kube-master   <none>           <none>
kube-system   coredns-86c58d9df4-lkzcx              1/1     Running   0          11h   172.80.0.3       kube-master   <none>           <none>
kube-system   etcd-kube-master                      1/1     Running   0          11h   192.168.193.64   kube-master   <none>           <none>
kube-system   kube-apiserver-kube-master            1/1     Running   0          11h   192.168.193.64   kube-master   <none>           <none>
kube-system   kube-controller-manager-kube-master   1/1     Running   0          11h   192.168.193.64   kube-master   <none>           <none>
kube-system   kube-flannel-ds-amd64-kjkfc           1/1     Running   0          10h   192.168.193.66   kube-node2    <none>           <none>
kube-system   kube-flannel-ds-amd64-l8sxc           1/1     Running   0          10h   192.168.193.65   kube-node1    <none>           <none>
kube-system   kube-flannel-ds-amd64-wv485           1/1     Running   0          11h   192.168.193.64   kube-master   <none>           <none>
kube-system   kube-proxy-mlpdj                      1/1     Running   0          10h   192.168.193.66   kube-node2    <none>           <none>
kube-system   kube-proxy-mw84r                      1/1     Running   0          10h   192.168.193.65   kube-node1    <none>           <none>
kube-system   kube-proxy-v4vxp                      1/1     Running   0          11h   192.168.193.64   kube-master   <none>           <none>
kube-system   kube-scheduler-kube-master            1/1     Running   0          11h   192.168.193.64   kube-master   <none>           <none>

```

### 问题
**1.如果安装过程中出现以下情况代表在下载镜像，请科学下载镜像**

```
[init] Using Kubernetes version: v1.13.4
[preflight] Running pre-flight checks
	[WARNING Hostname]: hostname "kube-master" could not be reached
	[WARNING Hostname]: hostname "kube-master": lookup kube-master on 192.168.106.172:53: server misbehaving
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
```
查看需要下载镜像
```
kubeadm config images list
```

**2.kubelet报错**

```
F0326 10:29:36.382908    4942 server.go:261] failed to run Kubelet: failed to create kubelet: misconfiguration: kubelet cgroup driver: "cgroupfs" is different from docker cgroup driver: "systemd" 
```

**解决方法：**
默认使用cgroup,这里更改为systemd，和docker一致。

```
[root@kube-node1 ~]# vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS $KUBELET_CGROUP_ARGS
```

**3.kubelet cni报错**

```
7541 docker_sandbox.go:384] failed to read pod IP from plugin/docker: NetworkPlugin cni failed on the status hook for pod "istio-ingressga...
```

**解决方法:**

```
[root@kube-node1 ~]# vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS $KUBELET_CGROUP_ARGS $KUBELET_NETWORK_ARGS
```
