---
title: 使用 nftables 和 tc 在 Linux 上實現 QoS
katex: false
mathjax: false
mermaid: false
excerpt: 介紹 tc 的使用，以及如何使用 nftables 和 tc 配合實現自動針對每個 IP 進行流量控制
date: 2024-09-08 20:43:12
updated: 2024-09-08 20:43:12
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

在家中的網路會需要做 QoS，主要是因為 ISP 的限速策略做的挺爛的，我家的 ISP 是屬於二級 ISP，簡單來說就是他會和中華電信買幾條網路，在往下分租給整棟樓的人。

我家的網路是 100/10Mbps (down/up)，與 ISP 的鏈路速度是100Mbps full duplex，下載當然沒什麼問題，就跑滿鏈路速度，但是上傳就有問題了， ISP 允許客戶以鏈路速度進行上傳，但當滿速上傳時只要0.1秒就會達成速度上限，然後 ISP 就會直接 drop 掉你後面0.9秒的流量(大約)，這就導致只要有東西在上傳，那麼其他人的延遲會直接從 < 30ms，跳到 > 4000ms 以上，所以只好在內部先對上行流量做完整型限制後再上傳，以此來控制延遲。

# 介紹

## tc (traffic control)

tc 一般都只對出站(egress)流量進行操作，雖然也可以對入站(ingress)流量進行操作，但相比出站的操作就少很多，所以如果要限制下載速度的話，一般就是在內網對應的網卡上進行限制(這裡的出站流量相當於內網的下載)

## 結構介紹

tc 由三個部分組成

- qdisc (queueing discipline)： 隊列規則
- class： 類別
- filter： 過濾器

整體結構如下。

![](tc_tree.png)

