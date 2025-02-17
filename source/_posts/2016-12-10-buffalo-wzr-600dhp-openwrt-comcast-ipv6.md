---
layout: post
title: Buffalo WZR-600DHP OpenWrt使用Comcast的IPv6
author: MaskRay
tags: [router, openwrt, network]
---

## 路由器

根據<https://wiki.openwrt.org/toh/buffalo/wzr-600dhp>在Buffalo WZR-600DHP上安裝OpenWrt。Buffalo路由器的好處在於brick了還能借助bootloader用tftp安裝。

我的Internet service provider爲COMCAST-7922 - Comcast Cable Communications, LLC, US，提供了4個IPv6 delegated prefix /64。

<!-- more -->

配置odhcpd的DHCPv6 relay，修改`/etc/config/dhcp`：

```
config dhcp 'wan6'
        option dhcpv6 relay
        option ra relay
        option ndp relay
        option master 1
```

修改`/etc/config/network`。wan和wan6的bridge沒啥用，註釋掉`option type 'bridge'`。

```
config interface 'wan'
        ......
        #option type 'bridge'

config interface 'wan6'
        ......
        #option type 'bridge'

config interface 'lan'
        ......
        option ip6assign '60'
```

如果不填寫`ip6assign '60'`的話，在連接路由器的設備上`dhcpcd -6`會看到：

```
% sudo dhcpcd -6 wlp3s0
DUID 00:01:00:01:1d:04:34:19:d0:7e:35:f4:c6:3a
wlp3s0: IAID 35:f4:c6:3a
wlp3s0: soliciting an IPv6 router
wlp3s0: Router Advertisement from fe80::b2c7:45ff:fe75:9e90
wlp3s0: adding default route via fe80::b2c7:45ff:fe75:9e90
wlp3s0: soliciting a DHCPv6 lease
wlp3s0: fe80::b2c7:45ff:fe75:9e90: DHCPv6 REPLY missing IA Address
wlp3s0: no useable IA found in lease
wlp3s0: fe80::b2c7:45ff:fe75:9e90: DHCPv6 REPLY missing IA Address
wlp3s0: no useable IA found in lease
wlp3s0: fe80::b2c7:45ff:fe75:9e90: DHCPv6 REPLY missing IA Address
wlp3s0: no useable IA found in lease
wlp3s0: fe80::b2c7:45ff:fe75:9e90: DHCPv6 REPLY missing IA Address
wlp3s0: no useable IA found in lease
^Creceived SIGINT, stopping
wlp3s0: removing interface
dhcpcd exited
```

我使用的`openwrt-15.05.1-ar71xx-generic-wzr-600dhp-squashfs-sysupgrade.bin`比較古怪，默認禁用了`eth0`(lan)及`eth1`(wan)的IPv6。

```
root@OpenWrt:~# sysctl -a | grep disable_ipv6
......
net.ipv6.conf.eth0.disable_ipv6 = 1
net.ipv6.conf.eth1.disable_ipv6 = 1
......
```

用`sysctl`把它們改成0。要持久化配置的話，修改`/etc/config/firewall`：

```
config defaults
	options disable_ipv6 0
```

修改完兩個文件後`/etc/init.d/network reload; /etc/init.d/dhcp reload`。

## 筆記本電腦

我的筆記本電腦安裝Arch Linux，使用netctl、netctl-auto管理網絡。向`/etc/netctl/$profile`添加兩行：

```
IP6=dhcp
DHCP6Client=dhcpcd
```

默認的DHCPv6客戶端dhclient不工作，不明原因，觀察`/usr/lib/network/ip`發現設置`DHCP6Client`即可指定DHCPv6客戶端。

之後訪問<http://test-ipv6-ct.comcast.net>，查看IPv6評分，10 of 10。

```
Your IPv4 address on the public Internet appears to be x.x.x.x
Your IPv6 address on the public Internet appears to be x:x:x:x:x:x:x:x
Your Internet Service Provider (ISP) appears to be COMCAST-7922 - Comcast Cable Communications, LLC, US
Since you have IPv6, we are including a tab that shows how well you can reach other IPv6 sites. [more info]
Good news! Your current configuration will continue to work as web sites enable IPv6.
Your DNS server (possibly run by your ISP) appears to have IPv6 Internet access.
```

## 診斷

路由器上`opkg install tcpdump`，之後在筆記本電腦上`mkfifo fifo; ssh root@192.168.1.1 'tcpdump -s0 -Unw - -i eth1' > fifo`，另一個shell裏`wireshark -ki fifo`。

IPv6一直弄不好，在TUNA羣裏大家討論診斷了很久，[王邈](https://innull.com)給出了很多診斷建議。小結下有這些東西：

```zsh
ip -6 a         # address
ip -6 r s t all # route
ip -6 ru        # rule
sysctl -a | grep net.ipv6.conf.$iface    # $ifame 填 eth0 eth1 br-lan br-wan 等
ip6tables -nL
```

注意以下值：
```
net.ipv6.conf.eth1.accept_ra = 1
net.ipv6.conf.eth1.autoconf = 1
net.ipv6.conf.eth1.disable_ipv6 = 0
```

怕ip6tables產生影響的話，`ip6tables -F; ip6tables -P ACCEPT FORWARD`。

對於OpenWrt系統，當`disable_ipv6=1`的時候，如果接口外面套了bridge，那麼對外發送IPv6包(如connect、sendto等syscall)找不到路由時會報告EACCES(Permision denied)；即使有路由，tcpdump也看不到有包發出去。當心套上bridge後會丟失一些錯誤信息，`ip -6 a a $ip6 dev br-wan`不會報錯，但`ip -6 a a $ip6 dev eth1`會報告EACCES(Permission denied)。

## 其他

之前使用[D-Link DIR-860 B1](https://wiki.openwrt.org/toh/d-link/dir-860l)，<https://www.amazon.com>上有打折後$30.75的。通過網頁界面安裝OpenWrt很方便，但是室友報告Steam上很多遊戲有明顯丟包。
