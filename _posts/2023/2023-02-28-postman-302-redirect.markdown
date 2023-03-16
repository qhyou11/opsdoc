---
layout: post
title:  "Postman禁用302自动跳转"
date:   2023-02-28 13:26:35 +0800
categories: Dev
---
最近在调试一个api时，碰到一个奇怪的事情。我从浏览器F12的网络标签页拷贝了一个bash格式的请求，这个请求在linux机器上返回的是302，但是把请求录入到postman之后，点击测试，得到的结果却是400.
经过一番折腾后，发现postman有如下配置：

![](https://f.003721.xyz/2023/02/23048c1bb7f88ae337fa343cae1c9a3e.png)

![](https://f.003721.xyz/2023/02/7b646aba8d5ffecf9a05c1064efaa879.png)

[Automatically follow redirects] 这个选项会导致postman自动获取302返回响应消息中的Location，并代替我们向这个location发起请求。
取消这个勾选后，我们可以获得和linux上手工curl一样的结果。
这个参数也可以设置在每个请求里：

![](https://f.003721.xyz/2023/02/f3b4f57758a86ebf9f0f44cd8572d7cf.png)

PS: curl 语句加上-L 选项也会自动触发对302的跳转。

