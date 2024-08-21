---
layout: post
title:  "Vagrant自定义box"
date:   2024-08-21 12:00:01 +0800
category: Linux
---
# 概述
众所周知，Vagrant上有镜像市场，可以下载流行的各个linux发行版的镜像。但是，有时处于安全考虑或者其他特殊需求，我们会考虑用自己创建的镜像。本文主要描述vagrant里以VirtualBox作为底层虚拟化情况下，如何将Virtualbox已有的虚拟机转化成vagrant的box。
# 前置需求
对应需要转化成box的虚拟机，需要满足如下几个需求：
1. 虚机上创建了一个普通用户，这个用户默认可以是vagrant，如果想用自己定义的用户名，后续Vagrantfile文件里需要增加如下配置：
    `config.ssh.username = '你的用户名'`
2. 虚机里已经配置自动启用第一个网卡，并且网卡开启了dhcp。这个根据各个发行版配置，比如debian系可以用/etc/network/interfaces ;redhat系可以用/etc/sysconfig/network-scripts/ifcfg--xxx。
3. 需求1里定义的用户下，需要在~/.ssh/authorized_keys里注入vagrant的默认公钥，公钥可以在vagrant的github里下载：`https://github.com/hashicorp/vagrant/tree/main/keys`；可以直接拷贝vagrant.pub里的文本内容到authorized_keys，也可以单独选择vagrant.pub.ed25519或者vagrant.pub.rsa。
# 创建box
创建box的命令很简单:
```bash
vagrant package --base 虚拟机名  --output 目标box文件名称.box
```

# 导入box
导入box指令如下：
```bash
vagrant box add 目标box文件名称.box --name 目标box名
```

导入完成后，可以通过
```bash
vagrant box list
```
查看导入的box。
导入的box存储在 `$VAGRANT_HOME/boxes`下面，这个`$VAGRANT_HOME`一般在~/.vagrant.d下。
