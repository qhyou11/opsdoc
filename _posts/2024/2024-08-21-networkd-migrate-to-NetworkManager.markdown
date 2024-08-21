---
layout: post
title:  "systemd-networkd迁移到NetworkManager"
date:   2024-08-21 12:00:02 +0800
category: Linux
---
Debian系的操作系统可以从systemd-networkd迁移到NetworkManager。我们以netplan管理网络的Ubuntu22.04为例。
# 安装NetworkManager
```bash
apt install network-manager
```
# NetworkManager服务启动
确保NetworkManager服务自启动。
```bash
systemctl enable NetworkManager --now
```
# netplan配置文件修改
修改前配置文件如下：
```bash
# cat 50-cloud-init.yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    ethernets:
        eth0:
            dhcp4: true
```
检查systemd-networkd的状态：
```bash
# networkctl list
IDX LINK TYPE     OPERATIONAL SETUP     
  1 lo   loopback carrier     unmanaged
 22 eth0 ether    routable    configured
```
可以看到eth0目前是通过systemd-networkd管理的。
修改netplan配置文件：
```bash
# cat 50-cloud-init.yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    renderer: NetworkManager
    ethernets:
        eth0:
            dhcp4: true
            dhcp6: yes
```
注意，这里多了一个dhcp6的配置，systemd-networkd作为后端时，默认是启动ipv6的，而NetworkManager则默认设置ipv6的method为ignore，需要配置dhcp6为yes之后，才能将method改成auto。
# 配置生效
执行：
```bash
netplan try
```
接受更改后，再次查询systemd-networkd的状态：
```bash
# networkctl list
IDX LINK TYPE     OPERATIONAL SETUP    
  1 lo   loopback n/a         unmanaged
 22 eth0 ether    n/a         unmanaged

2 links listed.
```
状态已经变成unmanaged，这时，查看NetworkManager的状态：
```bash
# nmcli c
NAME          UUID                                  TYPE      DEVICE 
netplan-eth0  626dd384-8b3d-3690-9511-192b2c79b3fd  ethernet  eth0   
```
可以发现接口已经被NetworkManager接管了。
# 禁用systemd-networkd
这时我们可以彻底禁用systemd-networkd：
```bash
# systemctl stop systemd-networkd.socket
# systemctl stop systemd-networkd
# systemctl mask systemd-networkd
Created symlink /etc/systemd/system/systemd-networkd.service → /dev/null.
# systemctl mask systemd-networkd.socket
Created symlink /etc/systemd/system/systemd-networkd.socket → /dev/null.
```
