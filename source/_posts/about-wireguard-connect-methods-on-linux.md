---
title: 關於 Wireguard 在 Linux 上的連接方式
katex: false
mathjax: false
mermaid: false
categories:
  - Tutorial
tags:
  - Linux
  - Network
  - wireguard
excerpt: 介紹在 Linux 上連接 Wireguard VPN 的幾種方式
date: 2026-04-30 23:14:45
updated: 2026-04-30 23:14:45
index_img:
banner_img:
---


感覺這東西之前好像根其他人講過很多次，所以寫篇文章節省時間

# wg-quick

我想大多數人第一時間想到的方式就是`wg-quick`了，不過我個人其實不太喜歡這個方式。

`wg-quick`最大的好處就是簡單，只要`wg-quick up`一行指令連線就成功了

不過有一點讓人困擾的是它會使用設定檔內的`AllowedIPs`作為設定的路由，像 NetworkManager 之類的方式也是如此，因此我個人覺得這種方式比較適合作為終端的連線方式，只需要連線單一 VPN 來連入一個網路的狀況。

## 為什麼我不喜歡用`AllowedIPs`設定路由

AllowedIPs 其實只是限制 wireguard 這個 tunnel 內允許通過的 IP destination，其實可以設定`AllowedIPs = 0.0.0.0/0`，但只將部分網路路由過去。

但是當同時連接多個 WireGuard interface、連接多個網路時，如果使用`wg-quick`這樣的工具，他就會把`0.0.0.0/0`當成路由來設定，多個介面就會發生衝突，導致全部無法連線，只能透過限縮`AllowedIPs`來解決。

這個方法雖然可以解決問題，不過我不太喜歡這種方式

## 關於`wg-quick`在`AllowedIPs = 0.0.0.0/0`時的行為

如果用過`AllowedIPs = 0.0.0.0/0`的設定檔，可能會注意到主路由表並沒有任何更動。

這是因為`wg-quick`在`AllowedIPs = 0.0.0.0/0`時的啟動方式會略有不同

它在使用`wg set`套用設定檔時會加上一個 fwmark 0xca6c，讓這個 wireguard 連線的封包都加上這個 fwmark

並且加上一條 ip rule，`32765:  not from all fwmark 0xca6c lookup 51820`，只要不是帶有 0xca6c 這個 fwmark 的封包都會 lookup table 51820，而 table 51820 就是放著讓所有封包都通過 wireguard interface 的路由

# Manual

另一種方式就是手動設定了，這部分可以直接參考 [WireGuard 官網的 Quick Start 的 Command-line Interface 這個區段](https://www.wireguard.com/quickstart/#command-line-interface)

簡單來說就是使用`ip`這個指令先建立一個 interface，並加上 IP，然後用`wg`這個指令將設定檔套用到這個介面上，然後啟用這個介面就完成了

當然，也可以把這一系列操作寫成腳本，開機自啟動之類的，不過在我的 Debian router 上，我會喜歡把他寫進`ifupdown`的設定裡，這樣可以和其他介面配置方式整合在一起，更方便管理

具體大概類似這樣，`$IFACE`會被`ifupdown`自動替換成介面名稱，只要把介面名稱對應的設定檔放在 /etc/wireguard 目錄下即可，並且也可以直接設定一些路由在這裡

/etc/network/interfaces

```conf
auto wg-interface
iface wg-interface inet6 static
    address x.x.x.x/x
    pre-up ip link add $IFACE type wireguard
    pre-up wg setconf $IFACE /etc/wireguard/$IFACE.conf
    up ip -4 route add x.x.x.x/x dev $IFACE
    post-down ip link del $IFACE
```



