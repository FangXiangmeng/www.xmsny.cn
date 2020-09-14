---
layout: post
title: "Centos7安装Openstack stein版本并集成Ceph RBD"
subtitle:   ""
description: ""
date: 2020-04-16T11:44:13+08:00
author: "FangXiangMeng"
tags:
  - Openstack
  - 虚拟化
---


# Centos7.4 安装Openstack Stein版本

## 一、环境准备
```
安装组件 节点IP 网卡名
controller  192.168.172.179 ens160、ens192
computer 192.168.172.180 ens160
```

## 二、环境配置
```
# 1、修改主机名
hostnamectl set-hostname controller

# 2、关闭防火墙
systemctl stop firewalld.service
systemctl disable firewalld.service

# 3、关闭SeLinux
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
grep -n 'SELINUX='  /etc/selinux/config

# 4、安装常用的软件包
yum install -y vim net-tools wget lrzsz tree screen lsof tcpdump nmap mlocate

# 5、修改host文件
cat >>/etc/hosts<<EOF
192.168.172.179 controller
192.168.172.180 computer
EOF
```
> controller&computer都需要执行。执行完毕上述代码后，关闭该虚拟机。


## 三、安装基础服务

1、启用OpenStack库
```shell
yum install -y centos-release-openstack-stein
```
2、安装时间同步服务
```shell
# 1、安装软件包
yum install -y chrony

# 2、允许其他节点可以连接到控制节点的 chrony 后台进程
echo 'allow 192.168.172.0/24' >> /etc/chrony.conf

# 3、启动 NTP 服务并将其配置为随系统启动
systemctl enable chronyd.service
systemctl start chronyd.service

# 4、设置时区
timedatectl set-timezone Asia/Shanghai

# 5、查询时间
timedatectl status

```

3、安装MariaDB
```shell
# 1、安装软件包
yum install -y mariadb mariadb-server MySQL-python

# 2、配置
vim /etc/my.cnf.d/mariadb-server.cnf #在mysqld模块下放入一下几行
default-storage-engine = innodb
innodb_file_per_table = on
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8

# 3、启动数据库服务，并将其配置为开机自启
systemctl start mariadb.service
systemctl enable mariadb.service

# 4、对数据库进行安全加固(设置root用户密码)
mysql_secure_installation
```

4、安装Memcache
```shell
# 1、安装软件包
yum install -y memcached python-memcached


# 2、修改监听ip
sed -i 's/127.0.0.1/0.0.0.0/' /etc/sysconfig/memcached

# 3、启动并加入开机自启
systemctl start memcached.service
systemctl enable memcached.service

#4、测试
printf "set foo 0 0 3\r\nbar\r\n"|nc controller 11211  # 添加数据
printf "get foo\r\n"|nc controller 11211  # 获取数据,在计算节点上也测试下
```

5、安装rabbitmq消息队列
```shell
# 1、安装
yum install -y rabbitmq-server

# 2、启动
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service

# 3、创建用户
rabbitmqctl add_user openstack openstack

# 4、授权
rabbitmqctl set_permissions openstack ".*" ".*" ".*"

# 5、启用web管理界面
rabbitmq-plugins list  # 查看rabbitmq有哪些插件
rabbitmq-plugins enable rabbitmq_management  # 启用web管理界面

# 6、浏览器上登录
# 在浏览器上输入http://10.0.0.11:15672/
# 用户名、密码均为：guest（第一次登录必须使用该用户密码）

# 7、在浏览器上为刚创建的openstack更新Tags为：administrator
# 点击Admin -> 点击Users列表中的openstack ->在Update this user中输入两次openstack作为密码(密码必须写，因此我们写原密码)，Tags设置为administrator -> 点击Update user
```

## 四、控制节点安装keystone
在控制器节点上安装和配置代号为Keystone的OpenStack身份服务。
为了实现可伸缩性，此配置部署了Fernet令牌和Apache HTTP服务器来处理请求。

### 4.1、安装前提
```shell
# 为keystone创建数据库并授权
-- 1、登录数据库管理系统
mysql -uroot -p

-- 2、创建数据库
create database keystone;

-- 3、创建用户并授权
grant all privileges on keystone.* to keystone_user@controller identified by 'keystone_pass';

-- 4、刷新权限
flush privileges;

-- 5、退出该session
quit;
```

### 4.2、安装及配置

1、安装软件包
```
yum install -y openstack-keystone httpd mod_wsgi
```

2、修改配置文件
```
# 1、备份原文件
sed -i.default -e '/^#/d' -e '/^$/d' /etc/keystone/keystone.conf

# 2、修改模块如下，vim /etc/keystone/keystone.conf
[database]
connection = mysql+pymysql://keystone_user:keystone_pass@controller/keystone

[token]
provider = fernet
```

3、同步数据库
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

4、初始化Fernet密钥存储库
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

5、创建keystone管理员
```
keystone-manage bootstrap --bootstrap-password admin_pass \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

### 4.3、配置并启动Apache HTTP server
```
# 1、配置ServerName
sed -i '/#ServerName/aServerName controller:80' /etc/httpd/conf/httpd.conf

# 2、连接keystone配置文件
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

# 3、启动并加入开机自启动
systemctl start httpd.service
systemctl enable httpd.service

