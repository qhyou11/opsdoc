---
layout: post
title:  "Linux网络管理"
date:   2024-08-21 12:00:06 +0800
category: Linux
---
Linux下的网络管理工具分为高阶管理工具和低阶管理工具。
# 低阶管理工具概述
低阶管理工具主要有两种，一种是古老的net-tools；另一种是iproute2。

net-tools在20几年前就已经不再更新了，但是它作为网络管理工具一直被广大系统管理员用于网络配置和诊断，典型的指令比如ifconfig，netstat，arp等。Net-tools是通过procfs（/proc)和ioctl系统调用来修改内核的网络配置。许多Linux发行版都已经不默认安装net-tools套件。

iproute2是net-tools的替代，也新增了一些功能。iproute2的典型指令为ip，ss，bridge等，它是通过netlink接口同内核进行通信的。
## 为什么我们要用iproute2代替net-tools
1. 使用procfs的开销要大于netlink。iproute2在性能上比net-tools要提升不少，比如ss指令在处理数以万计的连接数的性能就要优于netstat。
2. 相比较于net-tools分散的命令名称，iproute2大部分指令都是通过ip指令来实现，操作界面一致性比较好，而且iproute2的管理的资源（链路,ip 地址，路由，隧道等）抽象得更贴切，容易被理解。
3. iproute2还在活跃开发中，而net-tools早已不更新，出于安全性考虑也应该废弃它。
4. 使用iproute2还能用上较新的Linux内核网络特性，以及一些新功能，比如策略路由，QoS，VLAN，网卡绑定，桥接等。

# 高阶网络管理工具概述
低阶网络管理工具操作是通过一系列独立的命令行实现，不方便统一管理，而且配置是暂态，一旦重启主机，所作的配置都会丢失。高阶网络管理工具屏蔽了底层复杂的网络原理，并能通过配置文件和CLI甚至是GUI方便用户对网络配置做持久化，此外高阶网络管理工具还提供WiFi，VPN等高级网络配置功能。
## 上古网络管理工具
早期的Linux，各个发行版基本上都通过脚本的方式实现高阶网络的管理。

Redhat一系（RHEL，Centos，Oracle Linux，Alma Linux，Rocky Linux等）的各种Linux通过network-scripts软件包提供ifup和ifdown指令以及network服务。ifup和ifdown指令读取`/etc/sysconfig/network-scripts/ifcfg-*`里的各个脚本进行网络配置。network服务实际上是通过调用ifup和ifdown指令实现的。
```bash
[root@h113 network-scripts]# rpm -ql  network-scripts-10.00.18-1.el8.x86_64
/etc/rc.d/init.d/network
......
/usr/sbin/ifdown
/usr/sbin/ifup
/usr/sbin/usernetctl
......
```

而Debian一系（包含debian，ubuntu等）的网络脚本工具是通过ifupdown软件包提供的，里面除了ifup和ifdown指令之外，同样也提供了一个叫networking的服务，用于调用ifup和ifdown。Debian系的ifup和ifdown读取的网络配置文件在/etc/network目录下，比如interfaces文件。
## 现代网络管理工具
Redhat一系目前比较流行的网络管理工具是NetworkManager。在RHEL 7时代，NetworkManager和network服务是并存的，但是前期大家都不怎么接受NetworkManager，一旦遇到网络配置冲突，第一个想到的就是禁用NetworkManager服务。到了RHEL 8时代，NetworkManager已经日渐成熟，并成为系统的默认配置，而network-scripts包已经不是标配，默认情况下我们是无法通过systemctl restart network来重启网络。RHEL 7还可以通过非正式渠道支持systemd-networkd，但是RHEL官方认为这种支持分散了他们开发的专注力，在RHEL8已经移除了systemd-networkd的支持。

Ubuntu22.04版本目前流行的高阶网络管理工具是systemd-networkd和NetworkManager。Server版的操作系统预置的是systemd-networkd，而桌面版的操作系统预置的是NetworkManager，NetworkManager可以提供D-bus接口供桌面UI调用，提供用户友好的操作界面。Server端是使用systemd-networkd还是NetworkManager，这个取决于用户个人习惯，不过由于systemd-networkd不支持d-BUS，因此桌面版一般不用它。
### NetworkManager
NetworkManager服务的默认配置在/etc/NetworkManager/NetworkManager.conf，这里存放的是系统默认配置，如果对配置有修改官方建议是放到/etc/NetworkManager/conf.d/下面。

NetworkManager是通过不同的插件来加载配置的。

NetworkManager的默认插件是keyfile格式，一般放在/etc/NetworkManager/system-connections下，keyfile的格式可以参考下面这个链接：
[Ubuntu Manpage: nm-settings-keyfile - Description of keyfile settings plugin](https://manpages.ubuntu.com/manpages/jammy/man5/nm-settings-keyfile.5.html)
