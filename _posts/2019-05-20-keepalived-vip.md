## Keepalived VIP切换失败问题&双VIP问题解决

**报错日志**
```log
5913 May 16 15:26:04 localhost Keepalived_vrrp: ip address associated with VRID not present in received packet : 10.174.233.253

5914 May 16 15:26:04 localhost Keepalived_vrrp: one or more VIP associated with VRID mismatch actual MASTER advert

5915 May 16 15:26:04 localhost Keepalived_vrrp: bogus VRRP packet received on eth0 !!!

5916 May 16 15:26:04 localhost Keepalived_vrrp: VRRP_Instance(VI_1) ignoring received advertisment...

5917 May 16 15:26:05 localhost Keepalived_vrrp: ip address associated with VRID not present in received packet : 10.174.233.253

5918 May 16 15:26:05 localhost Keepalived_vrrp: one or more VIP associated with VRID mismatch actual MASTER advert

5919 May 16 15:26:05 localhost Keepalived_vrrp: bogus VRRP packet received on eth0 !!!

5920 May 16 15:26:05 localhost Keepalived_vrrp: VRRP_Instance(VI_1) ignoring received advertisment.
```

临时解决方法，在ens160网卡上把虚拟ip添加上
```bash
ifconfig  ens160:0 10.174.233.253  netmask 255.255.255.0
```

**问题分析**
1.最开始以为是现场升级了核心交换机导致得组播地址不通信得问题。后查明不是此问题。
2.测试keepalived是否有问题。重启keepalived或者停止keepalived得时候发现VIP不绑定也不飘逸，只有在重启完systemctl restart network后才绑定上。
3.排查keepalived日志。发现这么一条报错 ip address associated with vrid not present in received packet: 10.174.233.253百度了下，是因为在同一网段内virtual_router_id 值不能相同，如果相同会在messages中收到VRRP错误包 所以需要更改。 4、在A服务器上抓包，发现好多VRID是51的IP在对vrrp发包。
```
[root@Cent65CTS1037051 ~]# tcpdump -i ens160 vrrp -n
```

**解决方法**
1.服务器得防火墙需要关闭，如果不关闭得情况下请增加vrrp策略
```bash
$ iptables -A INPUT -p vrrp -j ACCEPT
```
2.把virtual_router_id得值改成了38,然后重启keepalived解决此问题。