# 4、配置管理员账号环境变量
export OS_USERNAME=admin
export OS_PASSWORD=admin_pass
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

### 4.4、创建域、项目、用户和角色
1、创建域
```
# 创建一个域示例，不创建也可以，Keystone已存在一个域：default
# openstack domain create --description "An Example Domain" example
```

2、创建服务项目
```
# 供glance、placement、nova和neutron等组件使用
openstack project create --domain default --description "Service Project" service
```

3、创建常规（非管理员）任务应使用无特权的项目和用户
```
# 1、创建项目
openstack project create --domain default --description "Demo Project" myproject

# 2、创建用户
openstack user create --domain default --password myuser_pass myuser

# 3、创建角色
openstack role create myrole

# 4、把用户和角色添加到项目
openstack role add --project myproject --user myuser myrole
```

### 4.5、验证身份及密码
1、删除临时环境变量OS_AUTH_URL、OS_PASSWORD
```
unset OS_AUTH_URL OS_PASSWORD
```

2、验证admin,密码为：admin_pass
```
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
```

3、验证myuser，密码为：myuser_pass
```
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name myproject --os-username myuser token issue
```

### 4.6、创建客户端环境变量脚本
由于临时环境变量只存在本session，每次session断开或重新打开一个session临时变量都会失效，因此将这些环境变量写入脚本中，需要用到时执行下脚本即可。

1、创建脚本
```
# 1、进入家目录
cd ~

# 2、创建admin用户的OpenStack客户端环境变量脚本
cat >admin-openrc<<EOF
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin_pass
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF

# 3、创建myuser用户的OpenStack客户端环境变量脚本
cat >demo-openrc<<EOF
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=myuser_pass
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```

2、验证脚本
```
# 1、加载环境变量
cd ~
. admin-openrc

# 2、请求验证token
openstack token issue
```
能看到返回的token就说明安装成功，具体看https://docs.openstack.org/keystone/stein/install/keystone-openrc-rdo.html

## 五、控制节点安装Glance
### 5.1、安装前提
1、为Glance建库并授权
```
create database glance;
grant all privileges on glance.* to glance_user@controller identified by 'glance_pass';
flush privileges;
quit;
```

2、获取keystone管理员凭据
```
. admin-openrc
```

3、创建Glance服务凭证
```
# 1、 创建glance用户
openstack user create --domain default --password glance_pass glance

# 2、将glance用户加入到service项目并授予admin(管理员)角色
openstack role add --project service --user glance admin

# 3、创建glance服务实体
openstack service create --name glance --description "OpenStack Image" image
```

4、创建Glance服务API端点
```
# 1、创建共有Glance服务API端点
openstack endpoint create --region RegionOne image public http://controller:9292

# 2、创建私有Glance服务API端点
openstack endpoint create --region RegionOne image internal http://controller:9292

# 3、创建管理Glance服务API端点
openstack endpoint create --region RegionOne image admin http://controller:9292
```

### 5.2、安装及配置
1、安装软件包
```
yum install -y openstack-glance
```

2、修改glance-api.conf配置文件
```
# 1、备份原文件
sed -i.default -e '/^#/d' -e '/^$/d' /etc/glance/glance-api.conf

# 2、修改模板如下，vim /etc/glance/glance-api.conf
[database]
connection = mysql+pymysql://glance_user:glance_pass@controller/glance

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

[keystone_authtoken]
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance_pass

[paste_deploy]
flavor = keystone
```

3、修改glance-registry.conf配置文件
```
# 1、备份原文件
sed -i.default -e '/^#/d' -e '/^$/d' /etc/glance/glance-registry.conf

# 2、修改模块如下，vim /etc/glance/glance-registry.conf
[database]
connection = mysql+pymysql://glance_user:glance_pass@controller/glance

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance_pass

[paste_deploy]
flavor = keystone
```

4、同步数据
```
su -s /bin/sh -c "glance-manage db_sync" glance
```

### 5.3、启动并加入开启自启
```
systemctl start openstack-glance-api.service openstack-glance-registry.service
systemctl enable openstack-glance-api.service openstack-glance-registry.service
```

### 5.4、上传镜像
1、下载镜像
```
cd ~
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```

2、将刚下载的镜像上传到glance
```
# 1、获取keystone管理员凭据
cd ~
. admin-openrc

# 2、上传镜像
cd ~
openstack image create "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public

# 3、查看上传结果
openstack image list
```

## 六、控制节点安装Placement
Placement组件从n版引入，p版强制用户使用，该组件的主要作用是参与 nova-scheduler 选择目标主机的调度流程中，负责跟踪记录 Resource Provider 的 Inventory 和 Usage，并使用不同的 Resource Classes 来划分资源类型，使用不同的 Resource Traits 来标记资源特征。

### 6.1、安装前提
1、为Placement建库并授权
```
create database placement;
grant all privileges on placement.* to 'placement_user'@'controller' identified by 'placement_pass';
flush privileges;
quit;
```
2、获取Keystone管理员凭据
```
cd ~
. admin-openrc
```

3、创建Placement服务凭证
```
# 1、 创建placement用户,密码设置为：placement_pass
openstack user create --domain default --password placement_pass placement

# 2、将管理员角色添加都placement用户和service项目中
openstack role add --project service --user placement admin

# 3、创建placement服务实体
openstack service create --name placement --description "Placement API" placement
```

