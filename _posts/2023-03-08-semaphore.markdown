---
layout: post
title:  "Semaphore简介"
date:   2023-03-07 14 16:22:00 +0800
categories: jekyll update
---
Semaphore是一个ansible GUI系统，它可以提供一个可视化的界面，用于执行playbook，查看任务执行状态，并且提供简单的认证授权功能。除了GUI功能之外，它还提供一套API，可以通过API来控制任务执行，方便我们开发扩展的应用或者和已有的管控平台对接。
# 安装配置
Semaphore提供了Ubuntu下的snap安装功能，还有deb，rpm等预置的安装包，也可以通过docker启动，甚至可以直接通过二进制部署。本文以docker部署方式为例，提供一个安装demo：
```
docker run  -d  --name=semaphore -p 3000:3000 -e SEMAPHORE_DB_DIALECT=bolt -e SEMAPHORE_ADMIN_PASSWORD=fortest -e SEMAPHORE_ADMIN_NAME=admin -e  SEMAPHORE_ADMIN_EMAIL=admin@localhost -e SEMAPHORE_ADMIN=admin -v /root/containers/semaphore/volumes/etc:/etc/semaphore -v /root/containers/semaphore/volumes/lib:/var/lib/semaphore semaphoreui/semaphore:latest 
```
这里我们用boltdb作为SEMAPHORE_DB_DIALECT的参数，用于数据持久化。除了bolt之外，Sempaphore还支持mysql和postgreSQL作为后端数据库，生产环境可以考虑用mysql或postgreSQL的高可用集群作为后端。

Semaphore支持对接LDAP，并可以通过邮件，telegram，slack等方式触发告警。

Semaphore有个concurrency_mode参数，默认情况下不允许并发执行任务，这个参数可以设置成project或者node。当参数设置成project时，如果task之间是属于不同的project_id时，它们是可以并发执行的，不会校验节点之间是否有相互影响。如果参数设置成node，则仅当这些节点间互不影响时才能并发执行。

# 入门指南

## 门户界面

打开系统登录界面：

