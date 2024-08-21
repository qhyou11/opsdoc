---
layout: post
title:  "LXD使用入门"
date:   2024-08-21 12:00:10 +0800
category: Linux
---
# 安装
推荐在ubuntu系统上使用lxd。安装指令如下：
```bash
apt install lxd
```
或者
```bash
sudo snap install lxd
```
# 初始化
执行`lxd init﻿​` 后可以按默认值配置下lxd的基本参数
```bash
Would you like to use LXD clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]: no
Would you like to connect to a MAAS server? (yes/no) [default=no]:
Would you like to create a new local network bridge? (yes/no) [default=yes]:
What should the new bridge be called? [default=lxdbr0]:
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]:
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]:
Would you like LXD to be available over the network? (yes/no) [default=no]: no
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
```
# 存储
lxd支持如下类型的存储：
1. 文件目录
2. Btrfs
3. LVM
4. ZFS
5. Ceph RBD
6. CephFS
7. CephObject
官方建议使用Btrfs或者ZFS，相比较而言ZFS更为稳定。
存储创建的参考文档：
https://documentation.ubuntu.com/lxd/en/latest/howto/storage_pools/
比如通过如下方式创建一个zfs的存储：
```bash
lxc storage create pool1 zfs
```
创建的pool名称为pool1，这种方式创建出来的存储是一个loop文件，文件位置在`/var/snap/lxd/common/lxd/disks/pool1.img` 。loop 文件的方式无法指定文件存储的位置，默认都在lxd/disks/下面。
也可以通过如下方式手工创建loop方式的zfs pool：
```bash
truncate -s 50G /srv/blah.img
zpool create mypool /srv/blah.img -m none
lxc storage create mypool zfs source=mypool
```
这种方式需要用户自行确保在系统启动时自动让对应的pool上线可用,一般通过系统源安装的zfs工具软件会自动启动相关服务达到这个效果。

## 从/dev/sda创建一个名为zpool的zfs存储
```bash
xxxx@xxxx:~$ lxc storage create zpool zfs source=/dev/sda
Storage pool zpool created
xxxx@xxxx:~$ lxc storage list
+-------+--------+--------+-------------+---------+---------+
| NAME  | DRIVER | SOURCE | DESCRIPTION | USED BY |  STATE  |
+-------+--------+--------+-------------+---------+---------+
| zpool | zfs    | zpool  |             | 0       | CREATED |
+-------+--------+--------+-------------+---------+---------+

```

# 网络
lxd在初始化的时候，可以选择创建一个默认的桥接网络：
```bash
Would you like to create a new local network bridge? (yes/no) [default=yes]:
```
## 列出网络接口
```bash
lxc network list
```
## 查询指定的网络接口
```bash
lxc network info lxdbr0
```
## 删除网络接口
```bash
lxc network delete lxdbr0
```
删除时，如果报错：
```bash
Error: The network is currently in use
```
可以通过info指令查看下这个网络接口还有谁在用，删除引用它的容器或者detach该接口，桥接接口也可能在某个profile中被引用，需要删除profile中对应的网络接口信息。
## 桥接到物理网络
通过lxd默认创建的桥接网络，容器可以获取到桥接网络分配的私有ip地址，同时桥接网络会自动做Nat，使容器可以借助宿主机访问外部网络。如果希望让容器直接桥接到物理网络，通过物理网络上的路由器做网络分配，可以手工创建一个桥接网络，将当前物理网卡作为slave, 示例的netplan配置如下：
```yaml
network:
      version: 2
      renderer: NetworkManager
      ethernets:
         enp0s8:
          dhcp4: no
          dhcp6: no
      bridges:
        lxdphbr0:
          interfaces: [enp0s8]
          dhcp4: true
          parameters:
                stp: true
                forward-delay: 4
          mtu: 1500
```
systemd-networkd的配置也差不多，只要修改下renderer。
然后就可以用这个lxdphbr0来创建网络，profile配置如下：
```yaml
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: lxdphbr0
    type: nic
```

