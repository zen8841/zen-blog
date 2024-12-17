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
excerpt: 記錄更新R710韌體的採坑過程
date: 2024-12-17 23:59:54
updated: 2024-12-17 23:59:54
index_img:
---


# 前情提要

簡單來說，前陣子在整理機房中的R710，準備重新拿來用，看到很多韌體的版本沒有更新到支援停止時的最新版本，所以就打算更新它的韌體。

# 採坑過程

最初是在某篇Reddit貼文底下發現了這個網站：[updateyodell.net](https://updateyodell.net/)[^1]，他備份了DELL給LifeCycle Controller使用來進行更新的Repository，DELL已經把`ftp.dell.com`上的11代伺服器更新的檔案刪除了。

我直接在開機時按F10進入LifeCycle Controller，並且連接他提供的FTP準備開始進行更新，LifeCycle Controller成功讀到了檔案，並且準備好更新，但當我更新時，等它重開機完發現Raid卡的韌體更新失敗，不更新Raid卡的韌體後變成網卡的韌體更新失敗，如果不更新這兩者，只更新其他內容，雖然不報錯了，但韌體的版本也沒有成功升級。

後來又在另一篇[Reddit貼文](https://www.reddit.com/r/homelab/comments/k839sb/r210ii_update_bios_and_firmware_through_lifecycle/)[^2]中看到貼主提供他用來更新R210韌體的方法，並且他說在11代伺服器上都有用，我嘗試按照他的方式進行，使用Dell EMC Repository Manger來下載更新檔案並製作成升級的ISO，使用此ISO開機後，有看到在進行升級的進度條，但是執行完重開機後，回到LifeCycle Controller，版本還是沒有更新。

最後，我找到另一個網站[dellupdate.t-vault.se](https://dellupdate.t-vault.se/)[^3]，他一樣是使用updateyodell.net提供的FTP來進行更新，但是在更新前先將BIOS和LifeCycle Controller升級到最新，由於我的R710 BIOS已經是最新的，我直接使用他提供的SUU( Server Update Utility ) ISO進行開機，在更新完成後，LifeCycle Controller從1.3.0版本變為最新的1.7.4版本，再重新使用updateyodell.net提供的檔案進行更新，就成功了，也不再報錯，韌體在更新幾次後都來到了最新的版本，可能是舊版的LifeCycle Controller不能去執行這些太新的更新吧。

## 參考

[^1]: [Update Yo Dell, foo! | An FTP server with life cycle repos for G11-G14 Dell PowerEdge Servers](https://updateyodell.net/)
[^2]: [r210ii update bios and firmware through Lifecycle Controller - Pulling my hair out : r/homelab](https://www.reddit.com/r/homelab/comments/k839sb/r210ii_update_bios_and_firmware_through_lifecycle/)
[^3]: [T-Vault Dell Legacy – Escape the Matrix](https://dellupdate.t-vault.se/)