
---
layout: post
title:  "EdgeRouter路由器IPv6配置指南"
date:   2024-08-22 12:24:00 +0800
category: Ops
---

# 环境介绍

## 网络拓扑

我的家庭网络架构大致如下图所示，由两个CPE(Customer Premise Equipment，讲人话就是光猫)，一个有线路由器(UBNT ER-X),两个无线路由器组成。两个CPE分别由电信和移动提供，电信的是烽火AN5006-02-A ，移动的是中兴ZXHN F663N。

![](https://f.003721.xyz/2024/08/8b7977c34c7115f377e39d039a52d2ef.png)

电信的CPE安装时让装维师傅开通了IPv4公网权限，同时做了桥接，也就是它本身不发起PPPoE拨号，需要下级路由器实现。电信CPE有两个网口，一个是千兆网口，一个是百兆网口。百兆网口接在一个无线路由器上，通过这个无线路由器拨号上网，提供稳定的网络给家里各个手机，平板上网。电信电信宽带签约速率时100M，因此这个百兆网口理论上是足够了，为了不影响家人上网，引发投诉，本次改造不涉及这个无线路由器的变更。千兆网口接在UBNT有线千兆路由器的eth0上，是我们本次测试的上联链路之一。
移动的CPE由设备自行拨号，我们通过DHCP上网，这个设备由一个千兆网口，四个十兆网口（是的，没错，在2023年还能见到10M的口子感觉有些魔幻）。移动的千兆口接在UBNT千兆路由器的eth2上。
我们这次测试的主要战场就是UBNT路由器和连接在UBNT路由器eth1和eth3的PC。

## UBNT简介

我们再来看看主路由UBNT ER-X(EdgeRouter)的配置。UBNT是著名跨国企业Ubiquiti公司的简称，这个ER-X是一个5口千兆路由器。UBNT的硬件设备都是在中国生产，因此这个小黑盒子上标注着“中国制造”。设备的操作系统是EdgeOS，EdgeOS是一个闭源商业版本，官方宣称是为了满足FCC的监管需求，无法开源。它是基于之前开源的路由系统Vyatta深度定制的系统。Vyatta是基于Debian开发的，在被博科收购之后这个开源版本已经不再演进。网上目前活跃的开源版本是从Vyatta fork出来的vyos。
这个小黑盒子具有硬件NAT能力，还是Linux系统，可玩性比较高。根据个人几年来的使用经验，它的性能表现也是不错的，遇到的比较大的硬件缺陷就是散热，一到夏天铁盒外壳就发烫，时不时断流，后来花了几块钱买了个USB风扇架在下面，问题解决。

## UBNT内部网络架构

ER-X这个UBNT路由器在初始化时，选择了Basic Setup模式，把eth0作为Internet Port，LAN Ports是eth1-4。从内部实现上LAN Ports上的接口都落在Switch0上。为了方便此次验证，我把eth1从LAN Ports中移出，然后新建了一个bridge接口br0，把eth0和eth1加入到这个br0上。这种改造的目的主要是想让PC能够绕过UBNT路由器直接进行拨号。从上面的网络拓扑可以看到，PC上有一个网卡接到eth1上，这个网卡和eth0（也就是电信CPE）在一个二层网络上，因此可以通过它来进行PPPoE拨号。这其实是一种妥协的方式，CPE上没有多余的网口可以直连PC，只能从主路由想法子。Br0的模式是逻辑的网桥，速度上比硬件实现的Switch0要慢不少，官方并不建议采用bridge，推荐的模式是采用外接的交换机来实现。我们这里只做些IPv6的验证，对性能要求不高，出于成本上的考量，就不引入多余硬件了。

# IPv6配置

## 向导模式配置

声明：向导模式我没有实际测试过，只是理论上可行。
打开UBNT路由器的管理页面，比如https://192.168.3.1 ,输入用户名密码登录到系统。
![](https://f.003721.xyz/2024/08/74ad505efe219f939c3b5afd80a7a776.png)

这里的管理页面地址和用户名密码取决于路由器初始化的配置，请参照官方说明文档配置。
[EdgeRouter X ER-X Quick Start Guide (ubnt.com)](https://dl.ubnt.com/guides/edgemax/EdgeRouter_ER-X_QSG.pdf)
点击Wizards，选择Basic Setup，按照下图设置，即可完成IPv6配置：
![](https://f.003721.xyz/2024/08/058c9a4d9ceba1f058ec8fbaf426f882.png)

LAN部分可以根据个人网络需求配置。
这里需要重点关注的是DHCPv6 PD部分，这个配置是路由器从PPPoE接口发起前缀委派请求。这里的Prefix length需要根据运营商实际下发的PD来确认，我这边两个运营商都是/60。
如何确认运营商分配的pd到底是多少呢？如果你是通过光猫（CPE）拨号，可以通过光猫的管理页面登录到系统查看，光猫设备上一般有提供用户名（user或useradmin）密码：
![](https://f.003721.xyz/2024/08/d31fd35a883e6c69e39a0f0beff5a6b5.png)

当然如果你是通过光猫拨号，也就没有必要在有线路由器再次拨号了。如果你是自行拨号，有些路由器系统，比如OpenWRT一脉，会将PD信息展示在接口上。如果是UBNT这种路由器，就只能一次次试验了，从64开始，看看最小能配多少。

## 手工配置

手工配置IPv6有两种方式，一种是通过界面配置树，另外一种是通过命令行指令。

### 配置树

点击Config Tree，依次展开Interfaces，Bridge，br0，在pppoe下新增index为0的配置，输入基本的pppoe连接信息：
![](https://f.003721.xyz/2024/08/c7a903f0c0e437ce61097821da0b0ac7.png)

这部分因人而异，常规情况下，应该配置的是Interfaces-Ethernet-eth0-pppoe。前面说过，我这边的环境略微特殊，把eth0和eth1加到br0里了，因此，pppoe需要在br0里配置。
在pppoe-0下面需要点击ipv6后面的+号，配置相关选项启用ipv6.
![](https://f.003721.xyz/2024/08/d5506447dd1a91918d2be4aed0d6e38b.png)

firewall里启用防火墙：
![](https://f.003721.xyz/2024/08/96a295da881c2bfbf2cabff33f9aa8b6.png)

![](https://f.003721.xyz/2024/08/823d7af25babda39d5e3fdfc03b93a7a.png)

重点在pd的配置，新增了pd0，prefix-length设置为/60：
![](https://f.003721.xyz/2024/08/2cba5ebf57fb74686f12324199cd2f09.png)

pd下面有个prefix-only，网上有些文档是要求打开，我这边打开配置后反而有问题，因此这里没有设置该参数。
下行接口配置的是硬件switch0：
![](https://f.003721.xyz/2024/08/c2fb55e043115335706457ce342651e1.png)

host-address和prefix-id可以自行配置，service选slaac。
![](https://f.003721.xyz/2024/08/a0bdfddad7362f6c1e1ef466b9c8f4b1.png)

prefix-id是有范围的，这个值其实是《IPv6基础》里提到的subnet-id.如果你的运营商下发的pd是/60,可选的值就是0-15.
LAN，也就是switch0需要开启路由通告：
![](https://f.003721.xyz/2024/08/687e31621ddcefca0e63f4407df0d04e.png)

prefix为::/64
![](https://f.003721.xyz/2024/08/9ac71f1f413af0987f8b86f9b6ae4eeb.png)

需要设置有效生存时间等参数：
![](https://f.003721.xyz/2024/08/8c239d962b064fe77215b7f94cb3f20b.png)

至此IPv6相关的配置基本完成，遗留一个防火墙规则的配置，在后续章节通过指令方式提供，可以参照配置。

#### 一个花絮

经过配置后，发现UBNT路由器没有触发DHCPv6-PD申请请求，switch0上没有分配IPv6地址，LAN中的客户端也无法获取IPv6地址。检查系统后发现，缺少了`/var/run/dhcp6c-pppoe0-pd.conf`文件。因此，我不得不手动创建该文件，并手动启动了wide-dhcpv6-client进程，问题得到解决。

尽管EdgeOS是闭源系统，但与系统配置相关的许多文件都是文本格式的，例如配置模板和Perl脚本。经过一番研（乱）究（翻），我发现`/opt/vyatta/share/perl5/Vyatta/Interface.pm`文件与PD配置的生成有很大关系。在该文件的第221行，原本是`} elsif ($intf =~ /(bridge\d+)/) {`，我将其更改为`} elsif ($intf =~ /(br\d+)/) {`，问题得到解决。

这可能是官方的笔误。通常情况下，大家将eth0或eth4作为WAN口，并在其中设置pppoe。而像我这种在br0中拨号并且还需要进行IPv6设置的情况比较少，所以很容易忽视这个错误。

### 命令行配置

命令行配置需要通过ssh登录到路由器后台，输入configure进入配置模式，然后执行如下指令：

```bash
set firewall ipv6-name WANv6_IN default-action drop
set firewall ipv6-name WANv6_IN description 'WAN inbound traffic forwarded to LAN'
set firewall ipv6-name WANv6_IN enable-default-log
set firewall ipv6-name WANv6_IN rule 10 action accept
set firewall ipv6-name WANv6_IN rule 10 description 'Allow established/related sessions'
set firewall ipv6-name WANv6_IN rule 10 state established enable
set firewall ipv6-name WANv6_IN rule 10 state related enable
set firewall ipv6-name WANv6_IN rule 15 action accept
set firewall ipv6-name WANv6_IN rule 15 destination port 22
set firewall ipv6-name WANv6_IN rule 15 protocol tcp
set firewall ipv6-name WANv6_IN rule 20 action drop
set firewall ipv6-name WANv6_IN rule 20 description 'Drop invalid state'
set firewall ipv6-name WANv6_IN rule 20 state invalid enable
set firewall ipv6-name WANv6_LOCAL default-action drop
set firewall ipv6-name WANv6_LOCAL description 'WAN inbound traffic to the router'
set firewall ipv6-name WANv6_LOCAL enable-default-log
set firewall ipv6-name WANv6_LOCAL rule 10 action accept
set firewall ipv6-name WANv6_LOCAL rule 10 description 'Allow established/related sessions'
set firewall ipv6-name WANv6_LOCAL rule 10 state established enable
set firewall ipv6-name WANv6_LOCAL rule 10 state related enable
set firewall ipv6-name WANv6_LOCAL rule 20 action drop
set firewall ipv6-name WANv6_LOCAL rule 20 description 'Drop invalid state'
set firewall ipv6-name WANv6_LOCAL rule 20 state invalid enable
set firewall ipv6-name WANv6_LOCAL rule 30 action accept
set firewall ipv6-name WANv6_LOCAL rule 30 description 'Allow IPv6 icmp'
set firewall ipv6-name WANv6_LOCAL rule 30 protocol ipv6-icmp
set firewall ipv6-name WANv6_LOCAL rule 40 action accept
set firewall ipv6-name WANv6_LOCAL rule 40 description 'allow dhcpv6'
set firewall ipv6-name WANv6_LOCAL rule 40 destination port 546
set firewall ipv6-name WANv6_LOCAL rule 40 protocol udp        
set firewall ipv6-name WANv6_LOCAL rule 40 source port 547
set interfaces bridge br0 pppoe 0 dhcpv6-pd pd 0 interface switch0 host-address '::1'
set interfaces bridge br0 pppoe 0 dhcpv6-pd pd 0 interface switch0 prefix-id ':1'
set interfaces bridge br0 pppoe 0 dhcpv6-pd pd 0 interface switch0 service slaac
set interfaces bridge br0 pppoe 0 dhcpv6-pd pd 0 prefix-length /60
set interfaces bridge br0 pppoe 0 dhcpv6-pd rapid-commit enable
set interfaces bridge br0 pppoe 0 firewall in ipv6-name WANv6_IN
set interfaces bridge br0 pppoe 0 firewall in name WAN_IN
set interfaces bridge br0 pppoe 0 firewall local ipv6-name WANv6_LOCAL
set interfaces bridge br0 pppoe 0 firewall local name WAN_LOCAL
set interfaces bridge br0 pppoe 0 ipv6 address autoconf
set interfaces bridge br0 pppoe 0 ipv6 dup-addr-detect-transmits 1
set interfaces bridge br0 pppoe 0 ipv6 enable
set interfaces switch switch0 ipv6 dup-addr-detect-transmits 1
set interfaces switch switch0 ipv6 router-advert cur-hop-limit 64
set interfaces switch switch0 ipv6 router-advert link-mtu 0
set interfaces switch switch0 ipv6 router-advert managed-flag false
set interfaces switch switch0 ipv6 router-advert max-interval 600
set interfaces switch switch0 ipv6 router-advert other-config-flag false
set interfaces switch switch0 ipv6 router-advert prefix '::/64' autonomous-flag true
set interfaces switch switch0 ipv6 router-advert prefix '::/64' on-link-flag true
set interfaces switch switch0 ipv6 router-advert prefix '::/64' valid-lifetime 691200
set interfaces switch switch0 ipv6 router-advert reachable-time 0
set interfaces switch switch0 ipv6 router-advert retrans-timer 0
set interfaces switch switch0 ipv6 router-advert send-advert true
```

# 系统实现

下面我们简单地对上文涉及的配置涉及的后台实现做个了解。

## WAN口IPv6配置

这里所说的WAN口，指的就是配置在br0上的pppoe0接口，这个接口是通过PPPoE拨号创建的。PPPoE已经是一个十分古老的协议，从ADSL上网开始就一直在用。现在运营商们都在升级改造，已经逐步推出IPoE的接入方式。
WAN口涉及IPv6部分主要是和`set interfaces bridge br0 pppoe 0 ipv6 *`相关的配置。这些配置在UBNT路由器中会通过一些模板生成对应的配置文件或配置指令。我们可以通过下面这个配置模板文件了解下相关配置会被实例化到哪些配置文件。
首先，模板会根据配置信息生成/etc/ppp/peers下的pppoe0配置文件：

```Shell
root@EdgeRouter-X:/opt/vyatta/share/vyatta-cfg/templates/interfaces/bridge/node.tag/pppoe# cat node.def 
#
# Configuration template for interface/ethernet/node.tag/pppoe
#
#
# Define a PPPOE interface.  The value of this node is the PPPOE unit
# number.  This number must be globally unique among all PPPOE units.
#

tag:
priority: 400

type: u32

help: PPPOE unit number

val_help: 0-15; Point-to-Point Protocol over Ethernet (PPPOE) unit number

syntax:expression:  ((($VAR(@) >= 0) && ($VAR(@) <= 15)) ; \
        "Only unit number must be between 0 and 15") && \
        ( exec "NOT_OURS=`grep -s ^#interface  /etc/ppp/peers/pppoe$VAR(@) | grep -v -c $VAR(../@)` ; \  
                if [ $NOT_OURS -eq 0 ]; then \
                        exit 0 ; \
                else \
                        exit 1 ; \
                fi " ; \
        "Unit number must be unique." )



#
# At create time, we initialize a new PPP provider
# configuration file with default values.  Other parameters will be inserted by
# the templates for those parameters.
#
create:
        ifname=pppoe$VAR(@)
        logfile=/var/log/vyatta/ppp_${ifname}.log
        sudo touch $logfile
        sudo chgrp adm $logfile
        sudo chmod 664 $logfile
        echo "`date`: PPP interface $ifname created" >> $logfile

        sudo rm -f /etc/ppp/peers/pppoe$VAR(@)
        sudo cp /opt/vyatta/etc/pppoe-provider-template \
           /etc/ppp/peers/pppoe$VAR(@)
    # interface is parsed by Interface.pm
    sudo sh -c "echo \#interface $VAR(../@) >> /etc/ppp/peers/pppoe$VAR(@)"
    sudo sh -c "echo plugin rp-pppoe.so >> /etc/ppp/peers/pppoe$VAR(@)"
    sudo sh -c "echo nic-$VAR(../@) >> /etc/ppp/peers/pppoe$VAR(@)"
        addon_v6addr=''
        for param in $VAR(./ipv6/address/secondary/@@); do
                addon_v6addr="$addon_v6addr,$param"
        done
        sudo sh -c "echo persist >> /etc/ppp/peers/pppoe$VAR(@)"
        sudo sh -c "echo mtu 1492  >> /etc/ppp/peers/pppoe$VAR(@)"
        sudo sh -c "echo mru 1492  >> /etc/ppp/peers/pppoe$VAR(@)"
    sudo sh -c "echo defaultroute  >> /etc/ppp/peers/pppoe$VAR(@)"
        sudo sh -c "echo usepeerdns  >> /etc/ppp/peers/pppoe$VAR(@)"
        sudo sh -c "echo ifname \\\"pppoe$VAR(@)\\\" >> /etc/ppp/peers/pppoe$VAR(@)"
        sudo sh -c "echo ipparam \\\"pppoe$VAR(@) ${addon_v6addr:1}\\\" >> /etc/ppp/peers/pppoe$VAR(@)"  
        sudo sh -c "echo debug  >> /etc/ppp/peers/pppoe$VAR(@)"
        sudo sh -c "echo logfile $logfile  >> /etc/ppp/peers/pppoe$VAR(@)"
#
# Delete means that this node and the tree below it has been deleted.
# We can delete the PPP provider configuration file.  The running PPPOE
# Daemon will be killed at "end:"
#
delete:
        ifname=pppoe$VAR(@)
        logfile=/var/log/vyatta/ppp_${ifname}.log
        echo "`date`: PPP interface $ifname deleted" >> $logfile
        sudo rm -f /etc/ppp/peers/pppoe$VAR(@)

#
# Three cases are handled here:
#  1) A new PPPOE configuration node has just been added.  Parameters may or
#     may not have also been added.
#  2) Some configuration parameters of a previously existing PPPOE node
#     have been changed.
#  3) A previously existing PPPOE configuration node has been deleted or disabled
#
# In case (1) we need to start the daemon for the first time.
# In case (2) we need to kill and restart the daemon.
# In case (3) we need to kill the daemon.
#
end:
        ifname=pppoe$VAR(@)
        logfile=/var/log/vyatta/ppp_${ifname}.log
        echo "`date`: Stopping PPP daemon for $ifname" >> $logfile

        sudo poff pppoe$VAR(@) | logger -p debug -t pppoe$VAR(@)-template

        if [ -e /etc/ppp/peers/pppoe$VAR(@) ]; then
                echo "`date`: Starting PPP daemon for $ifname" >> $logfile
                ( umask 0; sudo setsid sh -c 'nohup /usr/sbin/pppd \
                           call pppoe$VAR(@) > /tmp/pppoe$VAR(@).log 2>&1 &' )
        fi
```

如果配置中启用了ipv6，会在pppoe0这个配置文件上追加ipv6配置信息：

```shell
root@EdgeRouter-X:/opt/vyatta/share/vyatta-cfg/templates/interfaces/bridge/node.tag/pppoe/node.tag/ipv6/enable# cat node.def 
#
# Configuration template for the .../pppoe/node.tag/ipv6/enable parameter
#

help: Enable IPv6 address negotiation on the link

update: sudo sed -i '/^ipv6 /d' /etc/ppp/peers/pppoe$VAR(../../@)
        sudo sh -c "echo ipv6 $VAR(./local-identifier/@),$VAR(./remote-identifier/@) >> /etc/ppp/peers/pppoe$VAR(../../@)"

delete: sudo sed -i '/^ipv6 /d' /etc/ppp/peers/pppoe$VAR(../../@)
```

如果启用了地址自动配置，会对应触发一些配置指令，调整系统内核参数：

```Shell
root@EdgeRouter-X:/opt/vyatta/share/vyatta-cfg/templates/interfaces/bridge/node.tag/pppoe/node.tag/ipv6/address/autoconf# cat node.def 
#
# This is a valueless node, hence has no type associated with it.
#

help: Enable acquisition of IPv6 address using stateless autoconfig

update:
        sudo touch /var/run/vyatta/ipv6.autoconf.pppoe$VAR(../../../@)
        if [ -e /proc/sys/net/ipv6/conf/pppoe$VAR(../../../@)/autoconf ]; then
            echo "Enabling address auto-configuration for pppoe$VAR(../../../@)"
            sudo sh -c "echo 2 > /proc/sys/net/ipv6/conf/pppoe$VAR(../../../@)/accept_ra"
            sudo sh -c "echo 1 > /proc/sys/net/ipv6/conf/pppoe$VAR(../../../@)/autoconf"
            sudo /usr/bin/setsid /bin/rdisc6 -q "pppoe$VAR(../../../@)" 2>&1 > /dev/null &
        else
            echo "Address auto-configuration will be enabled when interface comes up."
        fi

delete:
        sudo rm -f /var/run/vyatta/ipv6.autoconf.pppoe$VAR(../../../@)
        if [ -e /proc/sys/net/ipv6/conf/pppoe$VAR(../../../@)/autoconf ]; then
            sudo sh -c "echo 1 > /proc/sys/net/ipv6/conf/pppoe$VAR(../../../@)/accept_ra"
            sudo sh -c "echo 0 > /proc/sys/net/ipv6/conf/pppoe$VAR(../../../@)/autoconf"
        else
            echo "Address auto-configuration will be disabled when interface comes up."
        fi

```

系统提供了一个手工启动pppoe的指令：connect interface pppoe0。我们可以通过这个指令大概了解下拨号过程中涉及到哪些应用。这个指令具体执行的内容可以通过下面这个文件查看：

```shell
root@EdgeRouter-X:/opt/vyatta/share/vyatta-op/templates/connect/interface/node.tag# cat node.def 
help: Bring up connection-oriented interface

allowed: local -a array ;
         array=( /etc/ppp/peers/pppoe* /etc/ppp/peers/pppoa* /etc/ppp/peers/wan* /etc/ppp/peers/wlm* /etc/ppp/peers/pptpc* ) ;
         echo  -n ${array[@]##*/}

run:
        IFNAME=${3}
        LOGFILE=/var/log/vyatta/ppp_${IFNAME}.log
        if [ ! -e /etc/ppp/peers/$IFNAME ]; then
                echo "Invalid interface: $3"
        elif [ -d /sys/class/net/$IFNAME ]; then
                echo "Interface $IFNAME is already connected."
        elif [ ! -z "`ps -C pppd -f | grep $IFNAME `" ]; then
                echo "Interface ${IFNAME}: Connection is being established."
        else
                echo "Bringing interface $IFNAME up..."
                sudo sh -c "echo \"`date`: User $USER starting PPP daemon for $IFNAME by connect command\" >> $LOGFILE"
                if [ "${IFNAME::3}" = "wan" ]; then
                    # Serial interfaces are started with "pon"
                    (umask 0; sudo /usr/bin/pon $IFNAME > \
                        /dev/null 2>&1 & )
                else
                    # PPPOE, PPPOA, WLM interfaces are started directly
                    ( umask 0; sudo setsid sh -c "/usr/sbin/pppd call $IFNAME > \
                        /tmp/${IFNAME}.log 2>&1 &" )
                fi
        fi
```

后台是通过pppd来完成pppoe的过程。
PPPoE涉及一些复杂的交互，比如客户端发起连接请求，双方会话建立，认证，IP地址分配，数据传输，终止服务等。我们这里不做深入的解读，仅关注IPv6地址分配这部分内容。服务端首先会向客户端发起一个PPP IPv6CP的请求，里面携带服务端的接口ID；客户端也会向服务端发起一个一个PPP IPv6CP的请求，里面携带客户端的接口ID；双方都会就对方的IPv6CP请求响应一个ACK消息。客户端根据自己的接口ID结合FE80前缀给自己设置link-local IP。
服务端会通过自己的link-local IP发送一个Router Advertisement组播消息，客户端根据这个消息完成IPv6的自动配置。

```bash
45: pppoe0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1492 qdisc pfifo_fast state UNKNOWN group default qlen 100  
    link/ppp
    inet 117.*.*.120 peer 117.*.*.1/32 scope global pppoe0
       valid_lft forever preferred_lft forever
    inet6 240e:*:*:*:14c5:*:*:f579/64 scope global dynamic mngtmpaddr
       valid_lft 1432261620sec preferred_lft 1432261620sec
    inet6 fe80::14c5:*:*:f579/10 scope link
       valid_lft forever preferred_lft forever
```

客户端从上述Router Advertisement消息中也会学习到网关信息：

```shell
default via fe80::fa75:*:*:be21 dev pppoe0 proto ra metric 1024 expires 1135sec pref medium
```

这个网关其实就是服务端的link-local IP。

## LAN口DHCPv6客户端配置

WAN口完成IPv6的autoconf之后，UBNT路由器会发起一个DHCPv6的客户端请求，通过这个请求给LAN口申请一个IPv6 前缀，并配置一个IPv6地址到LAN口。
同样我们先通过配置入手，相关的配置指令为：`set interfaces bridge br0 pppoe 0 dhcpv6-pd *`。
这是一个前缀委派(prefix delegation)配置，对应的配置模板如下：

```shell
root@EdgeRouter-X:/opt/vyatta/share/vyatta-cfg/templates/interfaces/bridge/node.tag/pppoe/node.tag/dhcpv6-pd# cat node.def 
help: DHCPv6 Prefix Delegation options

priority: 999 # Run after interface has been configured

end: sudo /opt/vyatta/sbin/dhcpv6-pd-client.pl --ifname="pppoe$VAR(../@)" --update
```

此处涉及到/opt/vyatta/sbin/dhcpv6-pd-client.pl这个脚本，脚本比较长，这里就不全文贴出，对于 --update选项，脚本会调用` my $output = gen_pd_conf($ifname);pd_write_file($pd_conffile, $output)`。 pd_conffile在/opt/vyatta/share/perl5/Vyatta/DhcpPd.pm里定义：`return "$basedir/dhcp6c-$intf-pd.conf"`。这个gen_pd_conf内容如下：

```perl
sub gen_pd_conf {
    my ($ifname) = @_;

    my $output;
    $output  = "# This file was auto-generated by $0\n";
    $output .= "# configuration sub-system.  Do not edit it.\n";
    $output .= "\n";

    my $config = new Vyatta::Config;
    my $interface = new Vyatta::Interface($ifname);
    die "Unknown interface type: $ifname" unless $interface;

    $output .= "interface $ifname {\n";
    my $prefix_only = 0;
    my $path = $interface->path();
    $config->setLevel("$path dhcpv6-pd");
    if ($config->exists('prefix-only')) {
        $prefix_only = 1;
    } else {
        $output .= "\tsend ia-na 0;\n";
    }
    $output .= "\trequest domain-name-servers, domain-name;\n";

    my $rapid = $config->returnValue('rapid-commit');
    $rapid = 'enable' if !defined $rapid;
    $output .= "\tsend rapid-commit;\n" if $rapid eq 'enable';

    my @pds = $config->listNodes('pd');
    foreach my $pd (@pds) {
        $output .= "\tsend ia-pd $pd;\n";
    }
    $output .= "\tscript \"/opt/vyatta/sbin/ubnt-dhcp6c-script\";\n";
    $output .= "};\n\n";

    $output .= "id-assoc na 0 {};\n\n" if $prefix_only == 0;

    foreach my $pd (@pds) {
        $output .= "id-assoc pd $pd {\n";
        $config->setLevel("$path dhcpv6-pd pd $pd");
        my $prefix = $config->returnValue('prefix-length');
        $prefix = 64 if !defined $prefix;
        $prefix =~ s/\///;
        my $sla_len = 64 - $prefix;
        $output .= "\tprefix ::/$prefix infinity;\n";
        my @intfs = $config->listNodes('interface');
        my $count = 0;
        foreach my $intf (@intfs) {
            $output .= "\tprefix-interface $intf {\n";
            $config->setLevel("$path dhcpv6-pd pd $pd interface $intf");
            my $prefix_id = $config->returnValue('prefix-id');
            if (defined $prefix_id) {
                $prefix_id = validate_sla_id($sla_len, $prefix_id);
                $output .= "\t\tsla-id $prefix_id;\n";
            } else {
                $output .= "\t\tsla-id $count;\n";
            }
            $output .= "\t\tsla-len $sla_len;\n";
            my $host_addr = $config->returnValue('host-address');
            if (defined $host_addr) {
                $host_addr = validate_ifid($host_addr);
                $output .= "\t\tifid $host_addr;\n";
            }

            $output .= "\t};\n";
            $count++;
        }
        $output .= "};\n\n";
    }

    return $output;
}
```

从这个配置文件名，我们可以查看下涉及的进程名称：

```shell
root@EdgeRouter-X:~# ps -ef|grep dhcp6c
root     11349     1  0 May28 ?        00:00:00 /usr/sbin/dhcp6c -c /var/run/dhcp6c-pppoe0-pd.conf -p /var/run/dhcp6c-pppoe0-pd.pid -df pppoe0
root     27783 32434  0 09:31 pts/0    00:0
```

dhcp6c,也就是wide-dhcpv6-client。这个应用通过ifup.d脚本在相关网络接口生效后启动：

```shell
root@EdgeRouter-X:/etc/network/if-up.d# ls -ltr
total 12
-rwxr-xr-x    1 root     root          1483 Jun  2  2015 upstart
lrwxrwxrwx    1 root     root            33 Jan 28  2016 wide-dhcpv6-client -> ../../wide-dhcpv6/dhcp6c-ifupdown  
......
```

/etc/wide-dhcpv6/dhcp6c-ifupdown内容如下：

```shell
root@EdgeRouter-X:~# cat  /etc/wide-dhcpv6/dhcp6c-ifupdown
#!/bin/sh
# Updates information whenever a network interface is brought up.

[ "$IFACE" = "lo" ] && exit 0
[ -r /etc/default/wide-dhcpv6-client ] || exit 0
[ -x /usr/sbin/dhcp6ctl ] || exit 0

# Check if dhcp6c is running
pidof dhcp6c > /dev/null 2>&1 ; [ $? -eq 1 ] && exit 0

. /etc/default/wide-dhcpv6-client

case $MODE in
    start|stop)
        for i in $INTERFACES ; do
            if [ "$IFACE" = "$i" ] ; then
                /usr/sbin/dhcp6ctl $MODE interface $IFACE
                exit 0
            fi
        done
    ;;
    *)
        if [ "$VERBOSITY" = "1" ] ; then
            echo "dhcp6c-ifupdown: unknown mode \"$MODE\""
        fi
        exit 1
    ;;
esac
```

里面有个配置文件/etc/default/wide-dhcpv6-client，定义了WAN口的接口名，如果dhcp6c启动失败，这个配置文件也在排查列表。

```shell
root@EdgeRouter-X:~# cat /etc/default/wide-dhcpv6-client
INTERFACES="pppoe0"
```

我们再来看下实际生成的/var/run/dhcp6c-pppoe0-pd.conf 文件：

```shell
root@EdgeRouter-X:~# cat /var/run/dhcp6c-pppoe0-pd.conf 
# This file was auto-generated by /opt/vyatta/sbin/dhcpv6-pd-client.pl
# configuration sub-system.  Do not edit it.

interface pppoe0 {
        send ia-na 0;
        request domain-name-servers, domain-name;
        send rapid-commit;
        send ia-pd 0;
        script "/opt/vyatta/sbin/ubnt-dhcp6c-script";
};

id-assoc na 0 {};

id-assoc pd 0 {
        prefix ::/60 infinity;
        prefix-interface switch0 {
                sla-id 1;
                sla-len 4;
                ifid 1;
        };
};

```

interface pppoe0 开始的段落是接口相关的语句，pppoe0表示dhcpv6客户端从pppoe0接口申请IP。对应的花括号内可以包含多个send指令，send 指令包含的选项有rapid-commit，ia-na和ia-pd。
我们在《IPv6基础》一文中已经指出rapid-commit是需要双向奔赴的，客户端和服务端都需要在报文中设置这个选项。设置了这个选项，我们通过solicit和advertisement两个消息就可以完成地
址自动配置。
IA-NA是向服务端申请一个非临时IP，而IA-PD是向服务端申请一个前缀委派。
request指令用于指示哪些其它信息（O）需要包含在报文里，可用的选项有:domain-name-servers,domain-name,ntp-servers,sip-server-address,sip-domain-name,nis-server-address,nis-domain-name,nisp-server-address,nisp-domain-name,bcmcs-server-address,bcmcs-domain-name,refreshtime。其中refreshtime是仅和information-request消息搭配使用。
script指令里配的是一个脚本的全路径，当应用收到符合某种条件的回复消息时，会执行这个脚本。
interface里还支持information-only语句，表明这个客户端只申请DNS服务器等其它信息，不需要有状态的地址信息。
除了interface之外，底下还有两个id-assoc相关的配置。我们在《IPv6基础》一文中提到身份关联（Identity association，IA）是dhcpv6中用于关联dhcp配置和客户端接口的。id-assoc后面可以接na或者pd作为IA类型，需要同interface里的send ia-na/ia-pd能一一对应。
对于na，后面可跟的语句是0到多个`address ipv6-address pltime [vltime];`，这里的address是关键字；ipv6-address是你想要服务端给你下发的IP地址，比如2409:x:x:x:x:x:x:2222；pltime和vltime是首选和有效生存时间字段，是一个数值类型，也可以设置成infinity表示无限制。
对于pd，后面同样可以接1到多个`prefix ipv6-prefix pltime [vltime];`，除此之外还可以接prefix interface语句。
prefix interface 语句格式如下：

```text
prefix-interface interface { substatements };
```

地址申请下来后，dhcpv6客户端会将它赋给interface指定的接口上。这个语句支持如下子语句：
**sla-id** ID，SLA是站点级别可汇聚前缀（site-level aggregator）的缩写，这个其实就是我们之前在《IPv6基础》上提到的地址块中黄色部分子网id概念。这个值也是ubnt路由器配置中的prefix-id字段。
**sla-len** length，sla长度字段，这个字段在ubnt路由器上是没有配置项的。路由器通过perl脚本gen_pd_conf函数自动计算出长度值，确保分配给LAN的pd是/64的（这是为了兼容radvd，后面会提到这个应用）。所以这个值和运营商可以下发的最短前缀长度A（我们这里是60）有关，也就是64-A。
**ifid** ID，如果指定了ifid，dhcpv6服务端下发na时会以这个字段作为接口id；如果没有指定，系统默认设置EUI-64地址作为接口id。
**ifid-random**，配置这个子语句可以让ifid使用随机值，以确保隐私。
LAN口通过这个dhcp6c获取到的地址如下所示：

```shell
9: switch0@itf0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether f4:xx:xx:xx:xx:b0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.xx.1/24 brd 192.168.xx.255 scope global switch0
       valid_lft forever preferred_lft forever
    inet6 240e:*:*:*:bd21::1/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::f692:*:*:a4b0/64 scope link
       valid_lft forever preferred_lft forever
```

## LAN口SLAAC设置

我们的LAN是需要通过SLAAC向内网设备提供IPv6服务的，这部分相对比较简单，在UBNT路由器上是通过radvd实现的。
涉及到的UBNT配置是和`set interfaces switch switch0 ipv6`相关的。
相关的进程为：

```shell
root@EdgeRouter-X:~# ps -ef|grep radvd
root      9854 29439  0 03:30 pts/0    00:00:00 grep radvd
root     28620     1  0 May04 ?        00:01:03 /usr/sbin/radvd --logmethod stderr_clean
root     28621 28620  0 May04 ?        00:00:00 /usr/sbin/radvd --logmethod stderr_clean
```

radvd配置文件如下所示：

```shell
root@EdgeRouter-X:~# cat /etc/radvd.conf
interface switch0 {
#   This section was automatically generated by the Vyatta
#   configuration sub-system.  Do not edit it.
#
#   Generated by root on Mon Apr 17 15:51:18 2023
#
    IgnoreIfMissing on;
    AdvLinkMTU 0;
    AdvRetransTimer 0;
    AdvDefaultPreference medium;
    AdvOtherConfigFlag off;
    AdvReachableTime 0;
    AdvCurHopLimit 64;
    MaxRtrAdvInterval 600;
    AdvManagedFlag off;
    MinRtrAdvInterval 198;
    AdvDefaultLifetime 1800;
    AdvSendAdvert on;
    prefix ::/64 {
        AdvAutonomous on;
        AdvPreferredLifetime 604800;
        AdvOnLink on;
        AdvValidLifetime 691200;
    };
};
```

radvd从switch0上获取IPv6地址前缀，并以/64子网向内网发送router-advert。如果switch0上配置的IPv6地址不是/64, 而是/60(我们可以通过魔改gen_pd_conf函数，把sla-len写死成0，这样可以给switch0分配一个/60的ip)，radvd启动会失败。
在UBNT路由器上radvd不是发送路由器通告的唯一实现方式，我们通过dnsmasq同样可以实现这个功能。在/etc/dnsmasq.d下创建一个ipv6ra.conf文件，内容为：

```
enable-ra  
dhcp-range=::,constructor:switch0,ra-only,slaac
```

重启服务

```shell
/etc/init.d/dnsmasq restart 
```

通过dnsmasq方式，switch0上的IPv6地址前缀可以是/60，当然，内网下发的依旧是/64。