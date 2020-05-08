---
layout: post
title: Centos7搭建ELK
subtitle:   ""
description: "ELK 不是一款软件，而是 Elasticsearch、Logstash 和 Kibana 三种软件产品的首字母缩写。这三者都是开源软件，通常配合使用，而且又先后归于 Elastic.co 公司名下，所以被简称为 ELK Stack。"
date: 2018-07-28T11:25:13+09:00
author: "FangXiangMeng"
published: true
tags:
  - Linux
categories: [ Docs ]
---

ELK 不是一款软件，而是 Elasticsearch、Logstash 和 Kibana 三种软件产品的首字母缩写。这三者都是开源软件，通常配合使用，而且又先后归于 Elastic.co 公司名下，所以被简称为 ELK Stack。


# CentOS 7上安装Elasticsearch，Logstash和Kibana

## 使用Logstash和Kibana在CentOS 7上集中日志记录

> 集中日志记录在尝试识别服务器或应用程序的问题时非常有用，因为它允许您在单个位置搜索所有日志。它也很有用，因为它允许您通过在特定时间范围内关联其日志来识别跨多个服务器的问题。本系列教程将教您如何在CentOS上安装Logstash和Kibana，然后如何添加更多过滤器来构造您的日志数据。
http://www.ibm.com/developerworks/cn/opensource/os-cn-elk/

## 安装介绍
> 在本教程中，我们将在CentOS 7上安装Elasticsearch ELK Stack，即Elasticsearch 5. x，Logstash 5. x和Kibana 5. x。我们还将向您展示如何配置它，以使用Filebeat 1.在一个集中的位置收集和可视化您的系统的系统日志。 Logstash是一个用于收集，解析和存储日志以供将来使用的开源工具。 Kibana是一个Web界面，可用于搜索和查看Logstash索引的日志。这两个工具都基于Elasticsearch，用于存储日志。

## 实验目的
> 本教程的目标是设置Logstash以收集多个服务器的syslog，并设置Kibana以可视化收集的日志。


**ELK堆栈设置有四个主要组件：**
- Logstash：处理传入日志的Logstash的服务器组件
- Elasticsearch：存储所有日志
- Kibana：用于搜索和可视化日志的Web界面，将通过Nginx
- Filebeat代理：安装在将其日志发送到Logstash的客户端服务器，Filebeat充当日志传送代理，利用伐木工具网络协议与Logstash进行通信
![](img/elk架构.png)

> 我们将在单个服务器上安装前三个组件，我们将其称为我们的ELK服务器。 Filebeat将安装在我们要收集日志的所有客户端服务器上，我们将统称为客户端服务器。

## 先决条件
> 您的ELK服务器将需要的CPU，RAM和存储量取决于您要收集的日志的卷。在本教程中，我们将使用具有以下规格的VPS用于我们的ELK服务器：
- OS: CentOS 7
- RAM: 4GB
- CPU: 2

> 注：根据自己的服务器资源分配各个节点的资源

## 安装 Java 8
> Elasticsearch和Logstash需要Java，所以我们现在就安装它。我们将安装最新版本的Oracle Java 8，因为这是Elasticsearch推荐的版本。
注：建议本地下载完最新版的JDK，然后上传到服务器的/usr/local/src目录



> 1. JDK下载地址：
> 2. http://www.oracle.com/technetwork/java/javase/downloads

然后使用此yum命令安装RPM（如果您下载了不同的版本，请在此处替换文件名）：
```shell
yum -y localinstall jdk-8u111-linux-x64.rpm \
or
rpm -ivh jdk-8u111-linux-x64.rpm
```

> 现在Java应该安装在/usr/java/jdk1.8.0_111/jre/bin/java，并从/usr/bin/java 链接。

## 安装 Elasticsearch
 Elasticsearch可以通过添加Elastic的软件包仓库与软件包管理器一起安装。
运行以下命令将Elasticsearch公共GPG密钥导入rpm：https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html 
```shell
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
在基于RedHat的发行版的/etc/yum.repos.d/目录中创建一个名为elasticsearch.repo的文件,其中包括：
```shell
echo '[elasticsearch-5.x]
name=Elasticsearch repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
| sudo tee /etc/yum.repos.d/elasticsearch.repo
```

Elasticsearch 源创建完成之后，通过makecache查看源是否可用，然后通过yum安装Elasticsearch ：
```shell
yum makecache
yum install elasticsearch -y
```
要将Elasticsearch配置为在系统引导时自动启动，请运行以下命令：
```shell
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable elasticsearch.service
```

Elasticsearch可以按如下方式启动和停止：

```shell 
sudo systemctl start elasticsearch.service
sudo systemctl stop elasticsearch.service
```

这些命令不会提供有关Elasticsearch是否已成功启动的反馈。相反，此信息将写入位于/ var / log / elasticsearch /中的日志文件中。 
默认情况下，Elasticsearch服务不会记录systemd日志中的信息。要启用journalctl日志记录，必须从elasticsearch中的ExecStart命令行中删除–quiet选项。服务文件。
> 注释24行的 --quiet \
vim /etc/systemd/system/multi-user.target.wants/elasticsearch.service

当启用systemd日志记录时，使用journalctl命令可以获得日志记录信息：
- 使用tail查看journal： \
```shell
  sudo journalctl -f 
