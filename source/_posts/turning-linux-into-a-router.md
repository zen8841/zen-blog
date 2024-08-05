---
title: 配置linux做為router
katex: false
mathjax: false
mermaid: false
excerpt: 如何設定linux實現router應有的功能
date: 2024-08-01 23:36:18
updated: 2024-08-01 23:36:18
index_img:
categories:
- 教程
tags:
- linux
- network
- router
- server
---

# 前言

在更換主路由為VyOS的一段時間後，我突然想嘗試使用linux直接手動設定router的功能，藉此達成更靈活的設定，因此有了這篇記錄。

# 基本設定

在以下設定中eth0代表wan接廣域網路，eth1～n代表lan接內部網，內部網路10.0.0.0/24，gateway: 10.0.0.254

使用的linux distro為debian，netinst最小安裝

## 安裝

這個應該不用教吧。 =_=

## SSH及權限設定

### SSH

因為這台機器接觸廣域網路，最好限制SSH的登入設定

`/etc/ssh/sshd_config`，自行替換\<username>

```conf
AllowUsers <username>
```

### sudo

更新並安裝sudo，讓管理帳戶允許使用sudo

```shell
# apt update
# apt upgrade
# apt install sudo
# usermod -aG sudo <username>
```

## 設定網路介面

編輯`/etc/network/interface`

```conf
# 應該都會有這段
# The loopback network interface
auto lo
iface lo inet loopback

# 安裝時如果有設定網路應該有這段，刪掉或註解掉
# allow-hotplug eth0
# iface eth0 inet dhcp

# 重新命名介面名稱，可不做，方便辨認而已
rename eth0=wan
rename eth1=lan

# 配置使用auto，自動啟動介面，而非有接入才啟動，如果有重命名，需使用重命名的接口名稱
auto wan
# 如果wan是dhcp
iface wan inet dhcp
# 如果是手動設IP
# iface wan inet static
#     address x.x.x.x/x
#     gateway x.x.x.x
# 如果是pppoe撥號，參考https://wiki.debian.org/PPPoE

auto lan
iface lan inet static
# 不用設gateway，這是和lan通訊的介面
    address 10.0.0.254/24
```
重啟`networking.service`應用設定，或是直接重開機(改介面名稱可能導致重啟`networking.service`失敗，可以用重開機解決)

```bash
# systemctl restart networking.service
```

### DNS

一般我不希望router上的dns受到其他因素影響而變動，像是DHCP之類的

可以在設定完`/etc/resolv.conf`後直接鎖死這個檔案的編輯

```shell
# chattr +i /etc/resolv.conf
```

如果不想看到DHCP client報錯，可以讓DHCP client不更改`/etc/resolv.conf`

編輯新檔案`/etc/dhcp/dhclient-enter-hooks.d/nodnsupdate`

```conf
#!/bin/sh

make_resolv_conf(){
        :
}
```

覆蓋掉更新dns的部分則不會更新dns

並加上執行權限

```shell
# chmod +x /etc/dhcp/dhclient-enter-hooks.d/nodnsupdate
```

## NAT

接下來有關防火牆的部分可以使用`iptables`亦或是`nft`來進行設定，不過現在iptables-nft後端同樣是hook nft，在少量規則的情況下並不會有太大的性能差異，如果不想學習nft可以使用iptables設定即可。

不管使用哪個工具都必須先開啟轉發鏈，編輯`/etc/sysctl.conf`，找到`net.ipv4.ip_forward=1`取消註解

```conf
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1
```

應用配置，或是直接重開也會自動應用

```shell
# sysctl -p
```

### iptables

安裝iptables及儲存iptables設定的工具(iptables-persistent是debian系的工具，RedHat系好像是用其他工具)

```shell
# apt install iptables iptables-persistent
```

新增nat規則

```shell
# iptables -A POSTROUTING -t nat -s 10.60.0.0/24 -o wan -j MASQUERADE
```

如果wan和lan的介面MTU不同，最好加上mss clamping

```shell
# iptables -A FORWARD -t mangle -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

儲存配置

```shell
# iptables-save > /etc/iptables/rules.v4
# systemctl enable iptables.service
```

記得編輯`/etc/iptables/rules.v4`中的封包計數，讓其歸零

以filter表的部分舉例，將[x:x]中的x都改為0，其他幾個表也相同

```conf
*filter
:INPUT ACCEPT [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
...
COMMIT
```

應用設定

```shell
# iptables-restore /etc/iptables/rules.v4
```

未來想更改設定，可以直接編輯`/etc/iptables/rules.v4`，語法和iptables相同，只是不用加iptables，也不用加`-t 表名` 只要將配置增加到對應表的部分內即可

如果不知道怎麼寫`rules.v4`也可以直接使用iptables新增規則，在直接執行`iptables-save`複製輸出新增的部分到`rules.v4`中

### nft

待更新

## DHCP server

安裝`isc-dhcp-server`

```shell
# apt install isc-dhcp-server
```

編輯`/etc/default/isc-dhcp-server`

```conf
INTERFACESv4="lan"
```

在雙引號中加入內部網路使用的介面，如果有多個，使用空格分隔

編輯`/etc/dhcp/dhcpd.conf`

```conf
# 檔案最前面的option、default-lease-time和max-lease-time可動可不動
# 我一般會修改max-lease-time到86400，不過這些設定可以被subnet區域的設定覆蓋
# 將authoritative;取消註解
authoritative;

# 新增一個subnet，subnet和netmask使用子網對應的值
subnet 10.0.0.0 netmask 255.255.255.0 {
    # 發的IP範圍
    range 10.0.0.2 10.60.0.99;
    # 可設可不設
    option domain-name "example.com";
    # 指定DHCP分配的DNS
    option domain-name-servers 1.1.1.1;
    # 網關
    option routers 10.0.0.254;
    # 好像可加可不加？
    option broadcast-address 10.0.0.255;
    default-lease-time 3600;
    max-lease-time 86400;
}

# 如果想做static DHCP可以新增host區段
host example-device {
    # mac位置，也可以用其他方式匹配客戶端，
    hardware ethernet e8:65:d4:67:1f:68;
    # 可以配發前面range以外的IP
    fixed-address 10.0.0.101;
}
```

基本上設定檔本身就有做教學了，可以直接跟著設定檔做

## 防火牆

### iptables

編輯`/etc/iptables/rules.v4`中的filter區段

```conf
*filter
:INPUT ACCEPT [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
# 基於狀態放行或丟棄封包
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -m state --state INVALID -j DROP
# 允許icmp封包(ping)
-A INPUT -p icmp -j ACCEPT
# 允許loopback的所有封包
-A INPUT -i lo -j ACCEPT
# 一分鐘內如果超過10的新連接，則DROP掉這個IP的封包
-A INPUT -i wan -m state --state NEW -m recent --set --name filter --mask 255.255.255.255 --rsource
-A INPUT -i wan -m state --state NEW -m recent --rcheck --seconds 60 --hitcount 10 --name filter --mask 255.255.255.255 --rsource -j DROP
# 允許SSH連線
-A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
# 設定入站規則若都沒匹配，則用icmp-host-prohibited拒絕封包
-A INPUT -j REJECT --reject-with icmp-host-prohibited
# 基於狀態放行或丟棄封包
-A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -m state --state INVALID -j DROP
# 允許子網對外發起連線
-A FORWARD -m state --state NEW --source 10.0.0.0/24 -j ACCEPT
COMMIT
```

應用設定

```shell
# iptables-restore /etc/iptables/rules.v4
```

### nft

待更新

# 進階設定

待更新
