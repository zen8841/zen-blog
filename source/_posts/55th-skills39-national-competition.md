---
title: 55屆技能競賽全國賽 - 39 資訊與網路技術
katex: false
mathjax: false
mermaid: false
categories:
  - 比賽
tags:
  - network
  - server
excerpt: 分享一下比賽的心得
date: 2025-07-29 02:03:10
updated: 2025-07-29 02:03:10
index_img:
---


# 前言

比賽結束了一週，終於提起精神來完成這次的心得了，順便補上每個月一篇文章的習慣。

去年打全國賽時完全沒準備，甚至比去年區賽的時候還不認真，結果自然是什麼都沒有，就乾脆懶得寫心得了

# 心得

這次比賽我抽了完整的一週來準備，我個人認為該準備的其實都有準備到，也有猜到好幾個考出來的東西，不過主要問題是第一天比賽的狀態沒調整好，拿分的重點卻沒有拿到太多分數，時間沒有安排好，會做的來不及做。

## Day1

第一天就是 OS Part1，因為區賽有出 IPv6 的部分，所以我有想到全國賽應該也會出來，這次全國賽和 IPv6 有關的部分是 NAT64 和 DNS64， NAT64 的部分我沒準備到，但是當場`apt search nat64`，有查到3個套件，其中`tnat64`我當場試沒做出來，`jool-dkms`有成功完成 NAT64，後來聽完賽後的檢討， jool 可以直接改 modprobe 的參數，寫進`/etc/module`就好，感覺比我在比賽中改`/etc/jool.conf`的方法更簡單，~~不過 NAT64 真的是很鬼畜的技術，我想說這是不被 RFC 推薦的技術，應該不會考出來，就沒準備這 part， 我寧願 IPv6 + IPv4 Dual stack~~， DNS64的部分則是完全沒時間在比賽中做到那部份。

另外一個 IPv6 我撞到的地方是 CISCO， 我沒開`ipv6 unicast-forwarding`，~~對 CISCO 真的不熟~~，我就在 Linux 那端開`tcpdump`，想說怎麼 CISCO 老是不回 NS 的 NA，最後 IPv6 就是到 CISCO 這段沒通，我也在這裡花太多時間，沒時間改成完全使用 IPv4 (比賽有給一套放棄 IPv6 分數的結構，但我太自信，想說 IPv6 我 OK，結果浪費太多時間，還沒連通)

今年還考了 OCSP 的部分，其實這個考點我也有猜到，我也特別去看 Windows server 上的 ADCA 要怎麼做 OCSP，不過我沒準備 Linux 上 CA 的 OCSP，一方面是我[以前做過](/using-openssl-to-create-ca-on-linux/#OCSP-可選)，我知道 OCSP 要記的指令太長、我懶得背，另一方面是去年的考題很開放，都沒限定東西要放哪，我準備的時候想說萬一真的考出 OCSP 就直接用 ADCA 做就好，結果題目直接出了 Linux CA，直接戳中我沒準備的部分。

另外一個今年第一次出現的考點是要求使用外部的 DNS ( Linux 上的 DNS )，不在 Winsrv 上安裝 DNS 下安裝 ADDC，我猜測就是給開 update 權限應該就可以，在 bind 裡新增一個 zone，並設定`allow-update { winsrv ip; };`， 設定 Winsrv 的 Primary DNS 指向 Linux，直接安裝 ADDC，不過設定中取消勾選安裝 DNS，我現場有成功做出來， Winsrv 會自動更新記錄到 bind 中。

還有最後一個我沒拿到分的部分是 Linux 和 Windows 間的 GRE tunnel；在比賽前一天的準備階段看到 VM 網路分別的名稱時就有猜到網路的結構，根據這個結構我有猜到可能會考兩個 gw 間的 tunnel，在52屆時就有考過這點，我也有提前準備，但在測試環境裡總是沒辦法成功(後來發現可能是我配置 RAAS 模塊的時候只初始化了 VPN 的部分，沒初始化 NAT )，我只有準備到要用`Add-VpnS2SInterface`這個指令可以建立 GRE tunnel，結果比賽時一直沒辦法和 Linux 連通，比完才知道要把 Winsrv 重啟才會生效，~~Windows 真的毛很多~~，隔天因為題目都不會，回來玩第一天的題目就成功搭起來了，其實我 D1 應該就要能成功才對，這部份的分數就丟掉了，還浪費了很多時間。

剩下基礎的 web 之類的部分，因為前面花太多時間，反而會做的都沒做到。

## Day2

第二天是接續第一天的 OS Part，不過今年的出題改了不少，以往第二天必定會出現 Ansible、Mail、Web mail 其中的幾個，或是全部出現，但今年都沒出這幾個項目。

今年的 OS Part2包含幾個主題， IPsec、SSTP VPN、SMB over Quic、SSH Certificate-Based Authentication 和 RemoteApp( RemoteApp 有額外提供 PowerShell Module，要在離線下安裝 )，其中 SMB over Quic 和 RemoteApp 應該是沒人做出來所以不會做也應該影響不大。

SSH Certificate-Based Authentication 我翻設定檔和 man 沒做出來；SSTP 的部分我沒練到，不過 Winserver 上的設定大部分都可以點一點來解決，我是卡在憑證的部分，他設定不允許使用 Web 的憑證，~~明明 EKU 都是一樣的~~；IPsec 算是我有猜到的題目，不過這次是考 Linux 和 CISCO 對接，最後是沒成功做出來，我嘗試把 Strongswan 和 CISCO 的 crypto、crypto map 對應的 phase1 和 phase2 參數設成相同，但看`tcpdump`，兩者似乎沒有成功交換金鑰，等之後再來檢討吧。

## Day3

在 OS Part 結束之後狀態就好很多了，第三天做了不少的項目。

今年的 Network Part 是佈建，聽裁判講是一年 Debug 一年佈建，佈建的難度比較低，~~不過我覺得去年的 Debug 比較好玩~~，今年的題目和50屆的 Network Part 蠻像的，這一屆是我能找到唯一有公開 Network Part 的全國賽題目。

第三天我唯一覺得我該拿分卻沒拿到分的部分是 Multilink PPP，當天早上搭捷運到南港展覽館的路上我還特別看了 [Jan Ho 的 PPP 文章](https://www.jannet.hk/point-to-point-protocol-ppp-zh-hant/)，其中就有 Multilink PPP，所以題目出來說要把兩條 Serial 做鏈路聚合的時候我就知道要用 Multilink PPP，但要做的時候就把早上剛看的全忘光了，也找不到對應的設定，這部份是真的很氣自己。

## Final

這次的比賽主要是我自己的時間沒掌控好，第一天因為這樣把心態搞崩就沒了，而主要的得分點應該是在 Day1 和 Day3，~~不過這次時間真的給很少，第一天好像6、7頁題目，4.5小時，第二天正反兩面題目，6小時，第二天因為不會做，回去玩第一天的題目還做出來不少~~，最後只撈到了一支佳作，~~沒撈到錢~~，沒能榮譽從比賽畢業(被禁賽)，因為年齡限制這次第二次參加就是最後一次，以後也不能比39職纇了。不過沒得名就真的是技不如人，練得不夠多，沒有練到心態穩定，可能我真的不擅長這種比賽吧。