4、创建Placement服务API端点
```
openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
```

### 6.2、安装及配置
1、安装软件包
```
yum install -y openstack-placement-api
```

2、修改placement.conf配置文件
```
# 1、备份原文件
sed -i.default -e '/^#/d' -e '/^$/d' /etc/placement/placement.conf

# 2、修改模块如下，vim /etc/placement/placement.conf
[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = placement
password = placement_pass

[placement_database]
connection = mysql+pymysql://placement_user:placement_pass@controller/placement
```

3、同步数据库
```
su -s /bin/sh -c "placement-manage db sync" placement
```

4、允许其他组件访问Placement API
```
# 1、修改Apache HTTP server配置
cat >>/etc/httpd/conf.d/00-placement-api.conf<<EOF

<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
EOF

# 2、重启Apache HTTP server使之生效
systemctl restart httpd
```

### 6.3、检查Placement安装结果
```
placement-status upgrade check
```

## 七、控制节点安装Nova
### 7.1、安装前提
1.为Nova建库并授权
```
# 1、建库
create database nova_api;
create database nova;
create database nova_cell0;

# 2、授权
grant all privileges on nova_api.* to 'nova_user'@'controller' identified by 'nova_pass';
grant all privileges on nova.* to 'nova_user'@'controller' identified by 'nova_pass';
grant all privileges on nova_cell0.* to 'nova_user'@'controller' identified by 'nova_pass';

# 3、刷新权限
flush privileges;
```

2、获取Keystone管理员凭证
```
cd ~
. admin-openrc
```

3、创建Nova服务凭证
```
# 1、 创建nova用户
openstack user create --domain default --password nova_pass nova

# 2、将管理员角色添加都nova用户和service项目中
openstack role add --project service --user nova admin

# 3、创建nova服务实体
openstack service create --name nova --description "OpenStack Compute" compute
```

4、创建Nova服务API端点
```
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```

### 7.2、安装及配置
1、安装软件包
```
yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler
```

2、编辑nova.conf配置文件
```
# 1、备份原文件
sed -i.default -e '/^#/d' -e '/^$/d' /etc/nova/nova.conf

# 2、修改模块如下，vim /etc/nova/nova.conf
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:openstack@controller
my_ip = 192.168.172.179
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
rpc_backend=rabbit

[api]
auth_strategy = keystone

[api_database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

[glance]
api_servers = http://controller:9292

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = nova_pass

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement_pass

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
```

3、同步nova-api数据库
```
su -s /bin/sh -c "nova-manage api_db sync" nova
```

4、注册cell0数据库
```
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

5、创建cell1原件
```
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

6、同步nova数据库
```
su -s /bin/sh -c "nova-manage db sync" nova
```

7、验证novacell0和cell1注册情况
```
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

### 7.3、启动并加入开机自启
```
systemctl start openstack-nova-api.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

systemctl enable openstack-nova-api.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
```

## 八、准备一台计算节点虚拟机
### 8.1、安装基础服务
1、启用OpenStack库
```
yum install -y centos-release-openstack-stein
```
2、安装 OpenStack 客户端
```
yum install -y python-openstackclient
```
3、时间同步
```
# 1、安装软件包
yum install -y chrony

# 2、将时间同步服务器修改为controller节点
sed -i '/^server/d' /etc/chrony.conf
sed -i '2aserver controller iburst' /etc/chrony.conf

# 3、启动 NTP 服务并将其配置为随系统启动
systemctl enable chronyd.service
systemctl start chronyd.service

# 4、设置时区
timedatectl set-timezone Asia/Shanghai

# 5、查看时间同步源
chronyc sources

# 6、查看时间是否正确
timedatectl status
```

## 九、计算节点安装Nova
### 9.1、安装及配置
1、安装软件包
```
yum install -y openstack-nova-compute
```

2、检查是否支持虚拟化
```
egrep -c '(vmx|svm)' /proc/cpuinfo  # 结果大于等于1,支持
```

3、编辑nova.conf配置文件
```
# 1、备份原文件
sed -i.default -e '/^#/d' -e '/^$/d' /etc/nova/nova.conf

# 2、修改模块如下，vim /etc/nova/nova.conf
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:openstack@controller
my_ip = 192.168.172.180
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova_pass

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[libvirt]
virt_type = qemu
```

4、启动并加入开机自启
```
systemctl start libvirtd.service openstack-nova-compute.service
systemctl enable libvirtd.service openstack-nova-compute.service
```

### 9.2、在控制节点上添加计算节点
1、取得keystone管理员凭据
```
cd ~
. admin-openrc
```

2、添加计算节点到cell 数据库
```
openstack compute service list --service nova-compute
```

3、发现计算节点
```
# 手动发现
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

# 定期主动发现
# 1、修改/etc/nova/nova.conf配置文件
[scheduler]
discover_hosts_in_cells_interval=300

# 2、重启nova服务
systemctl restart openstack-nova-api.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
```

## 十、控制节点安装Neutron
### 10.1、安装前提
1、建库并授权
```
create database neutron;
grant all privileges on neutron.* to 'neutron_user'@'controller' identified by 'neutron_pass';
flush privileges;
quit;
```

2、获取Keystone管理员凭证
```
cd ~
. admin-openrc
```

3、创建Neutron服务凭证
```
openstack user create --domain default --password neutron_pass neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
```

4、创建Neutron服务API端点
```
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```

### 10.2、安装及配置
1、安装软件
```
yum install -y openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```

2、编辑neutron.conf配置文件
```
# 1、备份原文件并删除注释
sed -i.default -e '/^#/d' -e '/^$/d' /etc/neutron/neutron.conf