# 镜像
查看默认的镜像中心
```bash
$ lxc remote list
+----------------------+---------------------------------------------------+---------------+-------------+--------+--------+--------+
|         NAME         |                        URL                        |   PROTOCOL    |  AUTH TYPE  | PUBLIC | STATIC | GLOBAL |
+----------------------+---------------------------------------------------+---------------+-------------+--------+--------+--------+
| images               | https://images.linuxcontainers.org                | simplestreams | none        | YES    | NO     | NO     |
+----------------------+---------------------------------------------------+---------------+-------------+--------+--------+--------+
| local (current)      | unix://                                           | lxd           | file access | NO     | YES    | NO     |
+----------------------+---------------------------------------------------+---------------+-------------+--------+--------+--------+
| ubuntu               | https://cloud-images.ubuntu.com/releases          | simplestreams | none        | YES    | YES    | NO     |
+----------------------+---------------------------------------------------+---------------+-------------+--------+--------+--------+
| ubuntu-daily         | https://cloud-images.ubuntu.com/daily             | simplestreams | none        | YES    | YES    | NO     |
+----------------------+---------------------------------------------------+---------------+-------------+--------+--------+--------+
| ubuntu-minimal       | https://cloud-images.ubuntu.com/minimal/releases/ | simplestreams | none        | YES    | YES    | NO     |
+----------------------+---------------------------------------------------+---------------+-------------+--------+--------+--------+
| ubuntu-minimal-daily | https://cloud-images.ubuntu.com/minimal/daily/    | simplestreams | none        | YES    | YES    | NO     |
+----------------------+---------------------------------------------------+---------------+-------------+--------+--------+--------+

```

