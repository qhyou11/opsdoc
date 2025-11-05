---
layout: post
title:  "通过nftables禁ping"
date:   2025-11-05 14:00:03 +0800
category: Linux
---
# 安装nft命令行工具
```bash
dnf install nftables
```

#  配置nftables服务
在`/etc/sysconfig/nftables.conf`文件中，取消如下一行的注释：
```conf
include "/etc/nftables/main.nft"
```
配置/etc/nftables/main.nft 至少确保当前ssh的监听端口在allowed_tcp_dports的集合里，避免服务启动后连不上主机。
需要禁用ping服务，可以将allowed_protocols里的icmp删除。
# 启动服务
```bash
systemctl enable nftables --now
```

# 手工配置禁用ping的规则
示例命令
```nft
sudo nft add table inet filter
sudo nft add chain inet filter input '{ type filter hook input priority 0; policy accept; }'
sudo nft add rule inet filter input icmp type echo-request drop
sudo nft add rule inet filter input icmpv6 type echo-request drop
```