```
- 要列出elasticsearch服务的日记帐分录：\
```shell
sudo journalctl --unit elasticsearch
```
- 要从给定时间开始列出elasticsearch服务的日记帐分录：
```shell
sudo journalctl --unit elasticsearch --since  "2017-1-4 10:17:16" \
since 表示指定时间之前的记录
```

> 使用man journalctl 查看journalctl 更多使用方法

## 检查Elasticsearch是否正在运行
> 您可以通过向localhost上的端口9200发送HTTP请求来测试Elasticsearch节点是否正在运行：
```shell
curl -XGET 'localhost:9200/?pretty'
```
我们能得到下面这样的回显：
```shell
{
  "name" : "De-LRNO",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "DeJzplWhQQK5uGitXr8jjA",
  "version" : {
    "number" : "5.1.1",
    "build_hash" : "5395e21",
    "build_date" : "2016-12-06T12:36:15.409Z",
    "build_snapshot" : false,
    "lucene_version" : "6.3.0"
  },
  "tagline" : "You Know, for Search"
}
```

## 配置 Elasticsearch
> Elasticsearch 从默认的/etc/elasticsearch/elasticsearch.yml加载配置文件， 
配置文件的格式考： 
https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html
```shell
[root@linuxprobe ~]# egrep -v "^#|^$" /etc/elasticsearch/elasticsearch.yml 
[root@linuxprobe ~]# egrep -v "^#|^$" /etc/elasticsearch/elasticsearch.yml
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 10.1.1.53  # 默认localhost，自定义为ip
http.port: 9200
```
> RPM还具有系统配置文件（/etc/sysconfig/elasticsearch），允许您设置以下参数：
```shell
[root@linuxprobe elasticsearch]# egrep -v "^#|^$" /etc/sysconfig/elasticsearch 
ES_HOME=/usr/share/elasticsearch
JAVA_HOME=/usr/java/jdk1.8.0_111
CONF_DIR=/etc/elasticsearch
DATA_DIR=/var/lib/elasticsearch
LOG_DIR=/var/log/elasticsearch
PID_DIR=/var/run/elasticsearch
```

## 日志配置
> Elasticsearch使用Log4j 2进行日志记录。 Log4j 2可以使用log4j2配置。属性文件。 Elasticsearch公开单个属性$ {sys：es。日志}，可以在配置文件中引用以确定日志文件的位置;这将在运行时解析为Elasticsearch日志文件的前缀。
例如，如果您的日志目录是/var/log/elasticsearch并且您的集群名为production，那么$ {sys：es。 logs}将解析为/var/log/elasticsearch/production。
默认日志配置存在：/etc/elasticsearch/log4j2.properties


## 安装 Kibana
> Kibana的RPM可以从ELK官网或从RPM存储库下载。它可用于在任何基于RPM的系统（如OpenSuSE，SLES，Centos，Red Hat和Oracle Enterprise）上安装Kibana。

## 导入Elastic PGP Key
> 我们使用弹性签名密钥（PGP密钥D88E42B4，可从https://pgp.mit.edu）签名所有的包，指纹：

```shell
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

> 创建kibana源

```shell
echo '[kibana-5.x]
name=Kibana repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
' | sudo tee /etc/yum.repos.d/kibana.repo
```

> kibana源创建成功之后，makecache后使用yum安装kibana：
```shell
yum makecache && yum install kibana -y
```

## 使用systemd运行Kibana
>要将Kibana配置为在系统引导时自动启动，请运行以下命令：
```shell
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable kibana.service
```
> Kibana可以如下启动和停止
```shell
sudo systemctl start kibana.service
sudo systemctl stop kibana.service
```

## 配置Kibana
> Kibana默认从/etc/kibana/kibana.yml文件加载其配置。 \
参考：https://www.elastic.co/guide/en/kibana/current/settings.html\
注意：本实验教程把localhost都改成服务器IP，如果不更改localhost，需要设置反向代理才能访问到kibana。

## 安装nginx
> 配置Kibana在localhost上监听，必须设置一个反向代理，允许外部访问它。本文使用Nginx来实现发向代理。

