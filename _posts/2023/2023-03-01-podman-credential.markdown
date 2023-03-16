---
layout: post
title:  "Podman仓库鉴权文件"
date:   2023-03-01 16:01:00 +0800
category: Container
---
podman login仓库后，默认会将鉴权的文件生成到`${XDG_RUNTIME_DIR}/containers/auth.json` 目录下，也就是
`/run/containers/0/auth.json`.
这个文件里的密码信息是以加密方式存储的.由于/run目录具有临时性特点，主机重启后我们就需要重新登录到仓库。如果需要做到类似免密登录的效果，可以通过REGISTRY_AUTH_FILE这个变量将文件指向一个持久化的地方。
当然这个操作是典型的以牺牲安全性来实现系统便利操作，不建议在生产系统使用。
