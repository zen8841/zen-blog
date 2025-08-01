---
title: 配置 Linux 做為 Router
katex: false
mathjax: false
mermaid: false
excerpt: 如何設定 Linux 實現 Router 應有的功能
date: 2024-08-01 23:36:18
updated: 2024-09-08 21:33:10
index_img:
categories:
- 教程
tags:
- linux
- network
- router
- server
---

{% note info %}

2024/8/18更新： 更新 nftables 的操作說明及進階設定：使用 Wireguard 建立 Site to Site Tunnel

2024/9/8更新： 更新 QoS 的設定：使用 nftables 和 tc 在 linux 上實現 QoS

{% endnote %}

# 前言

在更換主路由為 VyOS 的一段時間後，我突然想嘗試使用 Linux 直接手動設定 Router 的功能，藉此達成更靈活的設定，因此有了這篇記錄。

# 基本設定

在以下設定中 eth0 代表 wan 接廣域網路， eth1～n 代表 lan 接內部網，內部網路10.0.0.0/24， gateway: 10.0.0.254

使用的 Linux distro 為 debian， netinst 最小安裝

## 安裝

這個應該不用教吧。 =_=

## SSH 及權限設定

### SSH

因為這台機器接觸廣域網路，最好限制 SSH 的登入設定

`/etc/ssh/sshd_config`，自行替換\<username>

```conf
AllowUsers <username>
```

### sudo

更新並安裝 sudo，讓管理帳戶允許使用 sudo

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

```shell
# systemctl restart networking.service
```

### DNS

一般我不希望 Router 上的 dns 受到其他因素影響而變動，像是 DHCP 之類的

可以在設定完`/etc/resolv.conf`後直接鎖死這個檔案的編輯

```shell
# chattr +i /etc/resolv.conf
```

如果不想看到 DHCP client 報錯，可以讓 DHCP client 不更改`/etc/resolv.conf`

編輯新檔案`/etc/dhcp/dhclient-enter-hooks.d/nodnsupdate`

```conf
#!/bin/sh

make_resolv_conf(){
        :
}
```

覆蓋掉更新 dns 的部分則不會更新 dns

並加上執行權限

```shell
# chmod +x /etc/dhcp/dhclient-enter-hooks.d/nodnsupdate
```

## NAT

接下來有關防火牆的部分可以使用`iptables`亦或是`nft`來進行設定，不過現在 iptables-nft 後端同樣是 hook nft，在少量規則的情況下並不會有太大的性能差異，如果不想學習 nft 可以使用 iptables 設定即可。

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

安裝 iptables 及儲存 iptables 設定的工具( iptables-persistent 是 debian 系的工具， RedHat 系好像是用其他工具)

```shell
# apt install iptables iptables-persistent
```

新增 nat 規則

```shell
# iptables -A POSTROUTING -t nat -s 10.0.0.0/24 -o wan -j MASQUERADE
```

如果 wan 和 lan 的介面 MTU 不同，最好加上 mss clamping