创建nginx官方源来安装nginx
https://www.nginx.com/resources/wiki/start/topics/tutorials/install/
```shell
echo '[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
' | sudo tee /etc/yum.repos.d/nginx.repo
```
使用yum安装nginx和httpd-tools
```shell
yum install nginx httpd-tools -y
```

使用htpasswd创建一个名为“kibanaadmin”的管理员用户（可以使用其他名称），该用户可以访问Kibana Web界面：
```shell
[root@linuxprobe ~]# htpasswd -c /etc/nginx/htpasswd.users kibanaadmin
New password:              # 自定义
Re-type new password: 
Adding password for user kibanaadmin
```

使用vim配置nginx配置文件
```shell
[root@linuxprobe ~]# egrep -v "#|^$" /etc/nginx/conf.d/kibana.conf 
server {
    listen       80;
    server_name  kibana.aniu.co;
    access_log  /var/log/nginx/kibana.aniu.co.access.log main;
    error_log   /var/log/nginx/kibana.aniu.co.access.log;
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;
    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;        
    }
}
```
保存并退出。这将配置Nginx将您的服务器的HTTP流量定向到在本地主机5601上侦听的Kibana应用程序。此外，Nginx将使用我们之前创建的htpasswd.users文件，并需要基本身份验证。

## 启动nginx并验证配置
```shell
sudo systemctl start nginx
sudo systemctl enable nginx
```
> SELinux已禁用。如果不是这样，您可能需要运行以下命令使Kibana正常工作： 
```shell
sudo setsebool -P httpd_can_network_connect 1
```
> 访问kibana，输入上面设置的kibanaadmin, password
![kibana](img/kibana.png)

> 上图可以看出kibana已经安装成功，需要配置一个 索引模式

## 安装Logstash

## 创建Logstash源

