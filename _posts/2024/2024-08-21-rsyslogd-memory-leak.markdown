---
layout: post
title:  "rsyslogd内存占用高导致系统异常"
date:   2024-08-21 12:00:14 +0800
category: Linux
---
# 问题描述
用户反馈某环境k8s应用详情无法展示。应用管控页面显示系统查询k8s API 返回结果异常。

# 问题分析

由于这个环境比较异常，整个环境处于云端，而我们的应用管控平台是部署在内网，经常有网络不通的问题，因此首先排查的就是网络连通性问题。经过简单的ping，可以确认应用管控平台到master节点的网络连接是正常的。

问题就应该出在云上Master节点的服务上。我们尝试登录主机，发现登录过程有些卡。进入系统后，执行top，发现CPU的idle值很小，同时有百分之九十几的wait。进一步按照内存对应用排序，发现rsyslogd进程占用了系统百分之七十几的内存;检查系统的内存使用情况，发现可用值很低。

# 问题解决
执行如下指令，释放内存占用：

```bash
systemctl restart rsyslog
```
执行完成后，free显示内存可用值增加到总内存的70%左右。应用性能问题也随之解决。


# 关联问题处理

为了排查rsyslogd内存占用高的原因，我们从/etc/rsyslog.conf入手，从配置文件可以看到，日志是输出到/var/log/message文件。我们检查了/var/log下的message文件，发现message文件有30G，而上一个message文件是一个月前的。这个日志的保留策略是由logrotate实现的，这个进程有cron.daily负责定时执行，相关的配置在/etc/logrotate.conf 和/etc/logrotate.d下面。针对rsyslog，我们配置的是每周做日志滚动一次，保留4个备份。因此，这个日志滚动已经失败了好几个星期。

日志量过大的问题是和kubelet的日志级别设置有关，日志级别设置太低，导致相关应用输出大量日志，从而引发日志滚动失败和rsyslogd内存占用异常。

处理措施：
1. 调整kubelet上日志级别
2. 在systemd上对rsyslogd做内存限制保护：配置MemoryAccounting，MemoryHigh和MemoryMax参数。
```bash
[Service] 
MemoryAccounting=yes 
MemoryMax=100M 
MemoryHigh=20M
```