![](https://f.003721.xyz/2023/03/ce2d725a29b12b0dbb389d7d6a4d6d79.png)

登录之后，可以直接创建一个项目：

![](https://f.003721.xyz/2023/03/539fa10e5b9cb3722ab16343d5c5dfec.png)

主界面如下所示：

![](https://f.003721.xyz/2023/03/1967db6afba393802e4a86668fea4ab9.png)

Dashboard页面展示的是任务的执行历史。

Task Templates页面用于配置task，build，和deploy等任务。

Inventory页面用于配置playbook的Inventory信息。

Environment页面用于环境变量的相关配置。

Key Store页面用于配置密码或者密钥信息，这些信息可以用于Inventory里，也可以用在Repositories里。

Repositories页面用于配置ansible playbook存放的位置。

Team页面主要是维护哪些用户可以使用本项目。

除了这些主页面，界面还有一个Dark Mode单选框，选中后可以切换到Dark模式。

![](https://f.003721.xyz/2023/03/98d615a553994340358b3c3ced1bd200.png)


点击最底下的用户名

![](https://f.003721.xyz/2023/03/e28e2f0c94eeaa39f418eb8df815ecb9.png)

可以进行用户相关的管理：

![](https://f.003721.xyz/2023/03/ceb69ee5e86bf329945e73681f6946b0.png)

可以新增用户，修改用户，删除用户:

![](https://f.003721.xyz/2023/03/20d2000fad46a01f70859e337bfa812b.png)

## 入门任务

我们从一个全新的环境开始配置练手任务。
### 免密配置

在配置Inventory之前，我们先做下Semaphore到各个被控节点的公钥免密登录。首先，我们在Semaphore容器上生成一对密钥。

```
$ docker exec -ti semaphore sh
~ $ cd /var/lib/semaphore/
/var/lib/semaphore $ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/semaphore/.ssh/id_rsa): /var/lib/semaphore/id_rsa
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /var/lib/semaphore/id_rsa
Your public key has been saved in /var/lib/semaphore/id_rsa.pub
The key fingerprint is:
SHA256:8SgiC85Put7vbE5rLCaSsrmv/E5L9YgBpM0Rlg6zwiI semaphore@dc8b31b39871
The key's randomart image is:
+---[RSA 3072]----+
| .+o             |
|=+..             |
|o*o     .        |
|E.o      +       |
|= ..... S .      |
|o. o+.o.         |
| +.=.o .         |
|=oBo++.          |
|OXB*B*           |
+----[SHA256]-----+
/var/lib/semaphore $ ls -ltr
total 56
-rw-r--r--    1 semaphor root         65536 Mar  7 11:56 database.boltdb
-rw-r--r--    1 semaphor root           576 Mar  7 12:10 id_rsa.pub
-rw-------    1 semaphor root          2610 Mar  7 12:10 id_rsa
```
再把id_rsa.pub的内容加到各个被控节点的.ssh/authorized_keys文件中。

```
vi $HOME/.ssh/authorized_keys
```

![](https://f.003721.xyz/2023/03/9c00fedd7834859a225a06f9de6fb5e8.png)

从Semaphore容器里尝试登录到被控节点:
`ssh -i /var/lib/semaphore/id_rsa testuser@192.168.xxx.xxx` ,一方面可以验证下免密登录是否生效，另一方面可以避免后续ansible到各个客户端执行命令时碰到指纹验证问题。

### Key Store

完成所有节点的免密登录后，我们可以回到Semaphore界面的配置上。

在配置Inventory之前，我们先到Key Store里配置一个ssh的私钥，把我们上面创建的id_rsa托管在里面。

![](https://f.003721.xyz/2023/03/aea58405b3619e3186ec68eb43508e2c.png)

点击[New Key]创建一个SSH Key：

![](https://f.003721.xyz/2023/03/aa5ffa8b7caee02baefa4042c7258559.png)

![](https://f.003721.xyz/2023/03/7d7d8381666f0d6b7b1818903bcc7e58.png)


### Inventory

然后进入Inventory页面：

![](https://f.003721.xyz/2023/03/2e0dd712c3b608612a84114b94619ce5.png)

点击Inventory界面上的[New Inventory]按钮：

![](https://f.003721.xyz/2023/03/c653675df2609dae9fe6b01f0cbcae90.png)

这里可以选择Inventory的类型有Static, Static YAML和File。Static和Static YAML的用法类似，只是文件格式不同，具体可以查看下ansible官网文档:
[inventory_guide](https://docs.ansible.com/ansible/latest/inventory_guide/index.html),File类型需要和Repositories搭配使用，指定本地存储或git上的文件做为inventory文件。

此处我们选择Static Yaml。

![](https://f.003721.xyz/2023/03/6fd3981da8c6fb374a08d42d792cae42.png)

底下有示例，可以参照配置Inventory.

![](https://f.003721.xyz/2023/03/e035a0d55e2500fa689204ec39d48597.png)

User Credentials选择我们前面创建的SSH key。


### Environment

作为一个新手任务Environment我们不做过多配置，暂时都置空。

![](https://f.003721.xyz/2023/03/36f2ba311e1487dc6450e3b452b635c8.png)

这里有个坑，界面上给出的样例里1245是个数值类型，这种写法不符合json要求，会导致后续task执行失败，需要写成字符串形式。

### Repositories

我们创建一个本地存储类型的repository。所有的Repositories都需要配置AccessKey，对于本地存储，这个key就是一个空值。我们需要在Key Store中增加一个None类型的key来占位。

![](https://f.003721.xyz/2023/03/9e855b7f0127e84022461305dc676308.png)

打开Repositories页面：

![](https://f.003721.xyz/2023/03/c4d8fc30aa4e4b6298d2ed3b135c8166.png)

创建一个local类似的存储，key指定为None Key。

![](https://f.003721.xyz/2023/03/c8dd123ee78f4393a8cfee05803ccfd0.png)

### playbook 文件生成

我们可以让必应生成一个playbook样例

![](https://f.003721.xyz/2023/03/abe578fdfd4232cf53bcf2a55107f284.png)

然后将这个样例放到容器目录里，由于我们的容器是以卷组方式挂到宿主机，因此可以直接在宿主机上操作。
```
xxxx@xxxxx[/home/xxxx/semaphore/volumes/lib]$ sudo mkdir repo
xxxx@xxxxx[/home/xxxx/semaphore/volumes/lib]$ cd repo/
xxxx@xxxxx[/home/xxxx/semaphore/volumes/lib/repo]$ sudo vi diskusage.yml
xxxx@xxxxx[/home/xxxx/semaphore/volumes/lib]$ sudo chown -R 1001 repo/
```
### 配置task

在Task Templates页面上点击[New Template]

![](https://f.003721.xyz/2023/03/13614d18ccf02b7546ef0331c6f7dd5d.png)

Playbook Filename 里填写相对路径，其它选项用我们前面创建好的Inventory，repository和Env等：

![](https://f.003721.xyz/2023/03/e8116515eb7ca6e636234a1abd331891.png)

点击[Run],触发测试:

![](https://f.003721.xyz/2023/03/37b29135a1101c6a7519f5227aaebb9a.png)

![](https://f.003721.xyz/2023/03/3dcfe65d1453b84e3808ba6efd7b2279.png)

成功执行：

![](https://f.003721.xyz/2023/03/16add0b4e620f8498d2daf74f8ab9d93.png)

# 进阶配置
## git仓库

我们在入门配置中playbook是放在本地存储上的，这种方式对于配置的更新可能不是那么友好。Repositories除了本地存储之外，还支持git方式管理文件。
如果我们用的是私有的仓库，可以用ssh方式clone仓库，这就要求我们先在git系统上将ssh key配置好。

首先是ssh key的生成。Semaphore容器不支持通过rsa的key从仓库里拉playbook。 如果只能用rsa的key，需要在容器里的 `~/.ssh/config`里增加如下内容：

```
Host git.zzzz.com
PubkeyAcceptedKeyTypes=+ssh-rsa
```

这里的git.zzzz.com就是指你的git系统域名。

建议用ed25519类型的key，安全性较好：
```
$ ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/cloud/.ssh/id_ed25519): 
```
将生成的key私钥录入到Semaphore的 Key Store里，此处就不一一赘述，然后将公钥关联到git账号上，一般是点击Git UI的右上角图标，选择Settings。

![](https://f.003721.xyz/2023/03/fa4083c4f9e27b9451ba8aa187e2760a.png)

选择SSH/GPG keys，并在里面录入公钥:

![](https://f.003721.xyz/2023/03/d7fe3cb81f0b096e42e8553638cfe8a6.png)

Semaphore端Repositories的配置参考下图：

![](https://f.003721.xyz/2023/03/97634b900b79678feeb68351b0fb88a4.png)

## 定时执行
Semaphore支持定时触发playbook的执行，可以配置下图中的Cron选项：
![](https://f.003721.xyz/2023/03/f47bcd75756b28f541c79bb71716626e.png)

点击Cron右侧的文字，还支持通过git的提交配合Cron来触发任务的执行。

## 朴素的CICD

Semaphore 除了支持上面演示的简单task之外，还有build和deploy功能，可以用来搭建一个朴素的CICD流程。

![](https://f.003721.xyz/2023/03/cc9ac4f09656ec1dbfa779bb6780a4bf.png)

如上图所示，build流程和task流程基本一致，只是多了一个Start Version参数，用于指定起始的版本号，这个版本号会随着build 任务的执行自增。Build流程里的task可以获取如下变量信息:
```
semaphore_vars:
    task_details:
        type: build
        username: user123
        message: New version of some feature
        target_version: 1.5.33
```
我们可以在制品中将上述变量引入到制品名称里，比如：`semaphore_vars.task_details.target_version`这个变量。

Deploy的流程配置界面如下:

![](https://f.003721.xyz/2023/03/17de89fe09fdb7a909d2ceb62f2c28c5.png)

Deloy流程需要关联一个Build流程，playbook里可以获取到如下变量：
```
semaphore_vars:
    task_details:
        type: deploy
        username: user123
        message: Deploy new feature to servers
        incoming_version: 1.5.33
```
Build流程编译出制品后，会把版本号传递到deploy的`semaphore_vars.task_details.incoming_version`。Deploy流程里还有一个Autorun复选框，选中后可以在对应的build流程执行完成后自动触发deploy。