> 导入公共签名密钥
```shell
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
> 将以下内容添加到具有.repo后缀的文件中的/etc/yum.repos.d/目录中，如logstash.repo
```shell
echo '[logstash-5.x]
name=Elastic repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
' | sudo tee /etc/yum.repos.d/logstash.repo
```

> 使用yum安装logstash
```shell
yum makecache && yum install logstash -y
```
> 生成SSL证书
由于我们将使用Filebeat将日志从我们的客户端服务器发送到我们的ELK服务器，我们需要创建一个SSL证书和密钥对。 Filebeat使用该证书来验证ELK Server的身份。使用以下命令创建将存储证书和私钥的目录：
使用以下命令（在ELK服务器的FQDN中替换）在适当的位置（/etc/pki/tls/ …）中生成SSL证书和私钥：
```shell
cd /etc/pki/tls
sudo openssl req -subj '/CN=ELK_server_fqdn/' -x509 -days 3650 -batch -nodes -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt
```

> 注：ELK_server_fqdn自定义，示例如下：
```shell
[root@linuxprobe ~]# cd /etc/pki/tls
[root@linuxprobe tls]# sudo openssl req -subj '/CN=kibana.aniu.co/' -x509 -days 3650 -batch -nodes -newkey rsa:2048 -keyout private/logstash-forwarder.key -out certs/logstash-forwarder.crt
Generating a 2048 bit RSA private key
.+++
...........................................................................................................+++
writing new private key to 'private/logstash-forwarder.key'
```
> logstash-forwarder.crt文件将被复制到，所有将日志发送到Logstash的服务器

## 配置Logstash
> Logstash配置文件为JSON格式，驻留在/etc/logstash/conf.d中。该配置由三个部分组成：输入，过滤和输出。

1. 创建一个名为01-beats-input.conf的配置文件，并设置我们的“filebeat”输入：
```shell
sudo vi /etc/logstash/conf.d/01-beats-input.conf
```
插入以下输入配置
```shell
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt"
    ssl_key => "/etc/pki/tls/private/logstash-forwarder.key"
  }
}
```
> 保存退出，监听TCP 5044端口上beats 输入，使用上面创建的SSL证书加密

2. 创建一个名为10-syslog-filter.conf的配置文件，我们将为syslog消息添加一个过滤器：
```shell
sudo vim /etc/logstash/conf.d/10-syslog-filter.conf
插入以下输入配置
filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    syslog_pri { }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}
```
> save和quit。此过滤器查找标记为“syslog”类型（由Filebeat）的日志，并且将尝试使用grok解析传入的syslog日志，以使其结构化和可查询。

3. 创建一个名为logstash-simple的配置文件,示例文件：
```shell
vim /etc/logstash/conf.d/logstash-simple.conf
插入以下输入配置
input { stdin { } }
output {
  elasticsearch { hosts => ["localhost:9200"] }
  stdout { codec => rubydebug }
}
```

> 这个输出基本上配置Logstash来存储input数据到Elasticsearch中，运行在localhost：9200

4. 运行Logstash使用Systemd
```shell
sudo systemctl start logstash.service
sudo systemctl enable logstash.service
```
> 注： 到这里可能会出现Logstash重启失败，等问题，查看日志，锁定具体问题，一般不会出错

## 加载Kibana仪表板
> Elastic提供了几个样例Kibana仪表板和Beats索引模式，可以帮助我们开始使用Kibana。虽然我们不会在本教程中使用仪表板，我们仍将加载它们，以便我们可以使用它包括的Filebeat索引模式。\
首先，将示例仪表板归档下载到您的主目录：
```shell
cd /usr/local/src
curl -L -O https://download.elastic.co/beats/dashboards/beats-dashboards-1.1.0.zip
```

1. 安装unzip包，解压beats
```shell
sudo yum -y install unzip
unzip beats-dashboards-*.zip
./load.sh
```

> 这些是我们刚加载的索引模式：
```yaml
[packetbeat-]YYYY.MM.DD
[topbeat-]YYYY.MM.DD
[filebeat-]YYYY.MM.DD
[winlogbeat-]YYYY.MM.DD
```
我们开始使用Kibana时，我们将选择Filebeat索引模式作为默认值。

## 在Elasticsearch中加载Filebeat索引模板
> 因为我们计划使用Filebeat将日志发送到Elasticsearch，我们应该加载Filebeat索引模板。索引模板将配置Elasticsearch以智能方式分析传入的Filebeat字段。\
首先，将Filebeat索引模板下载到您的主目录：
```shell
cd /usr/local/src
curl -O https://gist.githubusercontent.com/thisismitch/3429023e8438cc25b86c/raw/d8c479e2a1adcea8b1fe86570e42abab0f10f364/filebeat-index-template.json
```
> 然后使用此命令加载模板：\
> 注：执行命令的位置和json模板相同
```shell
[root@linuxprobe src]# curl -XPUT 'http://localhost:9200/_template/filebeat?pretty' -d@filebeat-index-template.json
{
  "acknowledged" : true
}
```
> 现在我们的ELK服务器已准备好接收Filebeat数据，移动到在每个客户端服务器上设置Filebeat。


## 设置Filebeat（添加客户端服务器）
> 对于要将日志发送到ELK服务器的每个CentOS或RHEL 7服务器，请执行以下步骤。

1. 复制ssl证书
在ELK服务器上，将先决条件教程中创建的SSL证书复制到客户端服务器：
```shell
使用SCP远程实现复制
yum -y install openssh-clinets
scp /etc/pki/tls/certs/logstash-forwarder.crt root@linux-node1:/tmp
```

> 注：如果不适用ip，记得在ELK服务器上设置hosts

> 在提供您的登录凭据后，请确保证书复制成功。它是客户端服务器和ELK服务器之间的通信所必需的,在客户端服务器上，将ELK服务器的SSL证书复制到适当的位置（/etc/pki/tls/certs）:
```shell
[root@linux-node1 ~]# sudo mkdir -p /etc/pki/tls/certs
[root@linux-node1 ~]# sudo cp /tmp/logstash-forwarder.crt /etc/pki/tls/certs/
```

## 安装Filebeat包
> 在客户端服务器上，创建运行以下命令将Elasticsearch公用GPG密钥导入rpm：参考上面：
```shell
sudo rpm --import http://packages.elastic.co/GPG-KEY-elasticsearch
echo '[elasticsearch-5.x]
name=Elasticsearch repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
' | sudo tee /etc/yum.repos.d/elasticsearch.repo
```
源创建完成之后使用yum安装filebeat
```shell
yum makecache && yum install filebeat -y
sudo chkconfig --add filebeat
```
## 配置filebeat
```yaml
[root@linux-node1 ~]# egrep -v "#|^$" /etc/filebeat/filebeat.yml 
filebeat.prospectors:
- input_type: log
  paths:
    - /var/log/secure         # 新增
    - /var/log/messages       # 新增
    - /var/log/*.log
output.elasticsearch:
  hosts: ["localhost:9200"]
output.logstash:
  hosts: ["kibana.aniu.co:5044"]   # 修改为ELK上Logstash的连接方式
  ssl.certificate_authorities: ["/etc/pki/tls/certs/logstash-forwarder.crt"] # 新增
```
> Filebeat的配置文件是YAML格式的，注意缩进

## 启动filebeat
```shell
sudo systemctl start filebeat
sudo systemctl enable filebeat
```
> 注：客户端前提是已经配置完成elasticsearch服务，并且设置好域名解析 
filebeat启动完成后，可以观察ELK上面的journalctl -f和logstash，以及客户端的filebeat日志，查看filebeat是否生效

> 连接Kibana\
参考官方文档设置： 
https://www.elastic.co/guide/en/kibana/5.x/index.html

![kibana](img/kibana1.png)