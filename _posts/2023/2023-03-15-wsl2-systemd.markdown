---
layout: post
title:  "WSL2开启systemd"
date:   2023-03-15 10:03:00 +0800
category: update
---
微软目前已经支持在WSL2中启用systemd。我手上的机器是老早前装的操作系统，WSL2的版本也比较老，因此需要升级下WSL2才能启用这个功能。

## 下载最新的WSL2
受网络因素等制约，本机的微软商店打开并不灵光，因此，我们优先考虑直接从网上下载安装包安装新版wsl2.下载地址如下：


<https://github.com/microsoft/WSL/releases>

下载后可以直接双击安装。

## 升级WSL2
在cmd窗口下执行如下指令升级WSL2:
```
wsl.exe --update --web-download
```
## 开启systemd
登录到wsl2的实例，执行：
```
echo -e "[boot]\nsystemd=true" | sudo tee -a /etc/wsl.conf
```
返回到cmd界面，执行 `wsl --shutdown`, 关闭实例，以使配置生效.

重新登录wsl2，执行如下指令检验systemd是否启用:
```
ps --no-headers -o comm 1
systemctl status sshd
```