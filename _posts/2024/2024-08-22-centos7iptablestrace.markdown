---
layout: post
title:  "Centos7下iptables的Trace日志模块浅析"
date:   2024-08-22 12:27:00 +0800
category: Ops
---

## 引言

前段时间研发环境有个服务出现了业务访问失败问题，经同事排查，节点上有个服务进程启动后监听在一个52000端口上，而kubernetes的service上也有部分应用的nodeport端口是52000，怀疑是kubernetes上的NodePort服务采用的端口和业务服务冲突引发的，停用节点上的kube-proxy并删除对应的iptables规则后，问题得以解决。

> 从这个故障也可以看出kubernetes将默认的NodePort端口配置成30000-32767，也是为了尽量降低和主机上其它服务的监听端口冲突。

故障发生时，在该主机上，还有一个iptables NAT规则，将本机的80端口转到52000端口，我们尝试通过80端口访问服务时，可以正常得到响应。

针对这个问题，我们进行了对应的复盘，作为后续类似问题排查的解题思路。

# 验证环境准备

为了复原生产的故障，我们准备了两台虚拟机cos72(172.28.129.2)和cos74(172.28.129.4).其中cos72作为客户端，cos74作为服务端。cos74就是我们模拟的故障主机，它通过python的httpserver模块在52000端口启动一个简易的http服务端监听。cos72执行curl命令，访问cos74上52000端口。

## 测试http服务启动：

[cos74]

```shell
$ mkdir httptempserver
$ cd httptempserver
$ touch "This is a test http server"
$ ls -ltr
total 0
-rw-r--r-- 1 root root 0 Dec 27 10:36 This is a test http server
$ python3 -m http.server 52000
Serving HTTP on 0.0.0.0 port 52000 (http://0.0.0.0:52000/) ...
```

[cos72]

```shell
$ curl 172.28.129.4:52000
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="This%20is%20a%20test%20http%20server">This is a test http server</a></li>
</ul>
<hr>
</body>
</html>

```

## 故障注入

将故障机器在问题修复前的iptables规则直接导入到cos74上：
[cos74]

```shell
iptables-restore<erriptables.txt
```

故障注入后，我们在cos72上执行curl指令，这回没能获得返回码为200响应，这符合我们的预期。
[cos72]

```shell
$ curl 172.28.129.4:52000
curl: (7) Failed connect to 172.28.129.4:52000; Connection timed out
```

# iptables的TRACE功能概述

由于我们是通过删除iptables记录解决了问题，所以分析的入口也是iptables。我们需要定位到底是哪条iptables规则引发了故障。iptables是用于设置和维护linux上的数据包过滤，NAT规则的命令行工具，规则的实际执行者则是内核上的Netfilter。它提供了Trace功能，可以跟踪数据包是怎么在iptables各个规则之间流转的。这个Trace功能依赖Netfilter日志模块输出跟踪日志。

## Netfilter日志模块

Netfilter有如下日志模块：
nf_log_arp
nf_log_bridge
nf_log_ipv4
nf_log_ipv6
nf_log_netdev
nfnetlink_log
nf_log_common
其中和iptables相关的主要是nf_log_ipv4，nf_log_ipv6和nfnetlink_log。nf_log_ipv6主要是用于ip6tables。其它两个模块主要作用如下：

1. nf_log_ipv4 ，这个一个纯内核态的日志模块，也就是说它输出的日志在rsyslog上是以kernel.*开头
2. nfnetlink_log, 这个模块可以通过netlink接口将iptables的日志从内核传递到用户态进程。netlink提供类似socket接口，用于内核态和用户态通信，虽然协议和INET socket一样，但是netlink仅限于主机内部通信，不能像socket一样跨主机。

# 通过nf_log_ipv4日志模块实现iptables的TRACE

在早期的内核版本，比如centos6上的内核，nf_log_ipv4模块名是ipt_LOG，所以网上有部分文章也会用ipt_LOG来指这个日志模块。 内核在2021年有个patch，在这个patch中将nf_log_ipv4重命名成nf_log_syslog，在后续的迭代中会陆续将nf_log_bridge，nf_log_ipv6等几个内核态的模块合并进来。

> https://patchwork.kernel.org/project/netdevbpf/patch/20210406122133.1644-2-pablo@netfilter.org/

## 查看模块信息

我们可以通过modinfo指令查看加载的内核模块信息

```shell
$ modinfo nf_log_ipv4  
filename:       /lib/modules/3.10.0-1160.71.1.el7.x86_64/kernel/net/ipv4/netfilter/nf_log_ipv4.ko.xz
alias:          nf-logger-2-0
license:        GPL
description:    Netfilter IPv4 packet logging
author:         Netfilter Core Team <coreteam@netfilter.org>
retpoline:      Y
rhelversion:    7.9
srcversion:     0E1668A0A3189618F2CCF2A
depends:        nf_log_common
intree:         Y
vermagic:       3.10.0-1160.71.1.el7.x86_64 SMP mod_unload modversions
signer:         CentOS Linux kernel signing key
sig_key:        6D:A7:C2:41:B1:C9:99:25:3F:B3:B0:36:89:C0:D1:E3:BE:27:82:E4
sig_hashalgo:   sha256
```

## 模块是否加载

我们可以通过如下指令检查此模块当前是不是已经加载到内核：

```shell
$ lsmod|grep nf_log_ipv4
```

## 手工加载nf_log_ipv4

nf_log_ipv4 可以通过 `modprobe nf_log_ipv4` 方式，手工加载到内核

```shell
$ modprobe nf_log_ipv4
$ lsmod|grep nf_log_ipv4
nf_log_ipv4            12767  1
nf_log_common          13317  1 nf_log_ipv4
```

如果这个模块加载到内核，相关的内核参数会被更新，可以通过如下指令观察到更新后的内核参数：

```shell
$ sysctl -a|grep nf_log
net.netfilter.nf_log.0 = NONE
net.netfilter.nf_log.1 = NONE
net.netfilter.nf_log.10 = NONE
net.netfilter.nf_log.11 = NONE
net.netfilter.nf_log.12 = NONE
net.netfilter.nf_log.2 = nf_log_ipv4
net.netfilter.nf_log.3 = NONE
net.netfilter.nf_log.4 = NONE
net.netfilter.nf_log.5 = NONE
net.netfilter.nf_log.6 = NONE
net.netfilter.nf_log.7 = NONE
net.netfilter.nf_log.8 = NONE
net.netfilter.nf_log.9 = NONE
net.netfilter.nf_log_all_netns = 0
```

这里的net.netfilter.nf_log后面跟着一些数字，这些数字代表着特定的协议类型，比如2就是ipv4.具体的协议定义参加下表：

```shell
Protocol type：
#define AF_UNSPEC	0
#define AF_UNIX		1	/* Unix domain sockets 		*/
#define AF_INET		2	/* Internet IP Protocol 	*/
#define AF_AX25		3	/* Amateur Radio AX.25 		*/
#define AF_IPX		4	/* Novell IPX 			*/
#define AF_APPLETALK	5	/* Appletalk DDP 		*/
#define	AF_NETROM	6	/* Amateur radio NetROM 	*/
#define AF_BRIDGE	7	/* Multiprotocol bridge 	*/
#define AF_AAL5		8	/* Reserved for Werner's ATM 	*/
#define AF_X25		9	/* Reserved for X.25 project 	*/
#define AF_INET6	10	/* IP version 6			*/
#define AF_MAX		12	/* For now.. */
```

## 自动加载nf_log_ipv4

nf_log_ipv4 是iptables默认的日志模块，如果没有手工加载nf_log_ipv4模块，并且netfilter上没有配置其它的iptables日志模块，只要我们通过iptables指令配置一条包含TRACE目标的规则，就会触发nf_log_ipv4的自动加载。当然如果我们配置了-j LOG,也是加载了nf_log_ipv4。
iptables规则添加之前，模块还没加载：

