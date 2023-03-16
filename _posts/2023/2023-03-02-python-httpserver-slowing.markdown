---
layout: post
title:  "Python HttpServer启动缓慢问题"
date:   2023-03-02 14:54:00 +0800
categories: Linux
---
前段时间在做一个demo时，在一个虚拟机上用python自带的httpserver模块启动了简单的web服务端。启动的指令如下:
```
python3 -m http.server 7788
```
执行之后，发现指令不如预想的那样立马就把http服务拉起来，需要等个几秒到十几秒左右才能看到"Serving HTTP on xxxx"提示。当时以为是虚拟机的配置太差导致的，虚拟机是在一台老旧windows上拉的HyperV虚机。

今天又试了另外一台机器，发现这个指令同样也很慢，于是稍微挖了挖原因。
首先，尝试绑定localhost去起服务，发现速度非常快。
```
python3 -m http.server --bind 127.0.0.1 7788
```
敲完指令里面就得到响应。
接着绑定了本机的局域网ip，发现也很快。

紧接着绑定了0.0.0.0，这回变慢了。
通过import sys;sys.path找到python库文件目录后，对http的server设置了pdb的trace，跟踪到慢的点是调用系统_socket的gethostbyaddr慢。
```
hostname, aliases, ipaddrs = gethostbyaddr(name)
```
这种问题一般是由于主机名解析或者反向解析引起的，检查/etc/hosts文件，发现没有主机名对应的ip记录。

在/etc/hosts里127.0.0.1 后面的一串主机名中，加上了hostname指令返回的主机名，再次测试python的httpserver，这回响应正常了。
