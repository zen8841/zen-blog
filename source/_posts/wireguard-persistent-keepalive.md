---
title: Wireguard 的 keepalive 設定
katex: false
mathjax: false
mermaid: false
excerpt: 如何在只有單端能設定 PersistentKeepalive 下維持連線
date: 2024-07-10 23:28:37
updated: 2024-07-10 23:28:37
index_img:
categories:
- 教程
- 疑難雜症
tags:
- network
- wireguard
---

{% note info %}

2025/5/23更新： 更新問題發生的實際原因與解法

{% endnote %}

# 問題

~~在使用 wireguard 時會發現一個問題，當連線長時間靜默時， wireguard 會自動切斷連線，這時就無法連線到對方。 wireguard 有提供一個對應的設定值`PersistentKeepalive`，會隔設定的秒數向對方發送心跳包，但如果只在單端設定， wireguard 會因為對方長時間沒有回應，認為對方斷線，從而切斷連線。~~

~~但是有時候 server 端自己無法控制，剛好 server 端又沒有加上`PersistentKeepalive`的設定，此時就要用一些其他的方法來維持連線了。~~

斷線實際發生的原因是上游的路由器在檢測到有長時間的 udp 連接後會自動切斷，因為 udp 沒有連接的概念，所以實際上是根據 IP 五元組來定位連接的(protocal: udp, src ip, src port, dst ip, dst port)

# 解法

~~解法其實很簡單，定時向對面發送封包即可，可以將以下指令新增到 crontab，記得新增到可以免密碼使用`wg-quick`的 user 上，或者新增到 root 的，我是使用* * * * *(每分鐘發送一次 ping，如果斷線則重啟對應的 wireguard 連線)~~

```bash
ping -c1 -W5 <peer's wireguard IP> 1>/dev/null 2>/dev/null || (sudo wg-quick down <config name> ; sudo wg-quick up <config name>)
```

---

雖然上面的方法的確可以變更 src port，來規避路由器的連接切斷，但在被切斷到偵測到後重新連接會有一小段掉線時間，既然找到了真正的問題，那麼就可以來解決了。

解決方法是專門針對這條 wireguard 的連接去做隨機 masquerade，讓出去的 port 單獨隨機一次，然後間隔固定時間切斷對應的 conntrack 連線，強迫 kernel 重新 masquerade，但不影響 wireguard process 接收的 port，以 nftable 設定做示範

```conf
table inet nat {
        chain POSTROUTING {
                type nat hook postrouting priority srcnat; policy accept;
                oifname "wan" ip daddr <wireguard endpoint ip> udp dport <wireguard endpoint port> masquerade random
                oifname "wan" ip saddr { <internal ip cidr> } masquerade
        }
}
```

每隔10分鐘切斷 conntrack，視上游路由器切斷的頻率調整

```cron
*/10 * * * * /sbin/conntrack -D -p udp --dst <wireguard endpoint ip> --dport <wireguard endpoint port>
```

經過測試，這樣做的斷線時間應該<0.1s，我使用0.1秒的間隔 ping 也只掉一個包
