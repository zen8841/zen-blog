---
title: I218-V、I219-V系列網卡在Linux下突然關閉的解法
katex: false
mathjax: false
mermaid: false
excerpt: e1000e driver在高流量下網卡會突然關閉並留下錯誤記錄後重啟
date: 2024-07-09 22:05:43
updated: 2024-07-09 22:05:43
index_img:
categories:
- 教程
- 疑難雜症
tags:
- network
- server
- linux
---

# 前情提要

在今天下午上傳一個檔案的過程中，突然發現傳輸過程斷斷續續的，接著就徹底連不上了，進到pve的後台查看，網卡和IP都還在，但就是無法連線上另一端。

# 症狀

使用`dmesg`查看kernel log後發現出現了一堆err等級的log

```log
[xxxxxx] e1000e 0000:00:1f.6 eno1: Detected Hardware Unit Hang:
                  TDH                  <44>
                  TDT                  <9b>
                  next_to_use          <9b>
                  next_to_clean        <43>
                buffer_info[next_to_clean]:
                  time_stamp           <101d5f5e3>
                  next_to_watch        <44>
                  jiffies              <101d5f700>
                  next_to_watch.status <0>
                MAC Status             <40080083>
                PHY Status             <796d>
                PHY 1000BASE-T Status  <3800>
                PHY Extended Status    <3000>
                PCI Status             <10>
[xxxxxx] e1000e 0000:00:1f.6 eno1: Reset adapter unexpectedly
```

```log
[xxxxxx] NETDEV WATCHDOG: CPU: 8: transmit queue 0 timed out xxx ms
```

類似這樣的log在傳輸的過程中會不斷出現，一開始也許重開後還可以連線，但到後來就會直接徹底斷線，無法傳輸。

# 發生原因

把這些log貼到google上做簡單的查詢後就會發現這是一個存在Intel I218-V、I219-V系列網卡上很久的bug了，在使用e1000e driver時，如果有大流量通過就有可能觸發。

這個錯誤我查到最早的記錄在2011年就有了[^1]，我猜測可能是我更新後導致這個問題出現( 目前版本: PVE Version8、Kernel Version 6.8.8 )？不然實在沒有道理之前正常使用，突然就出問題。

根據這幾篇文章[^2][^3][^4]的解釋，問題都指向了網卡開啟的feature。

最有可能造成問題的可能是tso( tcp-segmentation-offload )，其次可能是gso( generic-segmentation-offload )

可以使用`ethtool -k <network interface> | grep ' on'`來檢視網卡開啟了哪些功能。

使用`ethtool -K <network interface> tso off`來將網卡的tso功能關閉，如果網卡關閉後仍然有問題，可以嘗試繼續關閉gso或其他功能，透過測試來判斷到底使哪個功能引起錯誤，只需要關閉對應的功能即可。

# 解法

找到引起錯誤的功能後就需要將設定自動在開機時應用，這一步有很多方法可以用，像是寫一個systemd service，或是其他在開機時執行的方法皆可，不過我習慣將網路設置的部分集中在一起放到`/etc/network/interface`

舉例來說eno1是有問題的網卡，在他底下加上這行`up ethtool -K eno1 tso off`即可在網卡開啟時自動關閉對應的feature

```config
#---
auto eno1
iface eno1 inet manual
#---
        up ethtool -K eno1 tso off
#---
```



## 參考

[^1]: [ubuntu 10.10 - Linux e1000e (Intel networking driver) problem with resume and pcie=off - Server Fault](https://serverfault.com/questions/249080/linux-e1000e-intel-networking-driver-problem-with-resume-and-pcie-off)
[^2]: [118721 – e1000e hardware unit hangs when TSO is on](https://bugzilla.kernel.org/show_bug.cgi?id=118721f)
[^3]: [e1000e eno1: Detected Hardware Unit Hang: | Proxmox Support Forum](https://forum.proxmox.com/threads/e1000e-eno1-detected-hardware-unit-hang.59928/)
[^4]: [e1000 driver hang | Proxmox Support Forum](https://forum.proxmox.com/threads/e1000-driver-hang.58284/)