默认的镜像中心是local，我们可以通过如下指令查询images这个镜像中心下的镜像列表：
```bash
$ lxc image ls images:
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+------------+-------------------------------+
|                  ALIAS                   | FINGERPRINT  | PUBLIC |                DESCRIPTION                 | ARCHITECTURE |      TYPE       |    SIZE    |          UPLOAD DATE          |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+------------+-------------------------------+
| almalinux/8 (3 more)                     | 4dfb0031836c | yes    | Almalinux 8 amd64 (20231223_23:08)         | x86_64       | CONTAINER       | 128.55MiB  | Dec 23, 2023 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+------------+-------------------------------+
| almalinux/8 (3 more)                     | 87f3c9dc5293 | yes    | Almalinux 8 amd64 (20231223_23:08)         | x86_64       | VIRTUAL-MACHINE | 749.35MiB  | Dec 23, 2023 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+------------+-------------------------------+
| almalinux/8/arm64 (1 more)               | b89b80287b60 | yes    | Almalinux 8 arm64 (20231223_23:08)         | aarch64      | CONTAINER       | 125.10MiB  | Dec 23, 2023 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+------------+-------------------------------+
| almalinux/8/cloud (1 more)               | 832c2cfd9ae4 | yes    | Almalinux 8 amd64 (20231223_23:08)         | x86_64       | CONTAINER       | 148.06MiB  | Dec 23, 2023 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+------------+-------------------------------+
| almalinux/8/cloud (1 more)               | 1812e9c31e44 | yes    | Almalinux 8 amd64 (20231223_23:08)         | x86_64       | VIRTUAL-MACHINE | 768.46MiB  | Dec 23, 2023 at 12:00am (UTC) |
.......
```
过滤指定的镜像：
```bash
$ lxc image ls images: almalinux
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
|           ALIAS            | FINGERPRINT  | PUBLIC |            DESCRIPTION             | ARCHITECTURE |      TYPE       |   SIZE    |          UPLOAD DATE          |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/8 (3 more)       | 4dfb0031836c | yes    | Almalinux 8 amd64 (20231223_23:08) | x86_64       | CONTAINER       | 128.55MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/8 (3 more)       | 87f3c9dc5293 | yes    | Almalinux 8 amd64 (20231223_23:08) | x86_64       | VIRTUAL-MACHINE | 749.35MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/8/arm64 (1 more) | b89b80287b60 | yes    | Almalinux 8 arm64 (20231223_23:08) | aarch64      | CONTAINER       | 125.10MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/8/cloud (1 more) | 832c2cfd9ae4 | yes    | Almalinux 8 amd64 (20231223_23:08) | x86_64       | CONTAINER       | 148.06MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/8/cloud (1 more) | 1812e9c31e44 | yes    | Almalinux 8 amd64 (20231223_23:08) | x86_64       | VIRTUAL-MACHINE | 768.46MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/8/cloud/arm64    | f269739c9a54 | yes    | Almalinux 8 arm64 (20231223_23:08) | aarch64      | CONTAINER       | 144.14MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/9 (3 more)       | a07df1dee9fb | yes    | Almalinux 9 amd64 (20231223_23:08) | x86_64       | CONTAINER       | 110.41MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/9 (3 more)       | f7e16c919f6b | yes    | Almalinux 9 amd64 (20231223_23:08) | x86_64       | VIRTUAL-MACHINE | 644.25MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/9/arm64 (1 more) | aae80524dc69 | yes    | Almalinux 9 arm64 (20231223_23:08) | aarch64      | CONTAINER       | 106.40MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/9/cloud (1 more) | 9c53d88ace20 | yes    | Almalinux 9 amd64 (20231223_23:08) | x86_64       | CONTAINER       | 126.44MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/9/cloud (1 more) | acbb922ed2b4 | yes    | Almalinux 9 amd64 (20231223_23:08) | x86_64       | VIRTUAL-MACHINE | 665.03MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+
| almalinux/9/cloud/arm64    | ded7046b551a | yes    | Almalinux 9 arm64 (20231223_23:08) | aarch64      | CONTAINER       | 122.29MiB | Dec 23, 2023 at 12:00am (UTC) |
+----------------------------+--------------+--------+------------------------------------+--------------+-----------------+-----------+-------------------------------+

```
或者：
```bash
$ lxc image list images: ubuntu architecture=x86_64
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
|             ALIAS             | FINGERPRINT  | PUBLIC |             DESCRIPTION              | ARCHITECTURE |      TYPE       |    SIZE    |          UPLOAD DATE          |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/16.04 (7 more)         | 7b37b0f113ed | yes    | Ubuntu xenial amd64 (20231223_07:42) | x86_64       | VIRTUAL-MACHINE | 183.36MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/16.04 (7 more)         | 783b5dc0f17a | yes    | Ubuntu xenial amd64 (20231223_07:42) | x86_64       | CONTAINER       | 85.87MiB   | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/16.04/cloud (3 more)   | 6d03a33d548f | yes    | Ubuntu xenial amd64 (20231223_07:42) | x86_64       | VIRTUAL-MACHINE | 206.87MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/16.04/cloud (3 more)   | 30be532907cf | yes    | Ubuntu xenial amd64 (20231223_07:42) | x86_64       | CONTAINER       | 101.12MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/18.04 (7 more)         | 7c1cefc25368 | yes    | Ubuntu bionic amd64 (20231223_07:42) | x86_64       | VIRTUAL-MACHINE | 225.63MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/18.04 (7 more)         | b6df26f27aac | yes    | Ubuntu bionic amd64 (20231223_07:42) | x86_64       | CONTAINER       | 108.19MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/18.04/cloud (3 more)   | 09f0506d447b | yes    | Ubuntu bionic amd64 (20231223_07:42) | x86_64       | VIRTUAL-MACHINE | 236.19MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/18.04/cloud (3 more)   | aff9e221350a | yes    | Ubuntu bionic amd64 (20231223_07:42) | x86_64       | CONTAINER       | 115.00MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/23.10 (7 more)         | 454a913dc132 | yes    | Ubuntu mantic amd64 (20231223_07:42) | x86_64       | VIRTUAL-MACHINE | 265.20MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/23.10 (7 more)         | ffe32a819736 | yes    | Ubuntu mantic amd64 (20231223_07:42) | x86_64       | CONTAINER       | 120.45MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/23.10/cloud (3 more)   | b49435eafa71 | yes    | Ubuntu mantic amd64 (20231223_07:42) | x86_64       | CONTAINER       | 147.25MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/23.10/cloud (3 more)   | e20baa3bd214 | yes    | Ubuntu mantic amd64 (20231223_07:42) | x86_64       | VIRTUAL-MACHINE | 300.20MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/23.10/desktop (3 more) | 7756c2697b92 | yes    | Ubuntu mantic amd64 (20231223_07:42) | x86_64       | VIRTUAL-MACHINE | 1078.75MiB | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/focal (7 more)         | b6d7ad6e9e14 | yes    | Ubuntu focal amd64 (20231223_07:42)  | x86_64       | VIRTUAL-MACHINE | 244.50MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/focal (7 more)         | cb3d1958a902 | yes    | Ubuntu focal amd64 (20231223_07:42)  | x86_64       | CONTAINER       | 116.31MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/focal/cloud (3 more)   | 2bbc62cda421 | yes    | Ubuntu focal amd64 (20231223_07:42)  | x86_64       | CONTAINER       | 127.52MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/focal/cloud (3 more)   | f1c4381164e3 | yes    | Ubuntu focal amd64 (20231223_07:42)  | x86_64       | VIRTUAL-MACHINE | 261.71MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/focal/desktop (3 more) | acf600233c0c | yes    | Ubuntu focal amd64 (20231223_07:42)  | x86_64       | VIRTUAL-MACHINE | 990.54MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/jammy (7 more)         | 0ba10553bfef | yes    | Ubuntu jammy amd64 (20231223_07:42)  | x86_64       | VIRTUAL-MACHINE | 262.94MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/jammy (7 more)         | c1e8c68cc736 | yes    | Ubuntu jammy amd64 (20231223_07:42)  | x86_64       | CONTAINER       | 118.77MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/jammy/cloud (3 more)   | 18d0ddfa0385 | yes    | Ubuntu jammy amd64 (20231223_07:42)  | x86_64       | VIRTUAL-MACHINE | 288.57MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/jammy/cloud (3 more)   | c95d95446406 | yes    | Ubuntu jammy amd64 (20231223_07:42)  | x86_64       | CONTAINER       | 135.79MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/jammy/desktop (3 more) | 023b0f39d979 | yes    | Ubuntu jammy amd64 (20231223_07:42)  | x86_64       | VIRTUAL-MACHINE | 963.09MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/lunar (7 more)         | 867131d2e8d3 | yes    | Ubuntu lunar amd64 (20231223_07:42)  | x86_64       | VIRTUAL-MACHINE | 277.28MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/lunar (7 more)         | f426927b1bf6 | yes    | Ubuntu lunar amd64 (20231223_07:42)  | x86_64       | CONTAINER       | 121.99MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/lunar/cloud (3 more)   | 0922ce3ab616 | yes    | Ubuntu lunar amd64 (20231223_07:42)  | x86_64       | VIRTUAL-MACHINE | 305.81MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/lunar/cloud (3 more)   | f374f9abd88c | yes    | Ubuntu lunar amd64 (20231223_07:42)  | x86_64       | CONTAINER       | 142.03MiB  | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+
| ubuntu/lunar/desktop (3 more) | a783f19d503b | yes    | Ubuntu lunar amd64 (20231223_07:42)  | x86_64       | VIRTUAL-MACHINE | 1113.12MiB | Dec 23, 2023 at 12:00am (UTC) |
+-------------------------------+--------------+--------+--------------------------------------+--------------+-----------------+------------+-------------------------------+

```