# 2、修改模块如下，vim /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[database]
connection = mysql+pymysql://neutron_user:neutron_pass@controller/neutron

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers =controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron_pass

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova_pass
```

3、配置模块化第2层（ML2）插件
```
# 1、备份原文件并删除注释
sed -i.default -e '/^#/d' -e '/^$/d' /etc/neutron/plugins/ml2/ml2_conf.ini

# 2、修改模块如下，vim /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 1:1000

[securitygroup]
enable_ipset = true
```

4、配置Linux桥代理
```
# 1、备份原文件并删除注释
sed -i.default -e '/^#/d' -e '/^$/d' /etc/neutron/plugins/ml2/linuxbridge_agent.ini

# 2、修改模块如下，vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
physical_interface_mappings = provider:eth0

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

5、配置DHCP代理
```
# 1、备份原文件并删除注释
sed -i.default -e '/^#/d' -e '/^$/d' /etc/neutron/dhcp_agent.ini

# 2、修改模块如下，vim /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

6、配置元数据代理
```
# 1、备份原文件并删除注释
sed -i.default -e '/^#/d' -e '/^$/d' /etc/neutron/metadata_agent.ini

# 2、修改模块如下，vim /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = metadata_secret
```

7、配置/etc/nova/nova.conf文件neutron模块
```
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron_pass
service_metadata_proxy = true
metadata_proxy_shared_secret = metadata_secret
```

8、创建网络服务初始化脚本需要的软连接
```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

9、同步数据库
```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

10、重启Compute API服务
```
systemctl restart openstack-nova-api.service
```

11、启动网络服务并开启自启
```
systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service \
  neutron-dhcp-agent.service \
  neutron-metadata-agent.service

systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service \
  neutron-dhcp-agent.service \
  neutron-metadata-agent.service
```

## 十一、计算节点安装Neutron
1、安装软件
```
yum install -y openstack-neutron-linuxbridge ebtables ipset
```

2、编辑neutron.conf配置文件
```
# 1、备份原文件并删除注释
sed  -i.default-e '/^#/d' -e '/^$/d' /etc/neutron/neutron.conf

# 2、修改模块如下，vim /etc/neutron/neutron.conf
[DEFAULT]
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers =controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron_pass

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

3、配置Linux桥代理
```
# 1、备份原文件并删除注释
sed -i.bak -e '/^#/d' -e '/^$/d' /etc/neutron/plugins/ml2/linuxbridge_agent.ini

# 2、修改模块如下，vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
physical_interface_mappings = provider:eth0

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

4、确保您的Linux操作系统内核支持网桥过滤器
```
# 1、添加配置
cat >>/etc/sysctl.conf<<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# 2、启用
modprobe br_netfilter

# 3、生效
 sysctl -p
```

5、编辑/etc/nova/nova.conf文件
```
# 1、备份原文件并删除注释
sed -i.default -e '/^#/d' -e '/^$/d'  /etc/nova/nova.conf

# 2、修改模块如下，vim /etc/nova/nova.conf
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron_pass
```

6、重新启动Nova Compute服务
```
systemctl restart openstack-nova-compute.service
```

7、启动Linux网桥代理并开机自启动
```
systemctl enable neutron-linuxbridge-agent.service
systemctl start neutron-linuxbridge-agent.service
```

8、验证(在控制节点上操作)
```
openstack extension list --network
openstack network agent list   # 注意：一共4个，其中两个是Linux bridge agent说明成功
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 14b7daf4-29bb-431f-bbf3-2688f095978b | Linux bridge agent | computer   | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 174e4857-63bb-4222-bbab-06d948fdb23a | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| 98c56b29-8c37-4181-b26e-af9333316be3 | Linux bridge agent | controller | None              | :-)   | UP    | neutron-linuxbridge-agent |
| eb049ecb-75d0-40d3-98d8-694d19a3aaaf | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
```

## 十二、创建一台虚拟机（控制节点）
> 注意： 以下步骤均在控制节点上操作

### 12.1、创建网络
1、获取keystone管理员凭证
```
cd ~
. admin-openrc
```

2、创建网络
```
openstack network create  --share --external \
  --provider-physical-network provider \
  --provider-network-type flat provider

openstack network list  # 查看
```

3、创建子网
```
openstack subnet create --network provider \
  --allocation-pool start=192.168.172.220,end=192.168.172.230 \
  --dns-nameserver 8.8.8.8 --gateway 192.168.172.1 \
  --subnet-range 192.168.172.0/24 provider-sub

openstack subnet list
```

### 12.2、创建主机规格
1、获取keystone管理员凭证
```
cd ~
. admin-openrc
```
2、创建主机规格
```
openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano
# openstack flavor create 创建主机
# --id 主机ID
# --vcpus cpu数量
# --ram 64（默认是MB，可以写成G）
# --disk 磁盘（默认单位是G）
```

### 12.3、创建一个实例
1、获取demo用户权限凭证
```
cd ~
. demo-openrc
```
2、生成秘钥对
```
ssh-keygen -q -N ""
```
3、将密钥放在openstack上
```
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```
4、验证密码是否创建成功
```
nova keypair-list
```
5、添加安全组规则
```
# 允许ICMP（ping
openstack security group rule create --proto icmp default