圖片來自 [CSDN 文章](https://blog.csdn.net/qq_44577070/article/details/123967699)[^1]，依據 CC BY-SA 授權

每個網路設備只會有一個根(root)的隊列規則(qdisc)，而 qdisc 又分為無類規則(Classless Qdisc)和有纇規則(Classful Qdisc)。

- Classless Qdisc 很簡單，他底下不會有手動創建的類別，流量怎麼排隊全看規則內部的設定，在 systemd 217後，預設的規則是 fq_codel，就是一種無類規則。

- Classful Qdisc 則是比較複雜，你可以在 root qdisc 底下創建各種不同的纇(class)，每個 class 可能對流量有不同的速度限制，而在 class 底下還可以放不同的 class，或是在 class 底下再放一個 qdisc，如果是 Classful Qdisc，就可以在底下創建更多的 class，因此 Classful Qdisc 可以達成較複雜、較精細化的流量控制。

而 filter 則是負責決定要將哪個封包丟到哪個 class，如果 class 底下還有 class，那就要被第二個 filter 決定要丟到更底下的哪個 class，直到最後到達最底下的 class，就會接受這個 class 底下的 qdisc 進行排隊(如果沒有給 class 設定最後的 qdisc，會因為不同種類的 parent qdisc 會有不同的最終 qdisc)

### qdisc

常見的隊列規則如下，更詳細的介紹可以參考 [ArchWiki](https://wiki.archlinux.org/title/Advanced_traffic_control)[^2]，還有很多種隊列規則，各有不同的特性，有興趣可以自己去翻 man page。

- Classless Qdisc
  - fq： 公平隊列
  - fq_codel： 結合公平隊列和延遲控制
  - fq_pie： 結合公平隊列和 pie
  - codel： 延遲控制
  - pie： Proportional Integral controller-Enhanced
  - cake： Common Applications Kept Enhanced
  - red： 隨機早期探測
  - sfq： 隨機公平隊列
  - pfifo： 先進先出
  - pfifo_fast： 先進先出，但有分三個通道，直到高優先級的通道沒有封包，才會輪到低優先級的通道
  - tbf： 令牌桶過濾，限速的好選擇[^3]
- Classful Qdisc
  - htb： 分層令牌桶，基本上要分區流量控制都是用這個
  - cbq： Class Based Queueing
  - prio： 優先級分類，不進行限速


### filter

filter 也有很多種規則，有興趣可以自己去翻 man page。

基本上會用到的只有兩種過濾器， u32 和 fw。

u32 會使用 IP 封包的內容來過濾，透過設置 match 和偏移量可以按照來源/目標位置、來源/目標端口來決定要分到哪個 class，或是更複雜的檢索功能，但是這個 filter 很難撰寫規則，在 man page 被戲稱為 ugly 32 bits filter(原名為 universal 32 bits filter)。

fw 就更好理解了，按照 fwmark 來決定要分到哪個 class，舊型 iptables 就是這樣與 tc 配合進行流量限速

不過如果有使用 nftables 的話，就可以直接跳過 filter 了， nftables 可以直接匹配後將封包直接塞到某個 class 裡，不需要進行 mark 再由 tc 分配，配合 nftables 的結構可以達成更好的效能和匹配。

### handle/classid

handle/classid 就是圖上顯示的`x:y`的數字，`x`和`y`都是可以允許最高16位的16進制數字。

每個 qdisc 都會有一個 handle 也就是`x:0`，也可簡寫為`x:`，而在這個 qdisc 底下的 class 就會有對應的 classid `x:y`，這會是識別不同 class 的參數。

## 分配規則

以 HTB 為例，結構如下

- root qdisc HTB 1:0
  - class 1:1 rate 2Mbit ceil 3Mbit prio 0
  - class 1:2 rate 2Mbit ceil 8Mbit prio 1
  - class 1:3 rate 1Mbit ceil 8Mbit prio 1

rate 代表保證帶寬， ceil 則是最高帶寬， prio 代表優先級，0最高7最低，這個設備上傳總帶寬為10Mbit

當今天只有 class 1:1在使用時，他可以使用最高 3Mbit 的帶寬，同理，只有 class 1:2或1:3使用時，他們都可以使用 8Mbit 的帶寬，這些都小於總帶寬 10Mbit。

但如果今天發生了爭搶，例如三個 class 都想要使用，那麼就會優先按照保證帶寬分配，所以每個 class 都會分到 rate 設定的值，現在剩餘帶寬還剩 10-2-2-1=5Mbit，優先滿足高優先級，因此 class 1:1可以使用 3Mbit，此時還剩 4Mbit 的帶寬，由於 class 1:2和1:3有相同優先級，因此會按照 rate 的比例來分配， class 1:2分到 2+2.6Mbit， class 1:3分到 1+1.4Mbit。

如果爭搶時連 rate 都無法滿足，就是優先滿足高優先級，相同優先級按照 rate 比例分配。

# 設定

我設定流量控制的目標是區分出少數需求帶寬少但高優先級的流量，以及設定對子網內每個 IP 的限速。

因此結構如下

- dev wan
  - root qdisc HTB rate 10Mbit 1:0 default 0x100
    - class 1:1 htb rate 10Mbit
      - class 1:10 rate 2Mbit
      - class 1:20 rate 2Mbit ceil 9Mbit prio 3
      - class 1:90 rate 2Mbit ceil 9Mbit prio 6
        - other IP class
      - class 1:100 rate 2Mbit ceil 8Mbit prio 7

rate 代表保證帶寬， ceil 則是最高帶寬， prio 代表優先級，0最高7最低。

class 1:10和1:20就是高優先級的 class，未來可以再細分。

class 1:90就是用來對每個 IP 進行限速的 class，每個 IP 都會被創立一個在1:90底下的 class 進行限速。

class 1:100就是沒有 match 到任何一個 class 的流量最後會到的地方， root qdisc 指定的 default 0x100。

## 腳本

因為 tc 的規則在重新啟動後會歸零，所以要使用腳本讓他在開機的時候可以重新生效。

### 初始設定

隨便找個地方放初始設定的腳本，像是`/opt/traffic-control/tc-setting.sh`，假設對外的網路設備是`wan`，上傳帶寬最大是 10Mbit，如果環境不同，要自己修改適合自己的數值。

```bash
#!/bin/bash

# clear qdisc
/sbin/tc qdisc delete dev wan root

/sbin/tc qdisc add dev wan root handle 1:0 htb default 100
/sbin/tc class replace dev wan parent 1: classid 1:1 htb rate 10mbit
tc_class='/sbin/tc class replace dev wan parent 1:1 classid'
$tc_class 1:10 htb rate 2mbit ceil 2mbit
$tc_class 1:20 htb rate 2mbit ceil 9mbit prio 3
$tc_class 1:90 htb rate 8mbit ceil 9mbit prio 6
$tc_class 1:100 htb rate 2mbit ceil 8mbit prio 7
```

腳本功能就是設定剛剛說的結構，腳本內只有設定 qdisc 和 class，因為 filter 的部分會直接在 nft 做

將腳本加上執行權限`chmod +x /opt/traffic-control/tc-setting.sh`

編輯`/etc/systemd/system/tc.service`建立自動啟動服務，並啟用`systemdctl daemon-reload ; systemdctl enable --now tc.service`

```service
[Unit]
Description=traffic control
After=nftables.service
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/opt/traffic-control/tc-setting.sh

[Install]
WantedBy=multi-user.target
```

---

接著要調整 nftables 的設定，編輯`/etc/nftables.conf`(~~不要問我為什麼是在 filter 做流量的分類， nftables 的 wiki 也這樣做~~)

```conf
table inet filter {
        map tc_classid {
                type ipv4_addr : classid
                counter
                flags interval
        }
        chain FORWARD {
                type filter hook forward priority filter; policy accept;
                meta priority none meta mark 0 meta priority set ip saddr map @tc_classid counter
        }
}
```

在 forward 鏈的這個規則，會匹配沒有被分類和沒有被打上 fwmark 的流量並幫他們打上分類，按照`tc_classid`裡的鍵值映射來幫對應的來源 IP 分類

這樣基礎的設定就完成了，不過分 IP 限速還沒生效，因為`map tc_classid`裡面沒有任何東西，所有的流量都會被歸類到1:100，按照他的規則限速，所以接下來要用自動執行的腳本自動新增 IP 和對應的 class

### 自動限制

編輯`/opt/traffic-control/arp-limit.sh`，並加上執行權限`chmod +x /opt/traffic-control/arp-limit.sh`

```bash
#!/bin/bash

grep lan /proc/net/arp | grep 0x2 | grep 10.0.0 | awk -F ' ' '{print $1}' | while read ipaddr
do
classid_upload="1:f${ipaddr##*.}"
/sbin/nft add element inet filter tc_classid {$ipaddr\:$classid_upload}
/sbin/tc class replace dev wan parent 1:90 classid $classid_upload htb rate 2mbit ceil 9mbit
/sbin/tc qdisc replace dev wan parent $classid_upload handle f${ipaddr##*.}: fq_codel
done
```

這個腳本會根據`/proc/net/arp`的 arp 記錄獲取 lan 底下的設備 IP，並過濾 Flags 為`0x2`的值，這代表對應的 IP 在線

接著會幫每個 IP 建立在1:90底下的一個 class，並加上限速和 qdisc， id 為1:fxxx， xxx 是 IP 的最後一位，不會重複，並將 IP 和 classid 的對應加到  nftables 中

把這個腳本加到 crontab 的定期執行，要用 root 的，或是具有免密執行`sudo tc`和`nft`權限的 user 裡，`/proc/net/arp`30秒更新一次，因此每分鐘都執行檢查有沒有新設備

```cron
* * * * * /opt/traffic-control/arp-limit.sh
```

有了新增規則的部分，也要有定期刪除規則的腳本

編輯`/opt/traffic-control/delete-arp-limit.sh`，並加上執行權限`chmod +x /opt/traffic-control/delete-arp-limit.sh`

```bash
#!/bin/bash

grep lan /proc/net/arp | grep 0x0 | grep 10.0.0 | awk -F ' ' '{print $1}' | while read ipaddr
do
classid_upload="1:f${ipaddr##*.}"
/sbin/nft delete element inet filter tc_classid {$ipaddr\:$classid_upload} > /dev/null 2>&1 || continue
/sbin/tc class delete dev wan parent 1:90 classid $classid_upload
/sbin/tc qdisc delete dev wan parent $classid_upload handle f${ipaddr##*.}:
done
```

這個腳本會檢查有沒有 Flags 為`0x0`的記錄，代表設備下線，就將 nftables、tc 對應的記錄刪掉

這個腳本不需要執行的很頻繁，定期檢查即可

```cron
0 * * * * /opt/traffic-control/delete-arp-limit.sh
```

這樣就完成限速的設定了

## 參考

[^1]: [Linux TC 流量控制与排队规则 qdisc 树型结构详解（以HTB和RED为例）-CSDN博客](https://blog.csdn.net/qq_44577070/article/details/123967699)
[^2]: [Advanced traffic control - ArchWiki](https://wiki.archlinux.org/title/Advanced_traffic_control)
[^3]: [Bandwidth Management 頻寬管理 - Jan Ho 的網絡世界](https://www.jannet.hk/bandwidth-management-zh-hant/)
[^4]: [Classification to tc structure example - nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Classification_to_tc_structure_example)
[^5]: [QoS in Linux with TC and Filters](http://linux-ip.net/gl/tc-filters/tc-filters.html)
[^6]: [学习nftables和tc](https://www.bilibili.com/read/cv21089529/?jump_opus=1)