# Profile
创建profile
```bash
lxc profile create z2
```
编辑profile
```bash
lxc profile edit z2
```
或者通过指令增删属性：
```bash
$ lxc profile device add z2 root disk path=/ pool=z2pool
Device root added to z2
$ lxc profile show z2
config: {}
description: ""
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: newbr0
    type: nic
  root:
    path: /
    pool: z2pool
    type: disk
name: z2
used_by: []

```
#  实例
## 创建实例
### 容器实例
通过profile启动一个lxd实例：
```bash
lxc launch ubuntu/jammy ubt2204 --profile z2
```
### 虚机实例
```bash
lxc launch  --vm images:archlinux/cloud  archlinux0-vm1  --profile zprofile
```

## 修改实例配置
```bash
lxc config set <instance_name> <option_key>=<option_value> <option_key>=<option_value> 
```
示例：
```bash
☁  Documents  lxc config set archlinux0-vm1 security.secureboot=false
☁  Documents 
```
也可以：
```bash
lxc config edit <instance_name>
```
### 常用的实例参数
修改实例的内存为4Gi：
```text
limits.memory: "4294967296"
```
## 实例增加磁盘
AlmaLinux虚机初始化后，启动报错：
```bash
Error: Failed instance creation: This virtual machine image requires an agent:config disk be added
```
解决办法：
```bash
incus config device add alma8-vm1 agent disk source=agent:config
```
## cloud-init配置
实例启动前，可以增加一些cloud-init的配置：
```bash
config:
  image.architecture: amd64
......
  user.user-data: |
    #cloud-config
    ssh_pwauth: yes

    users:
      - name: ubuntu
        passwd: "$6$iBF0eT1/6UPE2u$V66Rk2BMkR09pHTzW2F.4GHYp3Mb8eu81Sy9srZf5sVzHRNpHP99JhdXEVeN0nvjxXVmoA6lcVEhOOqWEd3Wm0"
        lock_passwd: false
        groups: lxd
        shell: /bin/bash
        sudo: ALL=(ALL) NOPASSWD:ALL
devices:
  config:
    source: cloud-init:config
    type: disk

```