# 允许安全shell（SSH）访问
openstack security group rule create --proto tcp --dst-port 22 default
```
6、查看创建实例需要的相关信息
```
openstack flavor list
openstack image list
openstack network list
openstack security group list
openstack keypair list
```
7、创建并启动实例
```
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=9e07c3d5-9a9e-496c-90b6-ba294f8b0699  \
  --security-group default \
  --key-name mykey hello-instance


# –flavor: 类型名称
# –image: 镜像名称
# --nic： 指定网络ID，根据刚刚openstack network list查到的网络ID填写,不是子网哦
# –security-group：安全组名

```
8、查看实例状态
```
[root@controller ~]# openstack server list
+--------------------------------------+----------------+--------+---------------------+--------+---------+
| ID                                   | Name           | Status | Networks            | Image  | Flavor  |
+--------------------------------------+----------------+--------+---------------------+--------+---------+
| 0d94ce6d-ae08-4ace-a183-3ecd44ccba56 | hello-instance | ACTIVE | provider=10.0.0.138 | cirros | m1.nano |
+--------------------------------------+----------------+--------+---------------------+--------+---------+
```

## 12.4 登录实例
### 12.4.1、通过SSH登录
```
ping 10.0.0.138

ssh cirros@10.0.0.138
```


## 十三、Cinder安装
### 13.1、创建数据库
```
$ CREATE DATABASE cinder;
$ GRANT ALL PRIVILEGES ON cinder.* TO ‘cinder’@‘localhost’
IDENTIFIED BY ‘123123’;
$ GRANT ALL PRIVILEGES ON cinder.* TO ‘cinder’@’%’
IDENTIFIED BY ‘123123’;
$ exit
```

### 13.2、创建cinder用户
```
$ source /root/admin-openrc
$ openstack user create --domain default --password-prompt cinder
```

### 13.3、将admin角色添加到cinder用户
```
openstack role add --project service --user cinder admin
```

### 13.4、创建cinderv2和cinderv3服务实体
```
$ openstack service create --name cinderv2  --description "OpenStack Block Storage" volumev2
$ openstack service create --name cinderv3  --description "OpenStack Block Storage" volumev3
```
### 13.5、创建Block Storage服务API端点
```
$ openstack endpoint create --region RegionOne  volumev2 public http://controller:8776/v2/%\(project_id\)s
$ openstack endpoint create --region RegionOne  volumev2 internal http://controller:8776/v2/%\(project_id\)s
$ openstack endpoint create --region RegionOne  volumev2 admin http://controller:8776/v2/%\(project_id\)s
$ openstack endpoint create --region RegionOne  volumev3 public http://controller:8776/v3/%\(project_id\)s
$ openstack endpoint create --region RegionOne  volumev3 internal http://controller:8776/v3/%\(project_id\)s
$ openstack endpoint create --region RegionOne  volumev3 admin http://controller:8776/v3/%\(project_id\)s
```

### 13.6、安装Cinder软件包
```
$ yum install openstack-cinder -y
```

### 13.7、配置
```
$ cp /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
$ vi /etc/cinder/cinder.conf
[DEFAULT]
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone
my_ip = 192.168.172.179

[database]
connection = mysql+pymysql://cinder:123123@controller/cinder

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = 123123

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```
2、修改cinder配置
```
$ vi /etc/nova/nova.conf
[cinder]
os_region_name = RegionOne
```

### 13.8、初始化数据
```
$ su -s /bin/sh -c "cinder-manage db sync" cinder
```

### 13.9、启动服务
```
$ systemctl restart openstack-nova-api.service
$ systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
$ systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
```

### 14、dashboard想要显示的花重启下httpd
```
$ systemctl restart httpd
```


## 十四、openstack对接ceph
### 14.1 Openstack对接Ceph，有3大模块可以使用Ceph

**镜像**
- Openstack的Glance组件提供镜像服务，可以将Image直接存储在Ceph中。

**操作系统盘**
- Openstack的Nova组件提供计算服务，一个虚机的创建必然需要操作系统，也就少不了系统盘，系统盘可以使用Ceph来提供。

**非操作系统盘**
- Openstack的Cinder组件提供块存储服务，也就是我们物理机中的普通磁盘，也可以通过Ceph来提供。


以上3个组件从Openstack角度来说，是不同的3个模块，提供的服务也不同，但对于Ceph来说，都是一个Rbd Image，也就是一个块存储。
走的是同样的API，但是在一些属性和参数间存在一些差异。
具体操作
创建存储池
针对Openstack的3个不同服务，需要把存储资源池隔离开，也就是每个服务一个Pool：
```
1、创建volumes池，对应Cinder服务
$ ceph osd pool create volumes 128

2、创建images池，对应Glance服务
$ ceph osd pool create images 128

3、创建vms池，对应Nova服务
$ ceph osd pool create vms 128

