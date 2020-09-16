---
layout: post
title: terraform管理openstack
subtitle:   ""
date: 2020-04-015T11:25:13+08:00
author: "FangXiangMeng"
published: true
tags:
  - terraform
---

## terraform管理openstack
### 1、创建flavor
```
provider "openstack" {
  user_name   = "admin"
  tenant_name = "admin"
  password    = "admin_pass"
  auth_url    = "http://controller:5000/v3/auth/tokens"
  region      = "RegionOne"
}

resource "openstack_compute_flavor_v2" "test-flavor" {
  name  = "my-flavor"
  ram   = "8096"
  vcpus = "2"
  disk  = "20"
}
```

### 2、创建云主机
```
provider "openstack" {
  user_name   = "admin"
  tenant_name = "admin"
  password    = "admin_pass"
  auth_url    = "http://controller:5000/v3/auth/tokens"
  region      = "RegionOne"
}

#查看租户id
data "openstack_identity_project_v3" "project_1" {
  name = "admin"
}

#查看镜像id
data "openstack_images_image_v2" "centos" {
  name        = "centos6.5"
}

#创建实例类型
resource "openstack_compute_flavor_v2" "terraform" {
  name  = "terraform"
  ram   = "8096"
  vcpus = "2"
  disk  = "20"
  swap  = "1024"
  is_public = "true"
  rx_tx_factor = "1"
}

#创建网络
resource "openstack_networking_network_v2" "network_1" {
  name           = "sub-10"
  admin_state_up = "true"
  external = "true"
  shared = "true"
  tenant_id = data.openstack_identity_project_v3.project_1.id

  segments {
    network_type = "flat"
    physical_network = "provider"
  }
}

#创建子网
resource "openstack_networking_subnet_v2" "subnet_1" {
  name       = "sub-10"
  network_id = openstack_networking_network_v2.network_1.id
  cidr       = "10.0.0.0/24"
  ip_version = 4
  gateway_ip = "10.0.0.1"
  enable_dhcp = "true"
  depends_on = [openstack_networking_network_v2.network_1]

  allocation_pool {
    start = "10.0.0.100"
    end = "10.0.0.200"
  }

}

#创建安全组
resource "openstack_compute_secgroup_v2" "secgroup_1" {
  name        = "secgroup_1"
  description = "a security group"

  rule {
    from_port   = 22
    to_port     = 22
    ip_protocol = "tcp"
    cidr        = "0.0.0.0/0"
  }
}

#创建实例
resource "openstack_compute_instance_v2" "basic" {
  name            = "centos"
  image_id        = data.openstack_images_image_v2.centos.id
  flavor_id       = openstack_compute_flavor_v2.terraform.id
  depends_on      = [openstack_networking_network_v2.network_1,openstack_compute_flavor_v2.terraform,openstack_networking_subnet_v2.subnet_1]

  network {
    name =  "sub-10"
  }
}
```

### 遇到问题
#### 1.解决依赖关系。
```
depends_on引用整个资源，而不是具体的属性值这种。需要写成openstack_compute_instance_v2.basic而不是openstack_compute_instance_v2.basic.id
```

### 学习地址
```
https://www.terraform.io/docs/providers/openstack/ //terraform官网

阿里云terraform学习4部曲
https://yq.aliyun.com/articles/713099?spm=a2c4e.11153940.0.0.32e05f4fQ0BpSL
https://yq.aliyun.com/articles/721188?spm=a2c4e.11153940.0.0.2e7f1970wCQPeg
https://yq.aliyun.com/articles/726467?spm=a2c4e.11153940.0.0.7e0670228Mt4S8
https://yq.aliyun.com/articles/727057?spm=a2c4e.11153940.0.0.1cd9346fhZrqy3
```