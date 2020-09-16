---
layout: post
title: terraform管理vmware vsphere问题
subtitle:   ""
date: 2020-04-24T11:25:13+08:00
author: "FangXiangMeng"
published: true
tags:
  - terraform
---

### 克隆虚拟机要求
- 克隆模板必须是带有VMware Tools，且安装好的
- 克隆时不得打开虚拟机电源。
- 虚拟机上的所有磁盘都必须是SCSI磁盘。
- 您必须指定的disk设备数量至少与模板中存在的磁盘数量相同。这些设备按unit_number属性排序和排列。可以在此之后添加其他磁盘。
该size虚拟磁盘必须至少大小作为模板其对应的磁盘相同。
- 当使用linked_clone的size，thin_provisioned以及 eagerly_scrub为每个磁盘设置必须完全匹配于源模板中的单个磁盘的副本。
- scsi_controller_count应该根据需要配置该设置，以覆盖模板上的所有磁盘。为了获得最佳结果，请仅针对需要满足磁盘数量和带宽需求的控制器数量配置此设置，然后相应地配置模板。对于大多数工作负载，此设置应保持其默认值1，并且模板中的所有磁盘应驻留在单个主控制器上。
- 某些操作系统（例如Windows）对磁盘控制器类型的更改不能很好地响应，因此在使用此类操作系统时，请确保将 scsi_type其设置为与模板的控制器集完全匹配。为了获得最大的兼容性，请确保源模板上的SCSI控制器均为同一类型。

### 遇到问题
#### 1、创建vapp时报错
```bash
Error: POST https://192.168.182.62/rest/com/vmware/cis/tagging/tag-association?~action=list-attached-tags: 503 Service Unavailable
```

#### 2、克隆虚拟机失败，说是在克隆的时候磁盘类型更改了(实际没有更改，还是使用的thin)。
```bash
Error: error reconfiguring virtual machine: error processing disk changes post-clone: disk.0: ServerFaultCode: com.vmware.pbm.persistence.exception.PersistenceOperationException: Error occured while getting entry with key 'vm-278:2000' from provider '[Name:spbmAssociationProvider Optimistic locking:false]': RESOURCE (vm-278:2000), ACTION (queryAssociatedProfile): RESOURCE (vm-278), ACTION (PolicyIDByVirtualDisk)
```

**解决方法：**\
1.将vsphere provider版本从v1.17降到1.14.0解决此问题

#### 3、克隆虚拟机报错,选择系统版本有问题。不管选择那个都会报这个错。
```bash
Error: error sending customization spec: Customization of the guest operating system 'otherLinux64Guest' is not supported in this configuration. Microsoft Vista (TM) and Linux guests with Logical Volume Manager are supported only for recent ESX host and VMware Tools versions. Refer to vCenter documentation for supported configurations.
```
**解决方法：**\
1、安装VMware tools解决此问题。

参考连接
```bash
https://everythingshouldbevirtual.com/automation/terraform-vsphere-centos-7-customization-issues/
```

#### 4、克隆虚拟机报错
```bash
Error: Get unexpected status code: 503: call failed: Error response from vCloud Suite API: {"value":{"messages":[{"default_message":"????????¨?","id":"com.vmware.vapi.endpoint.cis.ServiceUnavailable","args":[]}]},"type":"com.vmware.vapi.std.errors.service_unavailable"}
```
> 这个时候实际上虚拟机在vcenter已经创建完成并且启动完成了。

解决方法：\
1、重启vcenter解决此问题。

参考链接
```bash
https://flings.vmware.com/vsphere-html5-web-client/bugs/197
```