4、创建backups池，对应Cinder-backup服务。但这个backup在同一Ceph集群中，意义不大，既然是做备份的话，就应该跨集群或者跨机房、跨区域来达到备份容灾的目的。（这里没有使用）
$ ceph osd pool create backups 128
```


### 14.2 安装Ceph相关包
1、在glance-api的主机上安装python-rbd包
```
$ yum install python-rbd
```

2、在nova-compute、cinder-volume、cinder-backup节点上安装ceph-common包
```
$ yum install ceph-common
```
安装完ceph包之后，需要将ceph集群的ceph.conf copy到所有client端。

> 如果在Ceph的配置中打开了auth认证，就需要做如下的操作；如果Ceph中的auth都是设置的none，也就是关闭的话，可以不做如下操作。

3、在ceph中创建了cinder、glance等用户，并做了权限控制
```
$ ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images'
$ ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
$ ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups'
```
将上面生成的keyring文件，保存在相应的节点上，并修改为相应的权限

4、把ceph节点*.keyring文件cp到controller节点和computer节点
```
$ ceph auth get-or-create client.glance | ssh controller sudo tee /etc/ceph/ceph.client.glance.keyring
ssh controller  sudo chown glance:glance /etc/ceph/ceph.client.glance.keyring

$ ceph auth get-or-create client.cinder | ssh controller  sudo tee /etc/ceph/ceph.client.cinder.keyring
ssh controller  sudo chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring

$ ceph auth get-or-create client.cinder-backup | ssh controller  sudo tee /etc/ceph/ceph.client.cinder-backup.keyring
ssh controller  sudo chown cinder:cinder /etc/ceph/ceph.client.cinder-backup.keyring

$ ceph auth get-or-create client.cinder | ssh controller sudo tee /etc/ceph/ceph.client.cinder.keyring

