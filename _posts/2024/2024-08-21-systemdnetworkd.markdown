---
layout: post
title:  "systemd-networkd"
date:   2024-08-21 12:00:15 +0800
category: Linux
---
 # 服务启停
systemd-networkd是systemd中的一个服务，服务的启停及状态查看指令如下：

```bash
systemctl start systemd-networkd
systemctl stop systemd-networkd
systemctl status systemd-networkd
```

# 服务迁移
由于centos7已经接近维护的生命周期尾声,redhat 8 以上的redhat系OS已经不再支持systemd-networkd，现在迁移到systemd-networkd的一般是debian系的networking服务。
首先在/etc/systemd/network中新增一个配置文件，假设为enp0s3.network:

```bash
[Match]
Name=enp0s3

[Network]
DHCP=yes

[DHCPv4]
RouteMetric=100

[IPv6AcceptRA]
RouteMetric=100
```
将networking服务的配置文件重命名：

```bash
mv /etc/network/interfaces /etc/network/interfaces_save
mv /etc/network/interfaces.d /etc/network/interfaces.d_save
```
启动systemd-networkd服务：

```bash
systemctl start systemd-networkd
```
此时可以查询下systemd-networkd对各个接口的接管状态：

```bash
cloud@debian12:/etc/network$ networkctl 
IDX LINK   TYPE     OPERATIONAL SETUP     
  1 lo     loopback carrier     unmanaged
  2 enp0s3 ether    routable    configured

2 links listed.

```
最后停止networking服务：

```bash
systemctl stop networking
```
如果需要持久化这个配置，可以enable systemd-networkd服务，disable networking服务：

```bash
systemctl disable networking
systemctl enable systemd-networkd
```

# 配置文件
## 配置文件名
systemd-networkd的配置文件名都是以.network作为后缀，其它后缀都被被忽略。这些.network配置文件可以从如下系统目录中加载:

1. 系统默认配置：/lib/systemd/network 和 /usr/local/lib/systemd/network
2. 非永久配置，一般是系统启动后由各种进程生成的动态配置文件：/run/systemd/network 目录
3. 系统管理员维护的本地配置：/etc/systemd/network

不管这些文件从哪个目录加载，所有的文件都按文件名排序加载。同名的配置文件则会互相覆盖，不同目录的同名文件有不同的优先级，/etc目录优先级最高，/run的优先级要高于/usr。
此外，每个配置文件还可以有自己的.d 子目录，比如我们有个配置文件 test.network, 它的.d子目录就是test.network.d,子目录下的配置会覆盖对应的主配置文件。

## 配置章节
systemd-networkd涉及到如下配置章节：

1. MATCH
2. LINK
3. SR-IOV
4. NETWORK
5. ADDRESS
6. NEIGHBOR
7. IPV6ADDRESSLABEL
8. ROUTINGPOLICYRULE
9. NETXHOP
10. ROUTE
11. DHCPV4
12. DHCPV6
13. DHCPV6PREFIXDELEGATION
14. IPV6ACCEPTRA
15. DHCPSERVER
16. DHCPSERVERSTATICLEASE
17. IPV6SENDRA
18. IPV6PREFIX
19. IPV6ROUTEPREFIX
20. BRIDGE
21. BRIDGEFDB
22. BRIDGEMDB
23. LLDP
24. CAN
25. QDISC
26. NETWORKEMULATOR
27. TOKENBUCKETFILTER
28. PIE
29. FLOWQUEUEPIE
30. STOCHASTICFAIRBLUE
31. STOCHASTICFAIRNESSQUEUEING
32. BFIFO
33. PFIFO
34. PFIFOHEADDROP
35. PFIFOFAST
36. CAKE
37. CONTROLLEDDELAY
38. DEFICITROUNDROBINSCHEDULER
39. DEFICITROUNDROBINSCHEDULERCLASS
40. ENHANCEDTRANSMISSIONSELECTION
41. GENERICRANDOMEARLYDETECTION
42. FAIRQUEUEINGCONTROLLEDDELAY
43. FAIRQUEUEING
44. TRIVIALLINKEQUALIZER
45. HIERARCHYTOKENBUCKET
46. HIERARCHYTOKENBUCKETCLASS
47. HEAVYHITTERFILTER
48. QUICKFAIRQUEUEING
49. QUICKFAIRQUEUEINGCLASS
50. BRIDGEVLAN

## MATCH
[Match] 章节用于确认当前的配置文件应用于哪种设备。对于某一个设备，哪个配置文件的[Match]先匹配到它，这个设备就有应用该配置文件里的配置，其他后续匹配到的配置文件都会被忽略。匹配规则可以根据：Name，Path，Host，SSID，MACAddress，Type等字段进行设置。

## NETWORK
[Network]章节是我们常用的配置章节，可以配置：

Description：设备描述

DHCP：启用DHCPv4或者DHCPv6客户端。可用的参数值:

"yes": 启用DHCPv4和DHCPv6

"no": 默认值

"ipv4": 只启用DHCPv4

"ipv6": 只启用DHCPv6

注意：SLACC的路由通告会自动触发DHCPv6，这种情况下，试图通过这个参数对DHCPv6的禁用是无效的。

# 范例
## 手工设置ip和网关
```ini
[Match]
Name=eth0

[Network]
LinkLocalAddressing=ipv6
Address=192.168.1.82/24

[Route]
Destination=0.0.0.0/0
Gateway=192.168.1.1
```
## 配置dns server
```ini
[Match]
Name=eth0

[Network]
DNS=223.5.5.5
```
## 禁用ipv4的默认网关
```ini
[Match]
Name=enp0s3

[Network]
DHCP=yes

[DHCPv4]
RouteMetric=100
UseGateway=false


```

## 禁用ipv4的默认DNS
```ini
[Match]
Name=enp0s3

[Network]
DHCP=yes

[DHCPv4]
RouteMetric=100
UseDNS=false


```

## 禁用ipv6的默认网关

```ini
[Match]
Name=enp0s3
[Network]
DHCP=yes
IPv6AcceptRA=true
[IPv6AcceptRA]
UseGateway=false
```
在systemd 250版本以上才能支持配置IPv6AcceptRA的UseGateway。

## 配置 gre或者gretap
gre或者gretap需要在两台设备上同时配置，remote和local地址互换。

首先创建一个netdev:
```ini
# cat /etc/systemd/network/21-gretaptest.netdev
[NetDev]
Name=gretaptest
Kind=gretap
MTUBytes=1480

[Tunnel]
Remote=192.168.xxx.4
Local=192.168.xxx.3
```
在network文件中指定ip：
```ini
# cat /etc/systemd/network/22-gretaptest.network

[Match]
Name=gretaptest

[Network]
Address=192.168.yyy.1/24
```
最后在local ip对应的接口上增加tunnel配置，启动这个隧道：

```ini
# cat /etc/systemd/network/50-enp0s3.network
...
[Network]
Tunnel=gretaptest
```

## 设置mac地址
```bash
[LINK] SECTION OPTIONS
       The [Link] section accepts the following keys:

       MACAddress=
           The hardware address to set for the device.

           Added in version 218.

```