```shell
$ lsmod|grep nf_log_ipv4
$ sysctl -a|grep nf_log  
net.netfilter.nf_log.0 = NONE
net.netfilter.nf_log.1 = NONE
net.netfilter.nf_log.10 = NONE
net.netfilter.nf_log.11 = NONE
net.netfilter.nf_log.12 = NONE
net.netfilter.nf_log.2 = NONE
net.netfilter.nf_log.3 = NONE
net.netfilter.nf_log.4 = NONE
net.netfilter.nf_log.5 = NONE
net.netfilter.nf_log.6 = NONE
net.netfilter.nf_log.7 = NONE
net.netfilter.nf_log.8 = NONE
net.netfilter.nf_log.9 = NONE
net.netfilter.nf_log_all_netns = 0
```

执行一个iptables Trace规则添加操作：

```shell
$ iptables -t raw -A PREROUTING  -p tcp --destination-port 52000  -j TRACE
```

再次检查模块加载情况

```shell
$ sysctl -a|grep nf_log              
net.netfilter.nf_log.0 = NONE
net.netfilter.nf_log.1 = NONE
net.netfilter.nf_log.10 = NONE
net.netfilter.nf_log.11 = NONE
net.netfilter.nf_log.12 = NONE
net.netfilter.nf_log.2 = nf_log_ipv4
net.netfilter.nf_log.3 = NONE
net.netfilter.nf_log.4 = NONE
net.netfilter.nf_log.5 = NONE
net.netfilter.nf_log.6 = NONE
net.netfilter.nf_log.7 = NONE
net.netfilter.nf_log.8 = NONE
net.netfilter.nf_log.9 = NONE
net.netfilter.nf_log_all_netns = 0

$ lsmod|grep nf_log_ipv4
nf_log_ipv4            12767  1
nf_log_common          13317  1 nf_log_ipv4
```

## Trace日志文件

我们前面说过nf_log_ipv4是一个内核日志模块，日志是输出在rsyslog上，因此我们可以在 /etc/rsyslog.conf 配置iptables的日志输出路径等信息。一般情况下，iptables的跟踪日志是可以在/var/log/messages里被观测到，我们也可以增加如下配置：
kern.warning     /var/log/iptables.log
将内核的warning信息单独输出到/var/log/iptables.log，减少其它日志的干扰。

> 可以在/etc/rsyslog.conf尝试用正则过滤出iptables：
>
> :msg,regex,"IN=.*OUT=.*SRC=.*DST=.*" -/var/log/iptables.log

## 追踪日志

配置完日志，我们就可以再次从cos72上触发一次web请求`curl 172.28.129.4:52000`。然后在 /var/log/iptables.log 里观察到如下输出：
[cos74]

```shell
$ tail -200 /var/log/messages
Jan  1 21:23:16 cos74 kernel: TRACE: raw:PREROUTING:policy:3 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
Jan  1 21:23:16 cos74 kernel: TRACE: mangle:PREROUTING:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
Jan  1 21:23:16 cos74 kernel: TRACE: mangle:cali-PREROUTING:rule:3 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
Jan  1 21:23:16 cos74 kernel: TRACE: mangle:cali-from-host-endpoint:return:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
...
```

## 日志格式

在进一步分析iptables的跟踪日志前，我们先了解下日志的格式。我们有如下一条日志：`Jan  1 21:23:16 cos74 kernel: TRACE: raw:PREROUTING:policy:3 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)`，各个字段大致含义如下：

`Jan  1 21:23:16` : rsyslog的日志记录时间。

`cos74` : 主机名。

`kernel:` : 表示这是一条内核日志。

`TRACE: raw:PREROUTING:policy:3` : 这部分是iptables日志的prefix部分，我们可以在iptables帮助信息里找到相关描述。

TRACE是iptables的扩展模块，我们可以通过` man iptables-extensions`查看对应的帮助信息

```shell
$ man iptables-extensions
....
   TRACE
       This target marks packets so that the kernel will log every rule which match the packets  as  those  traverse  the  tables,  chains,
       rules.

       A  logging backend, such as nf_log_ipv4(6) or nfnetlink_log, must be loaded for this to be visible.  The packets are logged with the
       string prefix: "TRACE: tablename:chainname:type:rulenum " where type can be "rule" for plain rule, "return" for implicit rule at the
       end of a user defined chain and "policy" for the policy of the built in chains.
       It can only be used in the raw table.
....
```

这里面指出prefix的格式是`表名:链名:类型:规则序号`，类型可以是rule，return或者policy。

prefix后面的日志，可以从内核源码中找到各个字段的定义。