```shell
# iptables -A FORWARD -t mangle -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

---

儲存配置

```shell
# iptables-save > /etc/iptables/rules.v4
# systemctl enable iptables.service
```

記得編輯`/etc/iptables/rules.v4`中的封包計數，讓其歸零

以 filter 表的部分舉例，將`[x:x]`中的 x 都改為0，其他幾個表也相同

```conf
*filter
:INPUT ACCEPT [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
...
COMMIT
```

---

應用設定

```shell
# iptables-restore /etc/iptables/rules.v4
```

未來想更改設定，可以直接編輯`/etc/iptables/rules.v4`，語法和 iptables 相同，只是不用加 iptables，也不用加`-t 表名` 只要將配置增加到對應表的部分內即可

如果不知道怎麼寫`rules.v4`也可以直接使用 iptables 新增規則，在直接執行`iptables-save`複製輸出新增的部分到`rules.v4`中

### nftables

安裝 nftables

```shell
# apt install nftables
```

nftables 預設沒有任何表或鏈，可以用`nft list ruleset`查看目前的防火牆狀態，應該是不會看到任何東西輸出

#### 簡易介紹

nftables 的結構是 tables-chains-rules 三層，可以這樣新增一個表

```shell
# nft add table inet filter
```

inet 是 table 的 family，可以同時包含 ipv4 或 ipv6，也可以使用 ip 作為 family，不過這樣就和 iptables 一樣了，其他 family 可以參考[這裡](https://wiki.nftables.org/wiki-nftables/index.php/Nftables_families)[^1]

---

可以這樣新增位於 filter 底下的鏈

```shell
# nft add chain inet filter INPUT [ { type filter hook input priority filter; policy accept;  } ]
```

這邊的參數比較多，可以參考 [wiki 的說明](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains)[^2]，要注意的是預設的 policy 和 type 要在創建時就指定，不然想要改只能刪掉鏈重新創建

---

可以這樣創建位於 INPUT 鏈底下的規則

```shell
# nft add rule inet filter INPUT <match> <statements>
```

match 和 stataement 和 iptables 的概念類似，就是匹配，然後決定行為，詳細可以看 [wiki 的說明](https://wiki.nftables.org/wiki-nftables/index.php/Simple_rule_management)[^3]

---

簡易的介紹大概這樣，更詳細的可以直接看 [wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)[^4]

#### NAT 的設定

雖然可以打指令來配置防火牆，但是編輯設定檔再來直接應用我覺得更加好理解及方便，因此設定會直接編輯設定檔

編輯`/etc/nftables.conf`

```conf
#!/usr/sbin/nft -f

flush ruleset

table inet mangle {
        chain FORWARD {
                type filter hook forward priority mangle; policy accept;
                tcp flags syn tcp option maxseg size set rt mtu
        }
}
table inet nat {
        chain POSTROUTING {
                type nat hook postrouting priority srcnat; policy accept;
                oifname "wan" ip saddr 10.0.0.0/24 masquerade
        }
}
```

最上面的兩行代表使用 nft 來執行設定檔的腳本還有清空 nftables 的規則， nftables 的一個優點是 Atomic Rule Replacement，可以讓規則在更換時不會像 iptables-restore 之類的腳本出現一瞬間沒有防火牆的漏洞

table nat 設定了 POSTROUTING 鏈，裡面的規則和前面 iptables 設定的相同，都是執行 masquerade，不過我把 counter 這個 statements 取消了，反正我也不會去看計數器

table mangle 則是設定了 MSS Clamping 的規則，同前面的 iptables

---

應用設定直接執行`/etc/nftables.conf`即可，啟動 nftables 服務可以在開機時回復防火牆

```shell
# /etc/nftables.conf
# systemctl enable nftables.service
```

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
    range 10.0.0.2 10.0.0.99;
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

編輯`/etc/iptables/rules.v4`中的 filter 區段

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

### nftables

編輯`/etc/nftables.conf`，增加以下設定

```conf
table inet filter {
        set internalv4 {
                type ipv4_addr
                flags interval
                auto-merge
                elements = { 10.0.0.0/24 }
        }
        set global_ratelimitv4 {
                type ipv4_addr
                timeout 60s
                flags dynamic
        }
        set global_ratelimitv6 {
                type ipv6_addr
                timeout 60s
                flags dynamic
        }
        set input_service_port {
                type inet_service
                elements = { 22 }
        }
        chain INPUT {
                type filter hook input priority filter; policy accept;
                ct state related,established accept
                ct state invalid drop
                ip saddr @global_ratelimitv4 reject with icmp type admin-prohibited
                ip6 saddr @global_ratelimitv6 reject with icmpv6 type admin-prohibited
                meta l4proto icmp accept
                iif "lo" accept
                iifname "wan" tcp dport @input_service_port ct state new limit rate over 10/minute update @global_ratelimitv4 { ip saddr }
                iifname "wan" tcp dport @input_service_port ct state new limit rate over 10/minute update @global_ratelimitv6 { ip6 saddr }
                tcp dport @input_service_port accept
                reject with icmpx type admin-prohibited
        }
        chain FORWARD {
                type filter hook forward priority filter; policy drop;
                ct state related,established accept
                ct state invalid drop
                ip saddr @internalv4 ct state new accept
        }
}
```

整體設定的邏輯和前面 iptables 的部分相同，不過為了配合 ipv4+ipv6 雙棧做了一些修改， nftables 的可讀性還不錯，應該可以直接理解設定的內容

# 進階設定

[使用 Wireguard 建立 Site to Site Tunnel](/create-wireguard-site-to-site-tunnel)

[使用 nftables 和 tc 在 Linux 上實現 QoS](/linux-qos-with-tc-and-nft)

## 參考

[^1]: [Nftables families - nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Nftables_families)
[^2]: [Configuring chains - nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains)
[^3]: [Simple rule management - nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Simple_rule_management)
[^4]: [nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)