$ ceph$: scp /etc/ceph/* root@{controller,computer}:/etc/ceph
```

5、在libvirt上添加secret key(`computer`节点)
```
1.获取cinder keyring，并保存到一个临时文件中
$ ceph auth get-key client.cinder | ssh controller tee /etc/ceph/client.cinder.key

2.生成一个UUID
$ cd /etc/ceph
$ uuidgen
c4cd561d-5c25-44a4-8a9f-8b59659f1d1f

3.修改secret.xml文件，注意替换下面的uuid
$ cd /etc/ceph
$ cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>c4cd561d-5c25-44a4-8a9f-8b59659f1d1f</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF

$ virsh secret-define --file secret.xml
Secret c4cd561d-5c25-44a4-8a9f-8b59659f1d1f created

5.设置libvirt的secret key，并删除之前的key临时文件
$ sudo virsh secret-set-value --secret c4cd561d-5c25-44a4-8a9f-8b59659f1d1f --base64 $(cat client.cinder.key) && rm client.cinder.key secret.xml

6.将生成配置cp到controller节点
$ scp /etc/ceph/* root@controller:/etc/ceph
```

## 14.3 在nova、glance、cinder模块中集成ceph相关配置
### Glance配置
1、在/etc/glance/glance-api.conf中添加如下：
```
// 在glance_store域中增加如下，如果没有glance_store域，直接创建：
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

2、在控制节点（controller）重启glance-api服务：
```
$ systemctl restart openstack-glance-api.service
```

### Cinder配置
1、在/etc/cinder/cinder.conf中添加如下：
```
// 在DEFAULT域中增加：
[DEFAULT]
enabled_backends = ceph

// 在ceph域中增加如下，如果没有ceph域，直接创建：
[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
glance_api_version = 2
rbd_user = cinder
rbd_secret_uuid = c4cd561d-5c25-44a4-8a9f-8b59659f1d1f
```

2、在Cinder节点（controller）上，重启cinder-volume服务.
```
$ systemctl restart openstack-cinder-volume.service
```

### Nova配置(computer节点)
3、在Nova计算节点（compute）上，修改配置文件/etc/nova/nova.conf
```
[libvirt]
virt_type = qemu
inject_password=True
images_type = rbd
images_rbd_pool = vms
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = c4cd561d-5c25-44a4-8a9f-8b59659f1d1f
disk_cachemodes="network=writeback"
说明：此处uuid为4.4-配置Cinder集成Ceph中/etc/cinder/cinder.conf的uuid。
```
4、在Nova计算节点（compute）上，重启nova-compute服务：
```
$ systemctl restart openstack-nova-compute.service
```

## 十五、openstack集成ceph验证
### 15.1、验证glance集成ceph
1、在controller节点执行，创建新的镜像 CentOS6
```
$ openstack image create "CentOS6-ceph" --file centos-6.5-cloud.qcow2 --disk-format qcow2 --container-format bare --public
```
> 执行镜像上传，OpenStack本地环境中需要有.qcow2格式的镜像文件，例如本例中的 CentOS-7-aarch64-Custom.qcow2 镜像

2、在controller节点查看OpenStack中的可用镜像：
```
$ openstack image list
+--------------------------------------+---------------+--------+
| ID                                   | Name          | Status |
+--------------------------------------+---------------+--------+
| ebfa0cf8-8fdd-43bd-9c42-740852013784 | 初始化环境    | active |
| 18e3d6ec-4d75-4f20-8955-9d1f2365cb64 | CentOS-ceph   | active |
| 1597d670-043a-48ad-afc6-06ebf5a3f863 | CentOS6-ceph  | active |
| fcf0e3ca-8094-4727-acb4-23506720e279 | CentOS7-image | active |
| 64f97270-ac02-48e8-b1f6-3bf635320c01 | centos6.5     | active |
| a2875426-9356-477f-9411-88e8a0a9c1f5 | cirros        | active |
+--------------------------------------+---------------+--------+
```
3、查看ceph images存储池中的镜像：
```
$ rbd ls images
18e3d6ec-4d75-4f20-8955-9d1f2365cb64
```

### 15.2验证Cinder集成Ceph
1、在controller节点上执行，创建一个新的OpenStack 卷ceph_volume，大小为10G：
```
$ cinder create --name ceph_volume 10
```

2、在controller节点查看OpenStack以及ceph volumes存储池，发现卷创建成功并存储在ceph中：
```
[root@controller ~ ]# openstack volume list
+--------------------------------------+-------------+-----------+------+-------------+
| ID                                   | Name        | Status    | Size | Attached to |
+--------------------------------------+-------------+-----------+------+-------------+
| 9f334a48-6f47-40b0-8231-922a694c76cc | ceph_volume | available |   10 |             |
+--------------------------------------+-------------+-----------+------+-------------+

ceph节点执行
$ rbd ls volumes
volume-9f334a48-6f47-40b0-8231-922a694c76cc
```

### 15.3、验证Nova集成Ceph
1、在controller节点上执行，列出环境中可用的flavor以及image资源
```
$ openstack flavor list
+--------------------------------------+---------+------+------+-----------+-------+-----------+
| ID                                   | Name    |  RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+---------+------+------+-----------+-------+-----------+
| 0                                    | m1.nano |   64 |   10 |         0 |     1 | True      |
| ea80193f-3f79-4a51-a8e7-ab17f1fa5cf8 | centos7 | 4096 |   50 |        10 |     4 | True      |
+--------------------------------------+---------+------+------+-----------+-------+-----------+

$ openstack image list
+--------------------------------------+---------------+--------+
| ID                                   | Name          | Status |
+--------------------------------------+---------------+--------+
| ebfa0cf8-8fdd-43bd-9c42-740852013784 | 初始化环境    | active |
| 18e3d6ec-4d75-4f20-8955-9d1f2365cb64 | CentOS-ceph   | active |
| 1597d670-043a-48ad-afc6-06ebf5a3f863 | CentOS6-ceph  | active |
| fcf0e3ca-8094-4727-acb4-23506720e279 | CentOS7-image | active |
| 64f97270-ac02-48e8-b1f6-3bf635320c01 | centos6.5     | active |
| a2875426-9356-477f-9411-88e8a0a9c1f5 | cirros        | active |
+--------------------------------------+---------------+--------+
```

2、在controller节点上执行，使用现有资源创建一个虚拟机：
```
$ openstack server create --image 64f97270-ac02-48e8-b1f6-3bf635320c01 --flavor centos7 ceph-test
+-------------------------------------+--------------------------------------------------+
| Field                               | Value                                            |
+-------------------------------------+--------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                           |
| OS-EXT-AZ:availability_zone         |                                                  |
| OS-EXT-SRV-ATTR:host                | None                                             |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                             |
| OS-EXT-SRV-ATTR:instance_name       |                                                  |
| OS-EXT-STS:power_state              | NOSTATE                                          |
| OS-EXT-STS:task_state               | scheduling                                       |
| OS-EXT-STS:vm_state                 | building                                         |
| OS-SRV-USG:launched_at              | None                                             |
| OS-SRV-USG:terminated_at            | None                                             |
| accessIPv4                          |                                                  |
| accessIPv6                          |                                                  |
| addresses                           |                                                  |
| adminPass                           | rp4NX8J3hFke                                     |
| config_drive                        |                                                  |
| created                             | 2020-05-13T06:26:34Z                             |
| flavor                              | centos7 (ea80193f-3f79-4a51-a8e7-ab17f1fa5cf8)   |
| hostId                              |                                                  |
| id                                  | f4a61a04-37ed-4dad-bb32-5e00da1b40b9             |
| image                               | centos6.5 (64f97270-ac02-48e8-b1f6-3bf635320c01) |
| key_name                            | None                                             |
| name                                | ceph-test                                        |
| progress                            | 0                                                |
| project_id                          | 6c95ba186f1047e0a632c3cd1ab08168                 |
| properties                          |                                                  |
| security_groups                     | name='default'                                   |
| status                              | BUILD                                            |
| updated                             | 2020-05-13T06:26:34Z                             |
| user_id                             | f04d7001c5b64c4dac71135043dbff09                 |
| volumes_attached                    |                                                  |
+-------------------------------------+--------------------------------------------------+
```

3、在controller节点上执行，一段时间过后，查看虚拟状态，可以看到创建完成，在OpenStack环境和ceph查看虚拟机是否存在：
```
$ openstack server list
+--------------------------------------+-----------+--------+----------+-----------+---------+
| ID                                   | Name      | Status | Networks | Image     | Flavor  |
+--------------------------------------+-----------+--------+----------+-----------+---------+
| f4a61a04-37ed-4dad-bb32-5e00da1b40b9 | ceph-test | ACTIVE |          | centos6.5 | centos7 |
+--------------------------------------+-----------+--------+----------+-----------+---------+

ceph节点执行
$ rbd ls vms
f4a61a04-37ed-4dad-bb32-5e00da1b40b9_disk
f4a61a04-37ed-4dad-bb32-5e00da1b40b9_disk.eph0
f4a61a04-37ed-4dad-bb32-5e00da1b40b9_disk.swap
```



## 遇到问题：

1、创建实例失败。ERROR
![Openstack实例创建失败](D:/有道云存储/qqF92BCC0821E4E2EABBA59CC968555837/mdimage/openstack实例创建失败.png)

后台nova computer报错日志
![Openstack创建实例失败](D:/有道云存储/qqF92BCC0821E4E2EABBA59CC968555837/mdimage/openstack创建实例失败1.png)

#### 解决方法
![Openstack创建实例失败](D:/有道云存储/qqF92BCC0821E4E2EABBA59CC968555837/mdimage/openstack报错3.png)

在compute节点的`/etc/nova/nova.conf`中配置连接placement认证
```
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement_pass
```
> placement在S版本已经单独提出来做计算资源的管理。单独组件了


### 1.2、在keystone中无法找到默认角色"user"
问题描述：\
在dashboard创建项目的时候出现此报错。

解决方法:
```
openstack role create user
```
系统默认使用管理Role的角色 管理员用户：admin 普通用户：member（老版本） user（新版本）

### 1.3、openstack dashboard获取实例失败问题。创建失败问题。
问题描述：\
dashboard获取实例失败问题。创建失败,加载不出来实例列表。

解决方法：\
查看了maridb得链接数，发现默认只有100
```
MariaDB [(none)]> show global variables like '%max_conn%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| extra_max_connections | 1     |
| max_connect_errors    | 100   |
| max_connections       | 151   |
+-----------------------+-------+
```


1.mariadb数据库最大连接数，默认为151
```
MariaDB [(none)]> show variables like 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections |  151  |
+-----------------+-------+
```

2.配置/etc/my.cnf
```
[mysqld]下新添加一行如下参数：
max_connections=8000
```

3、由于mariadb有默认打开文件数限制
```
vi /usr/lib/systemd/system/mariadb.service
取消[Service]前的#号，
[Service]新添加两行如下参数：
LimitNOFILE=10000
LimitNPROC=10000
```

4.重新加载系统服务，并重启mariadb服务
```
systemctl --system daemon-reload
systemctl restart mariadb.service
再次查看mariadb数据库最大连接数，可以看到最大连接数已经是3000
MariaDB [(none)]> show variables like 'max_connections';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 8000  |
+-----------------+-------+
```


### 1.3 执行openstack volume service list返回503
报错

```
 openstack volume service list
The server is currently unavailable. Please try again at a later time.<br /><br />
The Keystone service is temporarily unavailable.

 (HTTP 503)
```

```
REQ: curl -g -i -X GET http://controller:8776/v2/6c95ba186f1047e0a632c3cd1ab08168/os-services -H "Accept: application/json" -H "User-Agent: python-cinderclient" -H "X-Auth-Token: {SHA256}dc4f470e39c8f3d8a2517483de6a5cb3f8fe5814b845624c645c55c8d66a6d9c"
Starting new HTTP connection (1): controller:8776
http://controller:8776 "GET /v2/6c95ba186f1047e0a632c3cd1ab08168/os-services HTTP/1.1" 503 218
RESP: [503] Connection: keep-alive Content-Length: 218 Content-Type: application/json Date: Wed, 13 May 2020 02:22:51 GMT X-Openstack-Request-Id: req-b378d292-7493-4e2f-93a0-fd64c395faff
RESP BODY: {"message": "The server is currently unavailable. Please try again at a later time.<br /><br />\nThe Keystone service is temporarily unavailable.\n\n", "code": "503 Service Unavailable", "title": "Service Unavailable"}
GET call to volumev2 for http://controller:8776/v2/6c95ba186f1047e0a632c3cd1ab08168/os-services used request id req-b378d292-7493-4e2f-93a0-fd64c395faff
The server is currently unavailable. Please try again at a later time.<br /><br />
The Keystone service is temporarily unavailable.
```

问题排查
- 1.一开始以为是keystone得问题，进行了keystone得日志排查后发现并非keystone得问题。
- 2.以为是httpd转发请求到keystone超时得问题。经过time测试发现时间只用得5s。默认超时时间15s。所以继续下一个排查
- 最后排查了cinder得日志又有了发现日志里面401.但是配置没问题。为什么出现这个问题呢? 上面debug看的是没有授权权限。后面开始根据用户排查。最终把用户删掉了重建后问题解决。应该是cinder用户得密码和配置文件中得不一致导次问题。

解决方法：
```
$ openstack user delete cinder
$ openstack user create --domain default --password-prompt cinder
$ openstack volume service list
+------------------+-----------------+------+---------+-------+----------------------------+
| Binary           | Host            | Zone | Status  | State | Updated At                 |
+------------------+-----------------+------+---------+-------+----------------------------+
| cinder-volume    | controller@ceph | nova | enabled | up    | 2020-05-13T03:08:07.000000 |
| cinder-scheduler | controller      | nova | enabled | up    | 2020-05-13T03:08:07.000000 |
+------------------+-----------------+------+---------+-------+----------------------------+
```