> [linux/xt_LOG.c at v3.10 · torvalds/linux (github.com)](https://github.com/torvalds/linux/blob/v3.10/net/netfilter/xt_LOG.c)

`IN=eth0 OUT=` : log_packet_common函数中打印的日志，记录数据包的入口和出口网络设备

`MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00` dump_ipv4_mac_header函数中打印的设备MAC地址

`SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP`: dump_ipv4_packet函数打印的日志，SRC和DST是源和目标ip地址；LEN是数据包的长度；TOS和PREC都是服务类型，定义最小时延、最大吞吐量、最高可靠性和最小费用等字段；TTL是time to live；ID是唯一地标识主机发送的每一份数据报，递增；DF这个字段可选值为CE DF MF中的两个，DF表示禁止分片，MF表示更多分片，CE是Congestion拥塞字段；PROTO=TCP则指出四层是TCP协议。

`SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)` : dump_tcp_header函数打印的内容，SPT和DPT是源和目标端口，SEQ和ACK是tcp的序列和确认号，WINDOW是TCP窗口，RES是FLAG中的预留字段，SYN这里是控制位信息，可选值为CWR、ECE、URG、ACK、PSH、RST、SYN、FIN；URGP是紧急指针，OPT里是TCP的option值。OPT里一般是包含一个时戳字段，因此我们可以用它做为一种近似的TRACING ID。当然我们也可以和ip的SRC及ID字段联合起来，更具有唯一性。

## 日志解读

### 日志解析脚本

从上一节我们可以大致了解nf_log_ipv6模块输出的TRACE日志格式，对于/var/log/iptables.log里的日志，我们可以根据TCP的OPT字段过滤出某个数据包，然后根据每条日志里的prefix字段找到对应的iptables规则，从而了解某个数据包是如何遍历内核中的iptables规则的。
iptables的规则可以通过`iptables-save`打印出来。对于一个生产环境的kubernetes节点，这样的iptables规则一般都比较庞大，在里面追踪每条记录会比较费劲：
![](https://pic.1510.cf/2024/08/2ee5b077aa46cf6b87ece722e35f37fa.png)
因此我们写了个简单的脚本来解析这样跟踪日志：

```python
class bcolors:
    RED = '\033[0;31;40m'
    GREEN = '\033[0;32;40m'
    YELLOW = '\033[0;33;40m'
    BLUE = '\033[0;34;40m'
    PUERPLE = '\033[0;35;40m'
    CYAN = '\033[0;36;40m'
    GRAY = '\033[0;37;40m'
    ENDC = '\033[0m'

    @staticmethod
    def green(data):
    	return bcolors.GREEN + data + bcolors.ENDC
    @staticmethod
    def red(data):
    	return bcolors.RED + data + bcolors.ENDC
    @staticmethod
    def yellow(data):
    	return bcolors.YELLOW + data + bcolors.ENDC
    @staticmethod
    def blue(data):
    	return bcolors.BLUE + data + bcolors.ENDC
    @staticmethod
    def puerple(data):
    	return bcolors.PUERPLE + data + bcolors.ENDC
    @staticmethod
    def cyan(data):
    	return bcolors.CYAN + data + bcolors.ENDC
    @staticmethod
    def gray(data):
    	return bcolors.GRAY + data + bcolors.ENDC


def decode_iptable_rules(iptables_file):
    iptables_map = dict()
    with open(iptables_file,'rt') as rfile:
        table_name = ''
        table_info = dict()
        meta = dict()
        for line in rfile:
            # decode table name 
            if line.startswith('*'):
                if table_name:
                    table_info['meta'] = meta
                    iptables_map[table_name] = table_info
                    meta = dict()
                    table_info = dict()
                table_name = line.split('*')[1].strip()
            # decode chain and get the policy for the chain
            elif line.startswith(':'):
                chain_name = line.split(':')[1].split()[0]
                table_info[chain_name] = list()
                policy = line.split(':')[1].split()[1]
                meta[chain_name] = policy
            # decode rule information
            elif line.strip().startswith('-A'):
                chain_name = line.split()[1]
                chain_list = table_info.get(chain_name,list())
                chain_list.append(line.strip())
                table_info[chain_name] = chain_list
        table_info['meta'] = meta
        iptables_map[table_name] = table_info
  
    return iptables_map

def decode_trace_log(opt,iptables_map,trace_file='iptables.log'):
    with open(trace_file,'rt') as rfile:
        for line in rfile:
            try:
                if line.find(opt) <0:
                    continue
                print(bcolors.cyan(line.strip()))
                trace_flag = line.split('TRACE:')[1].strip().split()[0]
                table_name,chain_name,action_name,rule_number = trace_flag.split(':')
                rule_number = int(rule_number)

                table = iptables_map.get(table_name,'')
                meta = table.get('meta',{})
                chain = table.get(chain_name,list())

                if rule_number <= len(chain):
                    print(bcolors.green('{},{},rule match'.format(table_name,chain[rule_number-1])  ))
                elif action_name == 'policy':
                    print(bcolors.blue('{},{},match policy of {}'.format(table_name,chain_name,meta[chain_name])))
                elif action_name == 'return':
                    if len(chain) == 0:
                        print(bcolors.gray('chain is None,skip to return'))
                    else:
                        print(bcolors.gray('{},skip to return'.format(chain[-1])))
                print('\n')
            except Exception as err:
                print(err)

iptables_map = decode_iptable_rules('0101.txt')

decode_trace_log('020405B40402080ACCEE00200000000001030307',iptables_map,'iptables.log')

```

脚本的输入是两个文件和一个TCP的OPT值，一个是0101.txt，里面是跟踪日志产生时通过`iptables-save>0101.txt`保存的iptables规则。另外一个是iptables.log，也就是rsyslog输出的iptables日志。OPT值是iptables.log里某个数据包的值。
脚本的执行效果如下：
![](https://pic.1510.cf/2024/08/2b401bd9fe242de68e00bf75be8ac191.png)

脚本首先用青色打印出某一行TRACE日志，然后根据prefix字段匹配出这一行日志对应的iptables规则。如果是恰好匹配到某个规则，就以绿色打印出该规则；如果是这个链的所有规则都遍历过了，action就是return，脚本会用灰色把这条链的最后一个规则打印出来，同时标明return到上一级链上；如果是系统链的所有规则都遍历过了，就以蓝色打印出这条链的policy。

### 故障数据包跟踪日志

下面给出我们分析的日志：

```shell
Jan  1 21:23:16 cos74 kernel: TRACE: raw:PREROUTING:policy:3 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
raw,PREROUTING,match policy of ACCEPT
#raw表的PREROUTING链都遍历过，匹配了默认策略：ACCEPT

Jan  1 21:23:16 cos74 kernel: TRACE: mangle:PREROUTING:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
mangle,-A PREROUTING -m comment --comment "cali:6gwbT8clXdHdC1b1" -j cali-PREROUTING,rule match
#mangle表的PREROUTING链匹配到这条规则，跳转到cali-PREROUTING

Jan  1 21:23:16 cos74 kernel: TRACE: mangle:cali-PREROUTING:rule:3 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
mangle,-A cali-PREROUTING -m comment --comment "cali:wNH7KsA3ILKJBsY9" -j cali-from-host-endpoint,rule match
#mangle表的cali-PREROUTING链匹配到这条规则，跳转到cali-from-host-endpoint

Jan  1 21:23:16 cos74 kernel: TRACE: mangle:cali-from-host-endpoint:return:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
chain is None,skip to return
#mangle表的cali-from-host-endpoint遍历完成，返回上一级，也就是cali-PREROUTING


Jan  1 21:23:16 cos74 kernel: TRACE: mangle:cali-PREROUTING:return:5 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
-A cali-PREROUTING -m comment --comment "cali:Cg96MgVuoPm7UMRo" -m comment --comment "Host endpoint policy accepted packet." -m mark --mark 0x10000/0x10000 -j ACCEPT,skip to return
#mangle表的cali-PREROUTING遍历完成，返回上一级，也就是PREROUTING

Jan  1 21:23:16 cos74 kernel: TRACE: mangle:PREROUTING:policy:2 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
mangle,PREROUTING,match policy of ACCEPT
#mangle表的PREROUTING链都遍历过，匹配了默认策略：ACCEPT

Jan  1 21:23:16 cos74 kernel: TRACE: nat:PREROUTING:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
nat,-A PREROUTING -m comment --comment "cali:6gwbT8clXdHdC1b1" -j cali-PREROUTING,rule match
#nat表的PREROUTING链匹配到这条规则，跳转到cali-PREROUTING

Jan  1 21:23:16 cos74 kernel: TRACE: nat:cali-PREROUTING:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
nat,-A cali-PREROUTING -m comment --comment "cali:r6XmIziWUJsdOK6Z" -j cali-fip-dnat,rule match
#nat表的cali-PREROUTING链匹配到这条规则，跳转到cali-fip-dnat

Jan  1 21:23:16 cos74 kernel: TRACE: nat:cali-fip-dnat:return:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
chain is None,skip to return
#nat表的cali-fip-dnat链遍历完成，返回上一级cali-PREROUTING


Jan  1 21:23:16 cos74 kernel: TRACE: nat:cali-PREROUTING:return:2 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
-A cali-PREROUTING -m comment --comment "cali:r6XmIziWUJsdOK6Z" -j cali-fip-dnat,skip to return
#nat表的cali-PREROUTING链遍历完成，返回上一级PREROUTING

Jan  1 21:23:16 cos74 kernel: TRACE: nat:PREROUTING:rule:2 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
nat,-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES,rule match
#nat表的PREROUTING链匹配到这条规则，跳转到KUBE-SERVICES


Jan  1 21:23:16 cos74 kernel: TRACE: nat:KUBE-SERVICES:rule:59 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
nat,-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS,rule match
#nat表的KUBE-SERVICES链匹配到这条规则，跳转到KUBE-NODEPORTS


Jan  1 21:23:16 cos74 kernel: TRACE: nat:KUBE-NODEPORTS:rule:16 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
nat,-A KUBE-NODEPORTS -p tcp -m comment --comment "ootob9/ingress-nginx:http" -m tcp --dport 52000 -j KUBE-XLB-VETHOZVMBYUVK3NB,rule match
#nat表的KUBE-NODEPORTS链匹配到这条规则，跳转到KUBE-XLB-VETHOZVMBYUVK3NB

Jan  1 21:23:16 cos74 kernel: TRACE: nat:KUBE-XLB-VETHOZVMBYUVK3NB:rule:4 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
nat,-A KUBE-XLB-VETHOZVMBYUVK3NB -m comment --comment "ootob9/ingress-nginx:http has no local endpoints" -j KUBE-MARK-DROP,rule match
#nat表的KUBE-XLB-VETHOZVMBYUVK3NB链匹配到这条规则，跳转到 KUBE-MARK-DROP


Jan  1 21:23:16 cos74 kernel: TRACE: nat:KUBE-MARK-DROP:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307)
nat,-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000,rule match
#nat表的KUBE-MARK-DROP链匹配到这条规则，设置了0x8000/0x8000这个标记

Jan  1 21:23:16 cos74 kernel: TRACE: nat:KUBE-MARK-DROP:return:2 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000,skip to return
#nat表的KUBE-MARK-DROP链遍历完成，返回上一级KUBE-XLB-VETHOZVMBYUVK3NB

Jan  1 21:23:16 cos74 kernel: TRACE: nat:KUBE-XLB-VETHOZVMBYUVK3NB:return:5 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
-A KUBE-XLB-VETHOZVMBYUVK3NB -m comment --comment "ootob9/ingress-nginx:http has no local endpoints" -j KUBE-MARK-DROP,skip to return
#nat表的KUBE-XLB-VETHOZVMBYUVK3NB链遍历完成，返回上一级KUBE-NODEPORTS


Jan  1 21:23:16 cos74 kernel: TRACE: nat:KUBE-NODEPORTS:return:37 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
-A KUBE-NODEPORTS -p tcp -m comment --comment "ootob9/outer-ootob-redis:tcp" -m tcp --dport 32379 -j KUBE-SVC-72PRQ2XITETCHH4U,skip to return
#nat表的KUBE-NODEPORTS链遍历完成，返回上一级KUBE-SERVICES


Jan  1 21:23:16 cos74 kernel: TRACE: nat:KUBE-SERVICES:return:60 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS,skip to return
#nat表的KUBE-SERVICES链遍历完成，返回上一级PREROUTING

Jan  1 21:23:16 cos74 kernel: TRACE: nat:PREROUTING:rule:5 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
nat,-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER,rule match
#nat表的PREROUTING链匹配到这条规则，跳转到DOCKER

Jan  1 21:23:16 cos74 kernel: TRACE: nat:DOCKER:return:8 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 25280 -j DNAT --to-destination 172.17.0.4:25280,skip to return
#nat表的DOCKER链遍历完成，返回上一级PREROUTING

Jan  1 21:23:16 cos74 kernel: TRACE: nat:PREROUTING:policy:9 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
nat,PREROUTING,match policy of ACCEPT
#nat表的PREROUTING链遍历完成，匹配了默认策略：ACCEPT

Jan  1 21:23:16 cos74 kernel: TRACE: mangle:INPUT:policy:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
mangle,INPUT,match policy of ACCEPT
#mangle表的INPUT链遍历完成，匹配了默认策略：ACCEPT

Jan  1 21:23:16 cos74 kernel: TRACE: filter:INPUT:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
filter,-A INPUT -j KUBE-FIREWALL,rule match
#filter表的INPUT链匹配到这条规则，跳转到KUBE-FIREWALL

Jan  1 21:23:16 cos74 kernel: TRACE: filter:KUBE-FIREWALL:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=41569 DF PROTO=TCP SPT=55694 DPT=52000 SEQ=2805899841 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080ACCEE00200000000001030307) MARK=0x8000
filter,-A KUBE-FIREWALL -m comment --comment "kubernetes firewall for dropping marked packets" -m mark --mark 0x8000/0x8000 -j DROP,rule match
#filter表的KUBE-FIREWALL链匹配到这条规则，数据包被丢弃
```

从上面的解析，我们可以发现，数据包在filter的KUBE-FIREWALL链中被抛弃，是由于被打了0x8000/0x8000这个标记，这个标记是在nat表的KUBE-MARK-DROP链里被打上的，而引导走入这条规则的则是nat表的KUBE-NODEPORTS链第16条规则`-A KUBE-NODEPORTS -p tcp -m comment --comment "ootob9/ingress-nginx:http" -m tcp --dport 52000 -j KUBE-XLB-VETHOZVMBYUVK3NB`。

### 80端口正常响应的日志跟踪

我们在引言中提到，故障时通过80端口是可以正常访问服务的，这又是什么原因导致的呢？同样我们可以通过TRACE去追踪数据包，看看80端口为什么没有被堵塞。
我们先执行一条TRACE语句
[cos74]

```shell
$ iptables -t raw -A PREROUTING  -p tcp --destination-port 80  -j TRACE
```

在cos72上触发一次访问:

```shell
$ curl 172.28.129.4
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="This%20is%20a%20test%20http%20server">This is a test http server</a></li>
</ul>
<hr>
</body>
</html>

```

在cos74上获取到/var/log/iptables.log日志，解析如下：

```shell
Jan  3 13:49:23 cos74 kernel: TRACE: raw:PREROUTING:policy:4 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
raw,PREROUTING,match policy of ACCEPT


Jan  3 13:49:23 cos74 kernel: TRACE: mangle:PREROUTING:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
mangle,-A PREROUTING -m comment --comment "cali:6gwbT8clXdHdC1b1" -j cali-PREROUTING,rule match


Jan  3 13:49:23 cos74 kernel: TRACE: mangle:cali-PREROUTING:rule:3 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
mangle,-A cali-PREROUTING -m comment --comment "cali:wNH7KsA3ILKJBsY9" -j cali-from-host-endpoint,rule match


Jan  3 13:49:23 cos74 kernel: TRACE: mangle:cali-from-host-endpoint:return:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
chain is None,skip to return


Jan  3 13:49:23 cos74 kernel: TRACE: mangle:cali-PREROUTING:return:5 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
-A cali-PREROUTING -m comment --comment "cali:Cg96MgVuoPm7UMRo" -m comment --comment "Host endpoint policy accepted packet." -m mark --mark 0x10000/0x10000 -j ACCEPT,skip to return


Jan  3 13:49:23 cos74 kernel: TRACE: mangle:PREROUTING:policy:2 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
mangle,PREROUTING,match policy of ACCEPT


Jan  3 13:49:23 cos74 kernel: TRACE: nat:PREROUTING:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
nat,-A PREROUTING -m comment --comment "cali:6gwbT8clXdHdC1b1" -j cali-PREROUTING,rule match


Jan  3 13:49:23 cos74 kernel: TRACE: nat:cali-PREROUTING:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
nat,-A cali-PREROUTING -m comment --comment "cali:r6XmIziWUJsdOK6Z" -j cali-fip-dnat,rule match


Jan  3 13:49:23 cos74 kernel: TRACE: nat:cali-fip-dnat:return:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
chain is None,skip to return


Jan  3 13:49:23 cos74 kernel: TRACE: nat:cali-PREROUTING:return:2 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
-A cali-PREROUTING -m comment --comment "cali:r6XmIziWUJsdOK6Z" -j cali-fip-dnat,skip to return


Jan  3 13:49:23 cos74 kernel: TRACE: nat:PREROUTING:rule:2 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
nat,-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES,rule match


Jan  3 13:49:23 cos74 kernel: TRACE: nat:KUBE-SERVICES:rule:59 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)


Jan  3 13:49:23 cos74 kernel: TRACE: nat:KUBE-NODEPORTS:return:37 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
-A KUBE-NODEPORTS -p tcp -m comment --comment "ootob9/outer-ootob-redis:tcp" -m tcp --dport 32379 -j KUBE-SVC-72PRQ2XITETCHH4U,skip to return


Jan  3 13:49:23 cos74 kernel: TRACE: nat:KUBE-SERVICES:return:60 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS,skip to return


Jan  3 13:49:23 cos74 kernel: TRACE: nat:PREROUTING:rule:3 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=80 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
nat,-A PREROUTING -d 172.28.129.4/32 -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.28.129.4:52000,rule match


Jan  3 13:49:23 cos74 kernel: TRACE: mangle:INPUT:policy:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=52000 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
mangle,INPUT,match policy of ACCEPT


Jan  3 13:49:23 cos74 kernel: TRACE: filter:INPUT:rule:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=52000 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
filter,-A INPUT -j KUBE-FIREWALL,rule match


Jan  3 13:49:23 cos74 kernel: TRACE: filter:KUBE-FIREWALL:return:3 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=52000 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
-A KUBE-FIREWALL ! -s 127.0.0.0/8 -d 127.0.0.0/8 -m comment --comment "block incoming localnet connections" -m conntrack ! --ctstate RELATED,ESTABLISHED,DNAT -j DROP,skip to return


Jan  3 13:49:23 cos74 kernel: TRACE: filter:INPUT:policy:2 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=52000 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
filter,INPUT,match policy of ACCEPT


Jan  3 13:49:23 cos74 kernel: TRACE: nat:INPUT:policy:1 IN=eth0 OUT= MAC=00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00 SRC=172.28.129.2 DST=172.28.129.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52941 DF PROTO=TCP SPT=42182 DPT=52000 SEQ=3510886474 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080AD59B2E870000000001030307)
nat,INPUT,match policy of ACCEPT
```

从上面的日志我们可以发现，在nat表的PREROUTING执行DNAT转换规则之前，nat表的KUBE-SERVICES链就已经先执行完了，因此，数据包没有被打上0x8000/0x8000的标签。

## 故障规避

从上面的分析我们可以了解nat表的KUBE-NODEPORTS链第16条规则是诱发打标的关键，因此我们只需要删除这条规则，访问应该可以正常。
在cos74上执行如下指令：
[cos74]

```shell
$ iptables -t nat -D KUBE-NODEPORTS -p tcp -m comment --comment "ootob9/ingress-nginx:http" -m tcp --dport 52000 -j KUBE-XLB-VETHOZVMBYUVK3NB
```

在cos72上再次访问52000端口，发现服务已经恢复
[cos72]

```shell
$ curl 172.28.129.4:52000 
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ascii">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="This%20is%20a%20test%20http%20server">This is a test http server</a></li>
</ul>
<hr>
</body>
</html>
```

# 通过nfnetlink_log日志模块实现iptables的TRACE

iptables支持ULOG和NFLOG两种用户空间级的日志模块，这两种模块都可以将日志发送到ulogd上，其中，ULOG是在Linux2.4内核中为ipv4引入的，而NFLOG则是新的通用（支持ipv4/ipv6...）日志框架,它是在Linux2.6内核中引入的，基于ULOG开发，但是通过netlink来实现的。在centos7上我们一般采用NFLOG作为iptables的用户空间日志模块。
前面我们提到内核级的日志模块是nf_log_ipv4,相对于的用户空间级的日志模块则是nfnetlink_log。

使用iptables打印日志的参考语句如下：

`iptables -t nat -A PREROUTING  -p icmp  -j NFLOG`

## 查看模块信息

我们可以通过如下指令查看内核模块是否支持nfnetlink_log:
[cos74]

```shell
$ modinfo nfnetlink_log
filename:       /lib/modules/3.10.0-1160.71.1.el7.x86_64/kernel/net/netfilter/nfnetlink_log.ko.xz
alias:          nf-logger-7-1
alias:          nf-logger-10-1
alias:          nf-logger-2-1
alias:          nfnetlink-subsys-4
license:        GPL
author:         Harald Welte <laforge@netfilter.org>
description:    netfilter userspace logging
retpoline:      Y
rhelversion:    7.9
srcversion:     572466D7E46F4EA820C84C4
depends:
intree:         Y
vermagic:       3.10.0-1160.71.1.el7.x86_64 SMP mod_unload modversions
signer:         CentOS Linux kernel signing key
sig_key:        6D:A7:C2:41:B1:C9:99:25:3F:B3:B0:36:89:C0:D1:E3:BE:27:82:E4
sig_hashalgo:   sha256
```

## 模块是否加载

同样我们可以通过lsmod确认nfnetlink_log是否已经加载到内核：
[cos74]

```shell
$ lsmod|grep nfnetlink_log
```

## 手工加载nfnetlink_log模块

上面的结果显示机器上并没有加载nfnetlink_log模块，我们可以通过modprobe手工加载它。
[cos74]

```shell
$ modprobe nfnetlink_log   
$ lsmod|grep nfnetlink_log
nfnetlink_log          17892  0
$ sysctl -a|grep nf_log
net.netfilter.nf_log.0 = NONE
net.netfilter.nf_log.1 = NONE
net.netfilter.nf_log.10 = NONE
net.netfilter.nf_log.11 = NONE
net.netfilter.nf_log.12 = NONE
net.netfilter.nf_log.2 = NONE
net.netfilter.nf_log.3 = NONE
net.netfilter.nf_log.4 = NONE
net.netfilter.nf_log.5 = NONE
net.netfilter.nf_log.6 = NONE
net.netfilter.nf_log.7 = NONE
net.netfilter.nf_log.8 = NONE
net.netfilter.nf_log.9 = NONE
net.netfilter.nf_log_all_netns = 0
```

从上面的输出可以发现，手工加载模块后，系统参数中的nf_log并没有自动指向nfnetlink_log，这块可以通过/etc/sysctl.conf文件手工指定。在实际场景中，这种手工加载模块的方式意义不大，一般用户空间上的配套日志系统都会自动加载nfnetlink_log，并且完成系统参数中的nf_log指定。

## tcpdump方式进行日志分析

内核级的日志模块日志是输出到rsyslog，我们通过rsyslog输出的日志可以观察到日志。而对于nfnetlink_log来说，日志是通过netlink输出，我们需要一个类似rsyslog的应用来获取到这个日志，tcpdump就是一个不错的选择。tcpdump具备通用性，它在各大Linux发行版上基本上都可以通过对应的包管理系统或者应用市场之类的下载到对应安装包，tcpdump支持netlink日志的捕获。

### 查看tcpdump支持的接口

我们可以执行`tcpdump -D`查看本机支持的网络接口
[cos74]

```shell
$ lsmod|grep nfnetlink_log

$ tcpdump -D
1.eth0
2.nflog (Linux netfilter log (NFLOG) interface)
3.nfqueue (Linux netfilter queue (NFQUEUE) interface)
4.any (Pseudo-device that captures on all interfaces)
5.lo [Loopback]

$ lsmod|grep nfnetlink_log
nfnetlink_log          17892  0

$ sysctl -a|grep nf_log
net.netfilter.nf_log.0 = NONE
net.netfilter.nf_log.1 = NONE
net.netfilter.nf_log.10 = NONE
net.netfilter.nf_log.11 = NONE
net.netfilter.nf_log.12 = NONE
net.netfilter.nf_log.2 = nfnetlink_log
net.netfilter.nf_log.3 = NONE
net.netfilter.nf_log.4 = NONE
net.netfilter.nf_log.5 = NONE
net.netfilter.nf_log.6 = NONE
net.netfilter.nf_log.7 = NONE
net.netfilter.nf_log.8 = NONE
net.netfilter.nf_log.9 = NONE
net.netfilter.nf_log_all_netns = 0
```

从上面的结果可以看出，在执行tcpdump之前，我们并没有加载nfnetlink_log模块，在执行完`tcpdump -D`之后，这个模块被自动加载了(不一定是-D参数，其它tcpdump跟踪指令也会触发模块的自动加载)，并且系统参数中对应的nf_log也被设置成nfnetlink_log:`net.netfilter.nf_log.2 = nfnetlink_log`. 从`tcpdump -D`中，我们可以看到它支持`2.nflog (Linux netfilter log (NFLOG) interface)` 这个nflog接口，我们可以通过这个接口来获取日志。

### tcpdump保存日志

首先我们执行`iptables -t raw -A PREROUTING  -p tcp --destination-port 52000  -j TRACE`开启跟踪。然后通过tcpdump将日志保存到一个pcap文件。
[cos74]

```shell
$ tcpdump -i 2  -w netfiltertest.pcap
tcpdump: listening on nflog, link-type NFLOG (Linux netfilter log messages), capture size 262144 bytes
```

在[cos72]上发起一个curl访问

```shell
$ curl 172.28.129.4:52000 
^C
```

在cos74上停止装包
[cos74]

```shell
$ tcpdump -i 2  -w netfiltertest.pcap
tcpdump: listening on nflog, link-type NFLOG (Linux netfilter log messages), capture size 262144 bytes
^C50 packets captured
50 packets received by filter
0 packets dropped by kernel
```

我们可以看到生成了50个数据包，这个pcap文件可以下载到本地，通过wireshark软件进行分析。
![](https://pic.1510.cf/2024/08/999b932d5332ab0cfa50c41ffff32b8e.png)
展开Linux Netfilter NFLOG，我们可以看到熟悉的prefix字段：
![](https://pic.1510.cf/2024/08/b409d39afe46a7ee29f0958b7f62a8ce.png)
右击prefix，选择【应用为列】
![](https://pic.1510.cf/2024/08/6beba3b96fcc25d86f2becb8ce1471fd.png)
这样就可以把这个字段展示在列表里：
![](https://pic.1510.cf/2024/08/714bd51dad3109f7855555b26bc6acec.png)
同样我们可以把TCP的option也展示在列表里
![](https://pic.1510.cf/2024/08/703151b42d98a9a139c62b0512b29c66.png)
通过这个视图，可以清楚的显示数据包在iptables中是怎么流转的。如果我们需要通过文本方式批量处理这个日志，也可以通过wireshark将它导出成csv文件。
![](https://pic.1510.cf/2024/08/212fef0b3df341b494b4518186517dc8.png)
导出结果如下:
![](https://pic.1510.cf/2024/08/6e63378fb14815319679f0c44bc5a3b8.png)
导出的文件同样可以作为iptables.log，然后用上文提及的脚本处理分析：
![](https://pic.1510.cf/2024/08/9827a8f65bf48984380ee08eb3c71dea.png)

![](https://pic.1510.cf/2024/08/ce44c79f3f63f3d37ee863c3bbfdc6b7.png)

## ulogd2的方式进行日志分析

除了tcpdump之外，我们也可以用ulogd2作为用户空间侧的日志应用来处理nfnetlink_log日志，ulogd2是netfilter官方提供的日志处理框架，但是centos7并没有预置它的安装包，因此我们需要从源码手工编译ulogd2的二进制程序。
ulogd2的编译涉及到各个依赖包的版本兼容性问题，处理起来比较费劲，以下是验证过的一种编译方式：

### ulogd2 编译安装

首先是编译环境的准备，主要是编译工具的安装和依赖的开发包安装。
[cos74]

```shell
$ yum install autoconf automake
$ yum install libmnl-devel jansson-devel libpcap-devel libnetfilter_conntrack-devel
```

编译libnetfilter_log模块：
[cos74]

```shell
$ git clone git://git.netfilter.org/libnetfilter_log
$ cd libnetfilter_log
$ ./autogen.sh
$ ./configure --prefix=/usr/local
$ make
$ sudo make install
```

编译libnetfilter_acct模块：
[cos74]

```shell
$ git clone git://git.netfilter.org/libnetfilter_acct
$ cd libnetfilter_acct
$ autoreconf -fi
$ ./configure --prefix=/usr/local
$ make
$ sudo make installl
```

编译ulogd2:
[cos74]

```shell
$ git clone git://git.netfilter.org/ulogd2 
$ ./autogen.sh
$ PKG_CONFIG_PATH=/usr/local/lib/pkgconfig  ./configure
$ make
$ sudo make install
```

### ulogd2的配置文件

对于我们上述选项编译出来的ulogd2，配置文件的位置是`/usr/local/etc/ulogd.conf`.

```properties
[global]
######################################################################
# GLOBAL OPTIONS
######################################################################


# logfile for status messages
logfile="/var/log/ulogd.log"

# loglevel: debug(1), info(3), notice(5), error(7) or fatal(8) (default 5)
# loglevel=1

plugin="/usr/local/lib/ulogd/ulogd_inppkt_NFLOG.so"
#plugin="/usr/local/lib/ulogd/ulogd_inppkt_ULOG.so"
#plugin="/usr/local/lib/ulogd/ulogd_inppkt_UNIXSOCK.so"
plugin="/usr/local/lib/ulogd/ulogd_inpflow_NFCT.so"
plugin="/usr/local/lib/ulogd/ulogd_filter_IFINDEX.so"
plugin="/usr/local/lib/ulogd/ulogd_filter_IP2STR.so"
#plugin="/usr/local/lib/ulogd/ulogd_filter_IP2BIN.so"
#plugin="/usr/local/lib/ulogd/ulogd_filter_IP2HBIN.so"
#plugin="/usr/local/lib/ulogd/ulogd_filter_PRINTPKT.so"
plugin="/usr/local/lib/ulogd/ulogd_filter_HWHDR.so"
plugin="/usr/local/lib/ulogd/ulogd_filter_PRINTFLOW.so"
#plugin="/usr/local/lib/ulogd/ulogd_filter_MARK.so"
plugin="/usr/local/lib/ulogd/ulogd_output_LOGEMU.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_SYSLOG.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_XML.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_SQLITE3.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_GPRINT.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_NACCT.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_PCAP.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_PGSQL.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_MYSQL.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_DBI.so"
plugin="/usr/local/lib/ulogd/ulogd_raw2packet_BASE.so"
#plugin="/usr/local/lib/ulogd/ulogd_inpflow_NFACCT.so"
#plugin="/usr/local/lib/ulogd/ulogd_output_GRAPHITE.so"
plugin="/usr/local/lib/ulogd/ulogd_output_JSON.so"


stack=log2:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,mac2str1:HWHDR,json1:JSON

[log2]
group=0 # Group has to be different from the one use in log1
numeric_label=1

[json1]
sync=1
device="cos74ulogd2"
boolean_label=1

```

ulogd2的配置文件中global段里配置如下字段
logfile: 输出ulogd2本身的日志信息
plugin: 默认的配置文件把所有的插件都注释了，对于在后续stack中需要用到的插件，需要取消注释。
stack: stack是ulogd2的实例配置，每个stack都以一个source插件开始，后面接多个Filter插件，以一个Output插件结束。配置文件中可以配置多个stack。对于示例中的stack：`stack=log2:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,mac2str1:HWHDR,json1:JSON`,NFLOG是source插件，JSON是output插件，中间的都是filter插件。插件的格式是 插件名:插件。比如log2:NFLOG这里的log2就是自定义的插件名，NFLOG则定义了它是什么插件。stack中的插件可以通过单独的插件名作为段名配置插件的参数。
比如[log2]里定义了这个插件的相关配置项。
我们可以通过如下命令查看插件的信息，包含插件支持的配置项
[cos74]

```java
$  ulogd -v -i /usr/local/lib/ulogd/ulogd_inppkt_NFLOG.so
Name: NFLOG
Config options:
        Var: bufsize (Integer, Default: 150000)
        Var: group (Integer, Default: 0)
        Var: unbind (Integer, Default: 1)
        Var: bind (Integer, Default: 0)
        Var: seq_local (Integer, Default: 0)
        Var: seq_global (Integer, Default: 0)
        Var: numeric_label (Integer, Default: 0)
        Var: netlink_socket_buffer_size (Integer, Default: 0)
        Var: netlink_socket_buffer_maxsize (Integer, Default: 0)
        Var: netlink_qthreshold (Integer, Default: 0)
        Var: netlink_qtimeout (Integer, Default: 0)
        Var: attach_conntrack (Integer, Default: 0)
Input keys:
        Input plugin, No keys
Output keys:
        Key: raw.mac (raw data)
        Key: raw.pkt (raw data)
        Key: raw.pktlen (unsigned int 32)
        Key: raw.pktcount (unsigned int 32)
        Key: oob.prefix (string)
        Key: oob.time.sec (unsigned int 32)
        Key: oob.time.usec (unsigned int 32)
        Key: oob.mark (unsigned int 32)
        Key: oob.ifindex_in (unsigned int 32)
        Key: oob.ifindex_out (unsigned int 32)
        Key: oob.hook (unsigned int 8)
        Key: raw.mac_len (unsigned int 16)
        Key: oob.seq.local (unsigned int 32)
        Key: oob.seq.global (unsigned int 32)
        Key: oob.family (unsigned int 8)
        Key: oob.protocol (unsigned int 16)
        Key: oob.uid (unsigned int 32)
        Key: oob.gid (unsigned int 32)
        Key: raw.label (unsigned int 8)
        Key: raw.type (unsigned int 16)
        Key: raw.mac.saddr (raw data)
        Key: raw.mac.addrlen (unsigned int 16)
        Key: raw (raw data)
        Key: ct (raw data)
```

### ulogd2启动

我们在启动前确保nfnetlink_log已经从内核中卸载，观察下ulogd2启动后能不能自动加载这个模块。
[cos74]

```shell
$ lsmod|grep nfnetlink_log

$ sysctl -a|grep nf_log   
net.netfilter.nf_log.0 = NONE
net.netfilter.nf_log.1 = NONE
net.netfilter.nf_log.10 = NONE
net.netfilter.nf_log.11 = NONE
net.netfilter.nf_log.12 = NONE
net.netfilter.nf_log.2 = NONE
net.netfilter.nf_log.3 = NONE
net.netfilter.nf_log.4 = NONE
net.netfilter.nf_log.5 = NONE
net.netfilter.nf_log.6 = NONE
net.netfilter.nf_log.7 = NONE
net.netfilter.nf_log.8 = NONE
net.netfilter.nf_log.9 = NONE
net.netfilter.nf_log_all_netns = 0
```

以后台方式启动ulogd并查看模块加载情况:
[cos74]

```shell
$ ulogd -d

$ lsmod|grep nfnetlink_log
nfnetlink_log          17892  1

$ sysctl -a|grep nf_log
net.netfilter.nf_log.0 = NONE
net.netfilter.nf_log.1 = NONE
net.netfilter.nf_log.10 = nfnetlink_log
net.netfilter.nf_log.11 = NONE
net.netfilter.nf_log.12 = NONE
net.netfilter.nf_log.2 = nfnetlink_log
net.netfilter.nf_log.3 = NONE
net.netfilter.nf_log.4 = NONE
net.netfilter.nf_log.5 = NONE
net.netfilter.nf_log.6 = NONE
net.netfilter.nf_log.7 = nfnetlink_log
net.netfilter.nf_log.8 = NONE
net.netfilter.nf_log.9 = NONE
net.netfilter.nf_log_all_netns = 0
```

我们可以看到ulogd2启动后，nfnetlink_log被自动加载，并且系统参数中 ipv4,ipv6和bridge都自动配置了nfnetlink_log。

### 日志查看

在配置好iptables TRACE指令之后，我们通过在cos72上执行curl可以触发相应的跟踪日志。在配置文件[json1]段里，我们并没有显式指定json模块输出日志文件的位置，我们可以通过插件的信息找到日志文件的默认位置：
[cos74]

```shell
$ ulogd -v -i /usr/local/lib/ulogd/ulogd_output_JSON.so
Name: JSON
Config options:
	Var: file (String, Default: /var/log/ulogd.json)
	Var: sync (Integer, Default: 0)
	Var: timestamp (Integer, Default: 1)
	Var: eventv1 (Integer, Default: 0)
	Var: device (String, Default: Netfilter)
	Var: boolean_label (Integer, Default: 0)
	Var: mode (String, Default: file)
	Var: host (String, Default: 127.0.0.1)
	Var: port (String, Default: 12345)
Input keys:
	No statically defined keys
Output keys:
	Output plugin, No keys
```

从上面输出的信息，我们可以了解到日志默认是输出到/var/log/ulogd.json。我们也可以在[json1]中指定file参数，将日志文件指向其它路径。
在/var/log/ulogd.json观察到如下日志:
[cos74]

```shell
$ cat ulogd.json
{"timestamp": "2023-01-05T11:04:26.620815+0800", "dvc": "cos74ulogd2", "raw.pktlen": 60, "raw.pktcount": 1, "oob.prefix": "TRACE: raw:PREROUTING:policy:3 ", "oob.time.sec": 1672887866, "oob.time.usec": 620815, "oob.mark": 0, "oob.ifindex_in": 2, "oob.hook": 0, "raw.mac_len": 14, "oob.family": 2, "oob.protocol": 2048, "action": "allowed", "raw.type": 1, "raw.mac.addrlen": 6, "ip.protocol": 6, "ip.tos": 0, "ip.ttl": 64, "ip.totlen": 60, "ip.ihl": 5, "ip.csum": 18892, "ip.id": 38576, "ip.fragoff": 16384, "src_port": 55768, "dest_port": 52000, "tcp.seq": 658789904, "tcp.ackseq": 0, "tcp.window": 29200, "tcp.offset": 0, "tcp.reserved": 0, "tcp.urg": 0, "tcp.ack": 0, "tcp.psh": 0, 
"tcp.rst": 0, "tcp.syn": 1, "tcp.fin": 0, "tcp.res1": 0, "tcp.res2": 0, "tcp.csum": 47472, "oob.in": "eth0", "oob.out": "", "src_ip": "172.28.129.2", "dest_ip": "172.28.129.4", "mac.saddr.str": "00:15:5d:12:e2:24", "mac.daddr.str": "00:15:5d:12:e2:28", "mac.str": "00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00"}
{"timestamp": "2023-01-05T11:04:26.620930+0800", "dvc": "cos74ulogd2", "raw.pktlen": 60, "raw.pktcount": 1, "oob.prefix": "TRACE: mangle:PREROUTING:rule:1 ", "oob.time.sec": 1672887866, "oob.time.usec": 620930, "oob.mark": 0, "oob.ifindex_in": 2, "oob.hook": 0, "raw.mac_len": 14, "oob.family": 2, "oob.protocol": 2048, "action": "allowed", "raw.type": 1, "raw.mac.addrlen": 6, "ip.protocol": 6, "ip.tos": 0, "ip.ttl": 64, "ip.totlen": 60, "ip.ihl": 5, "ip.csum": 18892, "ip.id": 38576, "ip.fragoff": 16384, "src_port": 55768, "dest_port": 52000, "tcp.seq": 658789904, "tcp.ackseq": 0, "tcp.window": 29200, "tcp.offset": 0, "tcp.reserved": 0, "tcp.urg": 0, "tcp.ack": 0, "tcp.psh": 0, "tcp.rst": 0, "tcp.syn": 1, "tcp.fin": 0, "tcp.res1": 0, "tcp.res2": 0, "tcp.csum": 47472, "oob.in": "eth0", "oob.out": "", "src_ip": "172.28.129.2", "dest_ip": "172.28.129.4", "mac.saddr.str": "00:15:5d:12:e2:24", "mac.daddr.str": "00:15:5d:12:e2:28", "mac.str": "00:15:5d:12:e2:28:00:15:5d:12:e2:24:08:00"}
...
```

跟踪日志以json方式输出，包含如下字段：
timestamp: 时戳
dvc: 配置文件中定义的设备名
raw.*: 数据包相关信息，比如type就是硬件类型信息。
oob.*: 带外信息，主要是一些非数据包自带的元数据。
action: 按照文档的意思应该是输出包的匹配规则，但是实际上只是输出input里的numeric_label，这个值为0，就是blocked，为1就是allowed,可能不具有参考意义
ip.*: ip头部相关信息
src_port/dest_port: 源、目标端口
tcp.*: TCP头部相关信息
src_ip/dest_ip: 源、目标ip地址
mac.*: 数据链路层信息

### 通过jq格式化日志信息

原始的json文件日志信息比较多，也不够直观，我们可以用jq命令把信息格式化一下。jq不是系统自带的命令，但是可以通过yum源直接安装。
[cos74]

```shell
$ yum install -y jq
$ jq '"time:"+ .timestamp+" source_ip:"+.src_ip+" dest_ip:"+.dest_ip+" src_port:"+(.src_port|tostring)+" dest_port"+(.dest_port|tostring)+" prefix:"+."oob.prefix"+" mark:"+(."oob.mark"|tostring)+" OPT ("+(."ip.id"|tostring)+.src_ip+.dest_ip+(."ip.csum"|tostring)+(."ip.fragoff"|tostring)+(.src_port|tostring)+(.dest_port|tostring)+(."tcp.seq"|tostring)+(."tcp.csum"|tostring)+")"' ulogd.json
"time:2023-01-05T11:04:26.620815+0800 source_ip:172.28.129.2 dest_ip:172.28.129.4 src_port:55768 dest_port52000 prefix:TRACE: raw:PREROUTING:policy:3  mark:0 OPT (38576172.28.129.2172.28.129.41889216384557685200065878990447472)"
"time:2023-01-05T11:04:26.620930+0800 source_ip:172.28.129.2 dest_ip:172.28.129.4 src_port:55768 dest_port52000 prefix:TRACE: mangle:PREROUTING:rule:1  mark:0 OPT (38576172.28.129.2172.28.129.41889216384557685200065878990447472)"
"time:2023-01-05T11:04:26.620963+0800 source_ip:172.28.129.2 dest_ip:172.28.129.4 src_port:55768 dest_port52000 prefix:TRACE: mangle:cali-PREROUTING:rule:3  mark:0 OPT (38576172.28.129.2172.28.129.41889216384557685200065878990447472)"
```

由于通过json输出的日志中没有tcp的OPT字段，我们手工构建一个伪OPT，以方便通过上文中的python脚本直接解码：
![](https://pic.1510.cf/2024/08/c3a759946157ab4eb552bcf3acfdaf6b.png)

# 附录

## iptables的表

iptables是用于检查，修改，转发，重定向或者丢弃 IP 数据包的，正如它名字中所带的table，主要由5张表组成：
mangle表主要用于包数据的修改，比如修改TOS字段或者其它类似字段。注意，我们不建议在这个表上做包过滤，或者NAT的功能。mangle表的支持如下目标：1. TOS 2.TTL 3.MARK 4.SECMARK 5.CONNSECMARK.
TOS：这个target是用于设置或修改数据包的 Type of Service 字段。这个字段在网络设备上不一定完全实现，因此不建议在广域网里设置它。
TTL：数据包的TTL字段，有些限制宽带接入设备数的宽带运营商会通过TTL字段判断你是否通过自己的路由器来实现设备网络共享，可以用这个功能来规避运营商的校验。
MARK:这个target比较常用，比如我们正文中k8s就通过这个功能给数据包打标，后面根据打标结果把对应的数据包丢弃；我们也可以将这个功能同ip rule配合起来，实现策略路由功能。
SECMARK和CONNSECMARK在08年后已经迁移到security表，mangle表可能还保留对这两个目标的支持，以满足兼容性。
注意，TOS和TTL只在mangle表有效，其它表不支持这两个目标。

NAT表只用于对不同的数据包做NAT(网络地址转换)。对于数据流来说只有第一个数据包会命中NAT表，其它的包会自动采用第一个包相同的动作。实际支持的目标有：DNAT/SNAT/MASQUERADE/REDIRECT.
DNAT:用于目标地址转换，常见的用法是把你的公网地址+端口映射到内部DMZ区的某个机器的私网地址+端口。
SNAT:用于源地址转换，典型的场景就是以路由器做公网出口，内网用户通过路由器上公网，ip地址被SNAT成路由器地址。
MASQUERADE:和SNAT一样，只不过它的目标不需要是固定的ip地址，可以根据网络接口自动获取到出口ip，适用于PPPoE这种动态获取公网地址的场景。

RAW表的设计目标是给数据包打一个NOTRACK的标记，这样这个数据包后续就不会走conntrack模块。因此RAW表在iptables内核处理时，处于靠前的位置。正是由于它的位置靠前，我们才把TRACE规则放在RAW表，确保数据包能在进入其它规则前就打好跟踪的标记。我们不建议在RAW表上配置包过滤等其它目的的规则。

filter:用于包过滤，可以支持对数据包做DROP或者ACCEPT操作，是防火墙的主要功能。

security:主要是用于配合SELiunx做MAC(Mandatory Access Control,强制访问控制),用得比较少。

## iptables的链和规则

iptables中每个表都配置了若干条链。系统预置的链有PREROUTING，INPUT，FORWARD，OUTPUT和POSTROUTING。每条链中又配置了若干条规则，具体某一条规则则是以一到多个匹配条件加上一个目标(target)构成。
除了系统预置的链，iptables还可以有自定义链，自定义链必须从预置链的某个规则中通过-J 自定义链名 的方式跳转进去，自定义链不能定义默认的策略，所以，如果自定义链的每个规则都没有命中，会跳转回原来的链。

## iptables数据包流转图

![](https://pic.1510.cf/2024/08/27894a7b4c45258c003ec96b62e326bf.png)
上图源于wikimedia：
[File:Netfilter-packet-flow.svg - Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Netfilter-packet-flow.svg)
iptables处理流程
这个流程图基本涵盖了数据包进入内核后的各个处理流程，我们这边主要关注的是ipv4层面的数据处理流程，也就是Network Layer以上的那部分。从图中我们可以看到Netfiler在内核中有5个钩子（HOOK），分别是:

### NF_IP_PRE_ROUTING

这是入方向的数据包进入到网络栈后最早触发的钩子。它作用在路由决策之前。

### NF_IP_LOCAL_IN

这个钩子在入方向的数据包通过路由决策之后执行，如果数据包被路由到本机，就会触发这个钩子。

### NF_IP_FORWARD

这个钩子也是在入方向的数据包经过路由决策后执行，如果数据包被路由到其它主机，会触发这个钩子。

### NF_IP_LOCAL_OUT

本地产生的外出数据包在进入网络栈后会最先触发这个钩子。

### NF_IP_POST_ROUTING

这个钩子触发的时机是外出或者转发的数据包在通过路由决策之后，经过这个钩子之后，数据包就会被发送到网卡上。

这里的Hook和各个表里系统预置的链有一定的关系。每个系统预置的链都会到Hook里注册函数，比如上图中，raw表，mangle表和nat表都会到NF_IP_PRE_ROUTING这个hook上注册一个函数，每个函数都带有优先级字段，Linux内核中定义了如下字段的优先级，优先级越小，执行时越靠前。

```go
enum nf_ip_hook_priorities {
	NF_IP_PRI_FIRST = INT_MIN，
	NF_IP_PRI_CONNTRACK_DEFRAG = -400,
	NF_IP_PRI_RAW = -300,
	NF_IP_PRI_SELINUX_FIRST = -225,
	NF_IP_PRI_CONNTRACK = -200,
	NF_IP_PRI_MANGLE = -150,
	NF_IP_PRI_NAT_DST = -100,
	NF_IP_PRI_FILTER = 0,
	NF_IP_PRI_SECURITY = 50,
	NF_IP_PRI_NAT_SRC = 100,
	NF_IP_PRI_SELINUX_LAST = 225,
	NF_IP_PRI_CONNTRACK_HELPER = 300,
	NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,
	NF_IP_PRI_LAST = INT_MAX,
};

```

NF_IP_PRI_RAW 是raw表的优先级，NF_IP_PRI_MANGLE 是mangle表的优先级，NF_IP_PRI_NAT_DST是NAT表目标地址转换的优先级，NF_IP_PRI_FILTER是filter表的优先级，NF_IP_PRI_SECURITY是security表的优先级，NF_IP_PRI_NAT_SRC是NAT表源地址转换的优先级。从这个优先级顺序可以看出raw表的优先级是最高的。

nat表则比较特殊，它分为DNAT和SNAT两种，这两个功能有不同的优先级，DNAT的优先级比filter表高，而SNAT的优先级比filter表要低。

DNAT的挂载点都在路由决策之前，这样DNAT转化后的地址才能作为路由参考，因此DNAT的挂载点主要在NF_IP_PRE_ROUTING和NF_IP_LOCAL_OUT（对应PREROUTING和LOCAL链），NF_IP_PRE_ROUTING这个hook挂载的函数主要是处理入口流量的DNAT，而NF_IP_LOCAL_OUT则是处理本机发出的数据包的DNAT。SNAT一般在路由决策之后修改，我们通常在NF_IP_POST_ROUTING里处理它，也有比较特殊的场景会需要在NF_IP_LOCAL_IN hook，也就是Local链里处理SNAT。Local 链处理SNAT的一个案例就是在你有相同的五元组（源IP，源端口，目标IP，目标端口，协议）数据包需要根据网卡区分不同处理的路径时，本机作为执行NAT的目标机器时要通过Local链做SNAT。

[kernel/git/torvalds/linux.git - Linux kernel source tree](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit?id=c68cd6cc21eb329c47ff020ff7412bf58176984e)

我们前面说了，DNAT和SNAT具有不同的优先级，因此wikimedia上的这个数据流转图在INPUT链上的顺序是不对的，INPUT上是SNAT，而SNAT的优先级是低于filter的。

![](https://pic.1510.cf/2024/08/dcb6fd694dc1b6e5b073b8a8d65b1dae.png)

我们从正文中正常响应的trace的日志也可以看出来：

![](https://pic.1510.cf/2024/08/4c7448b41290ac47b77d82fc5637c8a6.png)

Input链是先执行filter表，然后再执行那个nat表的。当然我们这个环境上nat表的INPUT链是空，在num为1的位置直接就命中了INPUT链的默认策略。