---
title: 使用Wireguard建立Site to Site Tunnel
katex: false
mathjax: false
mermaid: false
excerpt: 介紹如何在router上建立site to site tunnel，並讓路由通過
date: 2024-08-18 18:30:54
updated: 2024-08-18 18:30:54
index_img:
categories:
- 教程
tags:
- router
- wireguard
- network
- linux
---

# 前言

算是接續之前的linux router設定的部分，建立wireguard site to site tunnel

# 設定

## Wireguard

首先在兩端分別產生公、私鑰

```shell
# wg genkey | tee private.key | wg pubkey > public.key
```

也可以多產生一個PresharedKey，這個只要在其中一邊產生就好，兩邊都是相同的

```shell
# wg genpsk
```

編輯`/etc/wireguard/<wg-interface>.conf`，`<wg-interface>`就是待會interface的名字，我一般會設定成`wg-<toLocation>`

```conf
[Interface]
PrivateKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ListenPort = xxxxx

[Peer]
PublicKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Endpoint = x.x.x.x:xxxxx
PresharedKey = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

- PrivateKey填自己的私鑰
- PublicKey填另一個端點的公鑰
- PresharedKey都是一樣的值(可選)
- AllowedIPs填0.0.0.0/0才可以讓所有路由通過，如果有IPv6要過，也可多加::/0
  - 如果使用wg-quick或其他工具啟動wireguard，會使用AllowedIPs來設定路由，不過這個設定使用`ip`和`wg`指令來建立通道，AllowedIPs就只會限制可以通過這個Tunnel的封包IP，所以全開
- Endpoint填對方的IP和ListenPort
  - 也可以只有一邊有ListenPort，不設定Endpoint，另一端設定Endpoint，不設定ListenPort，讓另一端來連接有ListenPort的這一端

在兩邊都完成wireguard的config

## ifupdown

編輯`/etc/network/interfaces`，增加這段，`<wg-interface>`就是剛才設定的

```conf
auto <wg-interface>
iface <wg-interface> inet static
    address 10.10.10.1/32
    pointopoint 10.10.10.2
    pre-up ip link add $IFACE type wireguard
    pre-up wg setconf $IFACE /etc/wireguard/$IFACE.conf
    post-down ip link del $IFACE
```

另一端的interface設定

```conf
auto <wg-interface>
iface <wg-interface> inet static
    address 10.10.10.2/32
    pointopoint 10.10.10.1
    pre-up ip link add $IFACE type wireguard
    pre-up wg setconf $IFACE /etc/wireguard/$IFACE.conf
    post-down ip link del $IFACE
```

其實就是`address`和`pointopoint`互換，使用`/32的address`和`pointopoint`可以節省IP，這會設定一個/32的路由到對方

如果想要設定路由可以跑OSPF之類的動態路由協議，或是直接設定static route，如下

假設`10.10.10.2`這一邊一邊有`10.0.0.0/24`的路由，可以在`10.10.10.1`的這邊在\<wg-interface\>這邊設定這些

```conf
    up ip route add 10.0.0.0/24 dev $IFACE via 10.10.10.2
    down ip route del 10.0.0.0/24
```

這樣static route就會隨著wireguard interface的啟動和關閉被設定
