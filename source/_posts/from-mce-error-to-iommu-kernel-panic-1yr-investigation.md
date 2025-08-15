---
title: '從 MCE Error 到 IOMMU: 追查 Kernel panic 一年的真相'
banner_img: /from-mce-error-to-iommu-kernel-panic-1yr-investigation/banner.png
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
  - 疑難雜症
tags:
  - Proxmox
  - linux
  - 3C硬體
excerpt: '介紹這個長達一年 Debug 的故事: 關於我是如何從與實際錯誤毫無關係的記錄中推斷出真相'
date: 2025-08-15 04:32:17
updated: 2025-08-15 04:32:17
index_img:
---


# 前言

這篇文章主要是記錄這一年來斷斷續續逐漸 Debug 的過程，怎麼從一個幾乎和 IOMMU 沒有關係的 Kernel panic 錯誤記錄，到最終找到是由 IOMMU 引起的問題，也許這篇文章會給一些遇到類似問題的人一些啟發。

# 過程

## 前情提要

機房內的 PVE Cluster 為3台 Cisco UCS C240 M3 組成，其中一台和另外兩台的硬體略有不同

- 一台
  - CPU: E5-2650v1
  - NIC: Mellanox Technologies MT26448 [ConnectX EN 10GigE, PCIe 2.0 5GT/s] (rev b0)
  - Raid Card: Broadcom / LSI MegaRAID SAS 2208 [Thunderbolt] (rev 03)
- 兩台
  - CPU: E5-2620v1
  - NIC: Mellanox Technologies MT27500 Family [ConnectX-3]
  - Raid Card: Broadcom / LSI MegaRAID SAS 2008 [Falcon] (rev 03)

## 惡夢的開始

故事要從一年前說起...

在2024年的9月底，我例行完成了社團機房 PVE Cluster 的更新工作，在逐節點重啟時，重啟到第二台發現機器無法正常重開

![重啟到 Pentium 時發現無法上線](cluster.png)

打開 CIMC(Cisco Integrated Management Controller，~~就是 IPMI~~)提供的 KVM console，發現出現了 MCE Error，並且好像這個 MCE Error 造成了 Kernel panic

![Kernel panic](kvm-1.png)

並且 CIMC 也出現了 Moderate Fault 的記錄

![CIMC 的錯誤記錄](cimc.png)

在回到更新前的核心 6.5.13-6-pve 後，發現可以正常重開機

作為臨時的 work around 就先使用`apt-mark`來避免 Kernel 因更新被刪除，和編輯`/etc/default/grub`來固定使用這個 Kernel 進行開機

```shell
# apt-mark hold proxmox-kernel-6.5.13-6-pve-signed
# vim /etc/default/grub
# # 將GRUB_DEFAULT修改，可從/boot/grub/grub.cfg中得到entry的名字或id，'>'為進入子目錄的開機項目
# # GRUB_DEFAULT="Advanced options for Proxmox VE GNU/Linux>Proxmox VE GNU/Linux, with Linux 6.5.13-6-pve"
# # 或是修改成id
# # GRUB_DEFAULT="gnulinux-advanced-d6bc077e-b89d-4562-b438-be1df5a36ca9>gnulinux-6.5.13-6-pve-advanced-d6bc077e-b89d-4562-b438-be1df5a36ca9"
```

不過後來發現 Proxmox 有提供更好的固定 Kernel 方式

```shell
# pve-efiboot-tool
USAGE: /usr/sbin/pve-efiboot-tool <commands> [ARGS]

  /usr/sbin/pve-efiboot-tool format <partition> [--force]
  /usr/sbin/pve-efiboot-tool init <partition> [grub]
  /usr/sbin/pve-efiboot-tool reinit
  /usr/sbin/pve-efiboot-tool clean [--dry-run]
  /usr/sbin/pve-efiboot-tool refresh [--hook <name>]
  /usr/sbin/pve-efiboot-tool kernel <add|remove> <kernel-version>
  /usr/sbin/pve-efiboot-tool kernel pin <kernel-version> [--next-boot]
  /usr/sbin/pve-efiboot-tool kernel unpin [--next-boot]
  /usr/sbin/pve-efiboot-tool kernel list
  /usr/sbin/pve-efiboot-tool status [--quiet]
  /usr/sbin/pve-efiboot-tool help
```

