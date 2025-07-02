---
title: DELL 11代伺服器韌體更新採坑記錄
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
  - 疑難雜症
tags:
  - 3C硬體
  - server
excerpt: 記錄更新 R710 韌體的採坑過程
date: 2024-12-17 23:59:54
updated: 2024-12-17 23:59:54
index_img:
---


# 前情提要

簡單來說，前陣子在整理機房中的 R710 ，準備重新拿來用，看到很多韌體的版本沒有更新到支援停止時的最新版本，所以就打算更新它的韌體。

# 採坑過程

最初是在某篇 Reddit 貼文底下發現了這個網站：[updateyodell.net](https://updateyodell.net/)[^1]，他備份了 DELL 給 LifeCycle Controller 使用來進行更新的 Repository， DELL 已經把`ftp.dell.com`上的11代伺服器更新的檔案刪除了。

我直接在開機時按 F10 進入 LifeCycle Controller，並且連接他提供的 FTP 準備開始進行更新， LifeCycle Controller 成功讀到了檔案，並且準備好更新，但當我更新時，等它重開機完發現 Raid 卡的韌體更新失敗，不更新 Raid 卡的韌體後變成網卡的韌體更新失敗，如果不更新這兩者，只更新其他內容，雖然不報錯了，但韌體的版本也沒有成功升級。

後來又在另一篇 [Reddit 貼文](https://www.reddit.com/r/homelab/comments/k839sb/r210ii_update_bios_and_firmware_through_lifecycle/)[^2]中看到貼主提供他用來更新 R210 韌體的方法，並且他說在11代伺服器上都有用，我嘗試按照他的方式進行，使用 Dell EMC Repository Manger 來下載更新檔案並製作成升級的 ISO，使用此 ISO 開機後，有看到在進行升級的進度條，但是執行完重開機後，回到 LifeCycle Controller，版本還是沒有更新。

最後，我找到另一個網站 [dellupdate.t-vault.se](https://dellupdate.t-vault.se/)[^3]，他一樣是使用 updateyodell.net 提供的 FTP 來進行更新，但是在更新前先將 BIOS 和 LifeCycle Controller 升級到最新，由於我的 R710 BIOS 已經是最新的，我直接使用他提供的 SUU( Server Update Utility ) ISO 進行開機，在更新完成後， LifeCycle Controller 從1.3.0版本變為最新的1.7.4版本，再重新使用 updateyodell.net 提供的檔案進行更新，就成功了，也不再報錯，韌體在更新幾次後都來到了最新的版本，可能是舊版的 LifeCycle Controller 不能去執行這些太新的更新吧。

## 參考

[^1]: [Update Yo Dell, foo! | An FTP server with life cycle repos for G11-G14 Dell PowerEdge Servers](https://updateyodell.net/)
[^2]: [r210ii update bios and firmware through Lifecycle Controller - Pulling my hair out : r/homelab](https://www.reddit.com/r/homelab/comments/k839sb/r210ii_update_bios_and_firmware_through_lifecycle/)
[^3]: [T-Vault Dell Legacy – Escape the Matrix](https://dellupdate.t-vault.se/)