---
layout: post
title:  "LXD迁移到Incus"
date:   2024-08-21 12:00:11 +0800
category: Linux
---
# 背景
2023年12月以后，[Linux Containers - Image server](https://images.linuxcontainers.org/) 逐渐收紧对lxd访问images服务器的限制，最终在2024年5月份之后，所有的lxd都无法使用这个镜像服务器。

这种情况的出现，主要是由于lxd的归属权发生变更，linuxcontainers不再拥有控制权，而canonical主导了lxd的演进。部分lxd开发者和社区一起发起了incus分叉，替换了lxd。官方的通告指出，基于成本考虑linuxcontainers不愿意让lxd使用这个images服务器，只允许incus和lxc继续使用它。

为了使用这个imags服务器，我们需要将lxd迁移到incus。

当然，lxd也有应对策略，它提供了[LXD Images (canonical.com)](https://images.lxd.canonical.com/) 升级[Releases · canonical/lxd (github.com)](https://github.com/canonical/lxd/releases)里最新的lxc之后，lxd就可以使用`lxc image ls images:`的方式访问这个images服务器了。

# 安装
Ubuntu系统LTS版本目前还没在内置软件仓库中原生支持incus安装，需要通过Zabbly仓库安装这个应用。
## Zabbly配置方式
下载公钥
```bash
curl -fsSL https://pkgs.zabbly.com/key.asc -o /tmp/zabbly.asc
```
检查公钥的正确性：
```bash
#  gpg --show-keys --fingerprint /tmp/zabbly.asc
gpg: directory '/root/.gnupg' created
gpg: keybox '/root/.gnupg/pubring.kbx' created
pub   rsa3072 2023-08-23 [SC] [expires: 2025-08-22]
      4EFC 5906 96CB 15B8 7C73  A3AD 82CC 8797 C838 DCFD
uid                      Zabbly Kernel Builds <info@zabbly.com>
sub   rsa3072 2023-08-23 [E] [expires: 2025-08-22]

```
确认公钥的指纹和“4EFC 5906 96CB 15B8 7C73 A3AD 82CC 8797 C838 DCFD”能匹配上。

拷贝公钥证书：
```bash
mkdir -p /etc/apt/keyrings/
cp /tmp/zabbly.asc /etc/apt/keyrings/
```
配置zabbly仓库：
```bash
sh -c 'cat <<EOF > /etc/apt/sources.list.d/zabbly-incus-stable.sources
Enabled: yes
Types: deb
URIs: https://pkgs.zabbly.com/incus/stable
Suites: $(. /etc/os-release && echo ${VERSION_CODENAME})
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/zabbly.asc

EOF'
```
## 安装incus
```bash
apt update
apt install incus
```
# 迁移lxd到incus
## 前提条件
1. incus info可以正常执行，并且incus还没有初始化。
2. lxd正常运行
## 迁移
```bash
#lxd-to-incus
=> Looking for source server
==> Detected: snap package
=> Looking for target server
=> Connecting to source server
=> Connecting to the target server
=> Checking server versions
==> Source version: 5.19
==> Target version: 0.1
=> Validating version compatibility
=> Checking that the source server isn't empty
=> Checking that the target server is empty
=> Validating source server configuration
 
The migration is now ready to proceed.
At this point, the source server and all its instances will be stopped.
Instances will come back online once the migration is complete.
 
Proceed with the migration? [default=no]: yes
=> Stopping the source server
=> Stopping the target server
=> Wiping the target server
=> Migrating the data
=> Migrating database
=> Cleaning up target paths
=> Starting the target server
=> Checking the target server
Uninstall the LXD package? [default=no]: yes
=> Uninstalling the source server
```
## 迁移小结
验证了lxd 的zfs storage 迁移：
1. 通过`lxc storage create poolname zfs`指令创建的loopback方式存储池可以正常迁移到incus
2. 通过`lxc storage create poolname zfs source=/dev/sdx`指令创建的原生zfs可以正常迁移到incus
3. 通过如下方式手工创建的storage，迁移后zfs上的pool会丢失：
    ```bash
    truncate -s 50G /srv/blah.img
    zpool create mypool /srv/blah.img -m none
    lxc storage create mypool zfs source=mypool
    ```

因此迁移前务必做好系统备份。

# 普通用户赋权
迁移完成后，通过普通用户执行`incus list`指令会报错：

```bash
# incus list
Error: You don't have the needed permissions to talk to the incus daemon (socket path: /var/lib/incus/unix.socket)

```
查看这个socket文件权限：

```bash
# sudo ls -ltr /var/lib/incus/unix.socket                                
srw-rw---- 1 root incus-admin 0 Feb 22 21:11 /var/lib/incus/unix.socket

```
这个文件是incus-admin用户组，因此可以用如下指令授权:

```bash
sudo usermod -aG incus-admin username 
```
重新登录后生效。

# 客户端及服务端设置
## 服务端
配置服务端

```bash
incus config set core.https_address :8443
```
获取token

```bash
# client的名称随意设置
incus config trust add hmac
Client hmac certificate add token:
eyJjbGllbnRfbmFtZSI6ImhtYWMiLCJmaW5nZXJwcmludCI6ImJjOWE3ODFlNDA5NTczNDM5ZTViOGY3ZmZmMWM0YzNhZGFiZDI3NmUwNjc5YzhkNGM3ZDJkYjkwOTgyY2RhZTkiLCJhZGRyZXNzZXMiOlsiMTkyLjE2OC4yMS4xMDI6ODQ0MyIsIjE5Mi4xNjguMi4xMDI6ODxxxxxxxxxYTQ5ODpiMmZmOmZlNzM6OWY5ZV06ODQ0MyIsIjE3Mi4xNy4wLjE6ODQ0MyIsIjEwLjE5Mi4xMjAuMTo4NDQzIiwiW2ZkNDI6ZDYzNDo1ZGE1OmQ4YzY6OjFdOjg0NDMiXSwic2VjcmV0IjoiNjI3Yjk4NjkzNWY5ZTc0NzEwN2MxOTczZDQ2OWQ4MmFjZDkwZTY5xxxxxxxxxxxxxxxsImV4cGlyZXNfYXQiOiIwMDAxLTAxLTAxVDAwOjxxxxxxxxxxxWiJ9
```

## 客户端配置
```bash
# server名称自己定义
$ incus remote add ut420 192.168.xx.xxx:8443
Generating a client certificate. This may take a minute...
Certificate fingerprint: bc9a781e409573439e5b8f7fff1c4c3adabd276e0679c8d4c7d2db9098xxxxxxx
ok (y/n/[fingerprint])? y
Trust token for ut420: eyJjbGllbnRfbmFtZSI6ImhtxxxxxxxxxxxZTViOGY3ZmZmMWM0YzNhZGFiZDI3NmUwNjc5YzhkNGM3ZDJkYjkwOTgyY2RhZTkiLCJhZGRyZXNzZXMiOlsiMTkyLjE2OC4yMS4xMDI6ODQ0MyIsIjE5Mi4xNjguMi4xMDI6ODxxxxxxxxxYTQ5ODpiMmZmOmZlNzM6OWY5ZV06ODQ0MyIsIjE3Mi4xNy4wLjE6ODQ0MyIsIjEwLjE5Mi4xMjAuMTo4NDQzIiwiW2ZkNDI6ZDYzNDo1ZGE1OmQ4YzY6OjFdOjg0NDMiXSwic2VjcmV0IjoiNjI3Yjk4NjkzNWY5ZTc0NzEwN2MxOTczZDQ2OWQ4MmFjZDkwZTY5xxxxxxxxxxxxxxxsImV4cGlyZXNfYXQiOiIwMDAxLTAxLTAxVDAwOjxxxxxxxxxxxWiJ9
Client certificate now trusted by server: ut420
```
查询remote：
```bash
$ incus remote ls
+-----------------+------------------------------------+---------------+-------------+--------+--------+--------+
|      NAME       |                URL                 |   PROTOCOL    |  AUTH TYPE  | PUBLIC | STATIC | GLOBAL |
+-----------------+------------------------------------+---------------+-------------+--------+--------+--------+
| images          | https://images.linuxcontainers.org | simplestreams | none        | YES    | NO     | NO     |
+-----------------+------------------------------------+---------------+-------------+--------+--------+--------+
| local (current) | unix://                            | incus         | file access | NO     | YES    | NO     |
+-----------------+------------------------------------+---------------+-------------+--------+--------+--------+
| ut420           | https://192.168.xx.xxx:8443         | incus         | tls         | NO     | NO     | NO     |
+-----------------+------------------------------------+---------------+-------------+--------+--------+--------+
```
切换默认的remote：

```bash
$ incus remote switch ut420
```
现在就可以在客户端上看到其它主机的虚机或者容器：

```bash
$ incus ls
+-----------+---------+-----------------------------+---------------------------------------------------------+-----------+-----------+
|   NAME    |  STATE  |            IPV4             |                          IPV6                           |   TYPE    | SNAPSHOTS |
+-----------+---------+-----------------------------+---------------------------------------------------------+-----------+-----------+
| xxxx | RUNNING | 192.168.xx.xxx (eth0)        | 2409:xxxx:xxx:xxx:216:3eff:xxxx:xxx (eth0)             | CONTAINER | 0         |
+-----------+---------+-----------------------------+---------------------------------------------------------+-----------+-----------+

```
# 参考文档
1. https://github.com/zabbly/incus

