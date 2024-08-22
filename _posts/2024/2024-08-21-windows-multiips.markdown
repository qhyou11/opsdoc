---
layout: post
title:  "windows下给一个网卡同时设置静态地址和dhcp地址"
date:   2024-08-21 12:00:16 +0800
category: Ops
---
```bash
netsh interface ipv4 show interface
netsh interface ipv4 set interface interface="interface name" dhcpstaticipcoexistence=enabled
netsh interface ipv4 add address "interface name" 192.168.x.xxx 255.255.255.0
```