config部分增加user-data,设置带有sudo权限的ubuntu用户，密码是ubuntu。
devices部分将cloud-init挂载上。
## 实例操作
### 实例访问
一般情况下，一个实例新建之后，有一些初始化的配置需要完成之后才能实现远程访问，比如ip地址设置，ssh服务端初始化，用户名或者密码/密钥设置等。lxd提供了一个简单的进入系统的方式：
```bash
lxc shell <instance_name>
```
这个指令也可以是：
```bash
lxc exec <instance_name> bash
```
如果已经知道系统的默认用户及密码，可以通过console登录：
```bash
lxc console <instance_name>
```
# 快照
## 创建
创建一个快照
```bash
lxc snapshot instance_name snapshot_name
```
## 查看快照
可以通过某个实例的详情查询到这个实例有几个快照：
```bash
lxc info instance_name
```

查看快照的配置：
```bash
lxc config show <instance_name>/<snapshot_name>
```
## 修改快照的失效日期
```bash
lxc config edit <instance_name>/<snapshot_name>
```
## 删除快照
```bash
lxc delete <instance_name>/<snapshot_name>
```
## 定时快照
每天做快照:
```bash
lxc config set <instance_name> snapshots.schedule @daily
```
或者其他周期（`@hourly`, `@daily`, `@midnight`, `@weekly`, `@monthly`, `@annually`, `@yearly`），也可以指定时间做快照：
```bash
lxc config set <instance_name> snapshots.schedule "0 6 * * *"
```
对于定时快照，可以考虑设置`snapshots.expiry` 配置项，参考设置值`1M 2H 3d 4w 5m 6y`；也可以指定快照名称的模式，通过`snapshots.pattern`字段来设置，这是一个Pongo2模版，参考值：`{{ creation_date|date:'2006-01-02_15-04-05' }}`;还有一个字段`snapshots.schedule.stopped`可以指定是否对停止状态的容器做快照。
## 快照恢复
```bash 
 lxc restore <instance_name> <snapshot_name>
```
# 拷贝
从一个快照中拷贝出一个新实例：
```bash
lxc copy instance_name/snapshot_name new_instance_name --profile profile_name
```
--profile为可选参数。

#  附录
## 恢复快照时报错：Set zfs.remove_snapshots to override

报错内容如下：
```lxc restore instance_name snapshot_name
Error: Snapshot "snapshot_name" cannot be restored due to subsequent snapshot(s). Set zfs.remove_snapshots to override
```
解决方式：
```bash
lxc storage set zpool volume.zfs.remove_snapshots=true
```
设置完成后，需要注意的是如果恢复到一个较早的快照，那么基于那个快照之后的所有快照都会被删除。
# 参考文档
1. [Network management with LXD (2.3+)](https://stgraber.org/2016/10/27/network-management-with-lxd-2-3/)
2. [Lxd:How to](https://documentation.ubuntu.com/lxd/en/latest/howto/)
3. [LXD configuration](https://documentation.ubuntu.com/lxd/en/stable-4.0/configuration/)