使用`pve-efiboot-tool`即可輕鬆固定 Kernel，而且這個工具也會自動建立`/etc/default/grub.d/proxmox-kernel-pin.cfg`來選擇固定的核心開機，就不需要前面的工作了

```shell
# pve-efiboot-tool kernel list
Manually selected kernels:
None.

Automatically selected kernels:
6.11.11-2-pve
6.14.8-2-pve
6.5.13-6-pve
# pve-efiboot-tool kernel pin 6.5.13-6-pve
# pve-efiboot-tool kernel list
Manually selected kernels:
None.

Automatically selected kernels:
6.11.11-2-pve
6.14.8-2-pve
6.5.13-6-pve

Pinned kernel:
6.5.13-6-pve
```

於是就暫時透過固定 Kernel 開機先修復了問題

但是，誰也沒想到，這個核心將會在這次更新後繼續使用320天...

## 初步診斷

在把三個 Node 都更新並重啟完後發現，其中有兩個 Node 都發生了 Kernel panic 的狀況

猜猜是哪兩台？

沒錯，剛好就是[#前情提要](#前情提要)中硬體相同的那兩台

螢幕上顯示 CPU 相關的錯誤， CIMC 也說 Processors 發生錯誤，又剛好只有在 E5-2620 的節點上發生，這種種跡象似乎暗示著什麼，<sub>~~顯然暗示錯了(當時還沒注意到 Raid 卡和網卡也有不同)~~</sub>

因此第一個推測出現了， CPU 出問題了，或是 CPU 有不相容，實際上 E5-2650 和 E5-2620 的架構的確略有差異，當時猜測可能是因為這個原因引起更新到新核心才出的問題

根據 MCE Error 和 Kernel panic 的資訊下去搜尋，也有查到似乎可能和架構有關係(圖2、圖3)

{% gi 3 3 %}

![](discord-chat-1.png)

![](discord-chat-2.png)

![](discord-chat-3.png)

{% endgi %}

後來經過幾天的討論，決定直接採購7顆 E5-2650v2，幫所有 Node 都一起升級 CPU，~~反正 E5 二代已經是白菜價了，7顆加運費也才1290~~

![<del>過了一個月才買</del>](discord-chat-4.png)

## 再次診斷

又過了一段時間， CPU 到貨了，並且在1月多時喬時間進了機房換 CPU，其實換 CPU 時也發生了不少狀況，不過不在這篇文章的範圍，就跳過了。

至於結果？

當然是沒有成功，~~CPU 換上去之後也到禍了~~，不然也不會有之後後續的 Debug 和這篇文章了。

![慘，還是有問題](discord-chat-5.png)

這直接的代表最初的猜測是錯誤的，並沒有找到導致 Kernel panic 的真正原因。

又經過了一段時間的除錯，發現使用 Recovery mode 開機時會顯示較多的除錯訊息，我注意到 Kernel log 中多了有關`pcieport 0000:00:02.0`的錯誤訊息。

{% gi 2 2 %}

![](discord-chat-6.png)

![](discord-chat-7.png)

{% endgi %}

使用`lspci`查看 host 上的 PCIE 設備後發現`00:02.0`是一個 PCI bridge，這東西是在 CPU 內的，不然就是主板上的南橋，我的直覺認為這個設備應該沒有問題。

{% note light %}
`00:01.0`\~`00:03.0`和`80:01.0`\~`80:03.0`都是 PCI bridge，我猜測相隔這麼遠應該是因為這台是雙路伺服器，兩顆 CPU 的 PCI bridge 分配的 PCI ID 沒有相連。
{% endnote %}

```shell
# lspci -nn
00:00.0 Host bridge [0600]: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 DMI2 [8086:0e00] (rev 04)
00:01.0 PCI bridge [0604]: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 PCI Express Root Port 1a [8086:0e02] (rev 04)
00:01.1 PCI bridge [0604]: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 PCI Express Root Port 1b [8086:0e03] (rev 04)
00:02.0 PCI bridge [0604]: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 PCI Express Root Port 2a [8086:0e04] (rev 04)
00:02.2 PCI bridge [0604]: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 PCI Express Root Port 2c [8086:0e06] (rev 04)
00:03.0 PCI bridge [0604]: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 PCI Express Root Port 3a [8086:0e08] (rev 04)
...
04:00.0 Ethernet controller [0200]: Mellanox Technologies MT27500 Family [ConnectX-3] [15b3:1003]
...
80:01.0 PCI bridge [0604]: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 PCI Express Root Port 1a [8086:0e02] (rev 04)
80:02.0 PCI bridge [0604]: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 PCI Express Root Port 2a [8086:0e04] (rev 04)
80:03.0 PCI bridge [0604]: Intel Corporation Xeon E7 v2/Xeon E5 v2/Core i7 PCI Express Root Port 3a [8086:0e08] (rev 04)
...
81:00.0 RAID bus controller [0104]: Broadcom / LSI MegaRAID SAS 2008 [Falcon] [1000:0073] (rev 03)
...
```

不過雖然認為 PCI bridge 沒有問題，但是錯誤日誌都已經寫了 pcieport 了，我就往 PCIE 的部分開始調查，就發現了三台 Node 的配置不止是 CPU 有不同，連 Raid 卡和網卡也有不同(圖一)

使用`lspci -t`來查看 PCIE 的連接拓撲，在正常的機器上(celeron)發現 Raid 卡(`82:00.0`)是接在`80.01.0`2a 通道的 PCI bridge 底下，而網卡(`04:00.0`)則是接在另一顆 CPU 2a 通道的 PCI bridge(`00:02.0`)下(圖二)

而不正常的機器的拓樸也幾乎相同，除了 Raid 卡被分配到的 ID 不同外，其他都相同(圖三)

{% gi 3 3 %}

![](discord-chat-8.png)

![](discord-chat-9.png)

![](discord-chat-10.png)

{% endgi %}

看完拓樸後就會發現前面開機時 Kernel log 中出現的`pcieport 0000:00:02.0`正是連接網卡的 PCI bridge。

到這裡已經越來越接近真相了，這些證據指向可能是 Mellanox 的 driver: mlx4_core 在更新中出了問題

於是嘗試了`blacklist mlx4_core`並`update-initramfs`後使用新的核心開機，果然成功開機。

![blacklist mlx4_core](discord-chat-11.png)

因為屏蔽了`mlx4_core`後立刻就恢復正常，我當時過於果斷的認為就是`mlx4_core`的問題，甚至忽略了之後測試時，使用 grml 的 kernel-6.11 是可以正常開機，更新後的`mlx4_core`其實還能運作。

由於必須要使用這張網卡，當時也暫時解決不了`mlx4_core`的問題，於是只能繼續 pin kernel，並期待在未來的更新中會修復。

然而這次的誤判影響比第一次更大，第一次判斷錯誤，在更換 CPU 後發現故障依舊就能發現自己的判斷錯誤，而這次的判斷錯誤幾乎沒有被修正的機會，每次更新核心若是失敗都會覺得是正常的，可能只是還沒修復而已。

就這樣，核心從6.8到6.11又到了6.13，我都尚未發現其實並不是`mlx4_core`的問題，直到...

## 最終診斷

俗話說的好，事不過三；在這次的診斷，我想我應該是找到真相了。

前幾天幫自己的幾個 PVE lab 更新到 PVE 9 測試完成後，就決定來把社團的 Cluster 也一起升級了

在這次更新完後，6.14的 Kernel 還是依然沒能解決問題

![問題依舊](kvm-2.png)

不過到了8月有點時間，我打算趁著更新順便再仔細的調查看看

接續上次的調查進度，我將`mlx4_core` blacklist 後使用 Recovery mode 開機，果然還是正常開機

正當我認為問題就是這樣，上游尚未解決 Kernel 的問題，準備要把`mlx4_core`的 blacklist 刪除時，我注意到 console 中出現了很多 Kernel message 截斷了輸出，使用`journalctl -k`來仔細的觀察看看

![DMAR 的錯誤？？](kvm-3.png)

在我的印象中之前好像從來沒有出現這麼多 DMAR(DMA remapping) 的錯誤過

這時，一個不知道從哪來的靈感忽然擊中我的腦袋，我突然想起之前設置 [PCIE Passthrough](/pci-passthrough-with-proxmox/) 時也要檢查 DMAR，這一般是和 IOMMU 一起進行設定的，並且好像 PVE 的核心有設定編譯的參數預設打開 IOMMU，`CONFIG_INTEL_IOMMU_DEFAULT_ON=y`(參考[在 Proxmox 上進行 PCI-E 直通](/pci-passthrough-with-proxmox/#基礎設定)中基礎設定的第一段)

於是我檢查了機器裡不同核心上與 IOMMU 有關的編譯設定

![Kernel config](kernel-parameter.png)

直到 6.5 之前都沒有打開的`CONFIG_INTEL_IOMMU_DEFAULT_ON`，到了6.8和之後被打開了

{% gi 2 2 %}

![李組長眉頭一皺，發現事情並不單純](not-simple.gif)

![好像有什麼關聯？](discord-chat-12.png)

{% endgi %}

出於懷疑，我把`intel_iommu=off`加到了 Kernel parameter 中，並`update-grub`後重新開機

正如所料，關閉 IOMMU 後正常開機

![成功開機](kvm-4.png)

~~最後，慶祝我晉升資深開關工程師，能熟練操作多種開關~~

![<del>晉升資深開關工程師</del>](discord-chat-13.png)

# 總結

其實這個問題在網路上找不到解法的其中一個原因是因為關鍵詞用錯了，我一開始都是使用 MCE Error、Kernel panic、mlx4 這些關鍵詞進行搜尋，幾乎搜尋不到和遇到的環境接近的狀況。

在解決問題後，我使用`mt27500 iommu kernel panic`做為關鍵詞才找到了一篇在 [Proxmox forum 上的貼文](https://forum.proxmox.com/threads/random-6-8-4-2-pve-kernel-crashes.145760/)有相關的內容，~~都知道 IOMMU 的問題了才找到有什麼用~~

不過測試用`proxmox 6.8 mt27500 kernel panic`做為關鍵詞也能找到這篇貼文，~~只能說當時搜尋關鍵詞的能力不到位~~，用其他不包括 IOMMU 關鍵詞組合基本很難找到這篇貼文

至於 IOMMU 為什麼會和 ConnectX-3 發生衝突的原因還是未知，貼文內有提到說更新 BIOS 可能會有幫助，但 UCS C240M3 早就 EOL 了，我們也已經更新到最新的版本，更不用說還有一台其實是正常的。

至於 ConnectX-3 上的 firmware 我也已經更新到最新，看[貼文內有一個有11台機器的用戶](https://forum.proxmox.com/threads/random-6-8-4-2-pve-kernel-crashes.145760/post-659209)，其中只有3台使用 ConnectX-3 的機器發生了 Kernel crash，但是看他的 crash.log 和我們發生的情況也並不相同，可能真的就是 ConnectX-3 與某些東西發生衝突了。

這次 Debug 比較困難的原因主要是引起問題的 Kernel 編譯設定`CONFIG_INTEL_IOMMU_DEFAULT_ON`被打開後，引起的問題五花八門，基本很難用對應問題的錯誤訊息去找到相關的回答，但是這些問題最後往往會引導到 Kernel panic，反而直接查詢像 Kernel panic 這種大範圍的關鍵詞才能找到資訊，和過往 Debug 的經驗不太相同。
