---
title: 使用 CH341A 燒錄器復活舊主機板
katex: false
mathjax: false
mermaid: false
categories:
  - Tutorial
tags:
  - 3C硬體
  - 刷機
excerpt: 記錄使用燒錄器重新刷 bios 的過程
date: 2026-09-15 00:43:07
updated: 2026-09-15 00:43:07
index_img:
banner_img:
---


前幾天整理一些舊主機板時，測試發現其中一張主機板雖然可以正常 POST，但是 POST 結束後因為斷電後 BIOS 設定丟失，要求按 F1 進入 BIOS。

按下 F1 後，螢幕正常黑掉又重新點亮，接著就一直維持在白屏這個狀態，也就是螢幕有接受到顯示輸出，但是不顯示任何東西，同時在 POST 時可以切換的 NumLock/CapsLock 也在按下 F1 後無法切換。

初步猜測可能是 BIOS 損壞，不過 ASUS 官網上有說明這張 Z97-C 主機板支援 CrashFree BIOS 3，理論上在檢測到 BIOS 損壞時，應該會自動提示插上 USB flash 來重刷 BIOS，不過當時也沒有其他的頭緒，所以決定還是先用燒錄器刷個 BIOS 試試看

# 操作

Z97-C 的 BIOS 是使用 DIP8 封裝的 Flash，連燒錄夾都不用了，可以直接從 IC 座上拆下來夾進燒錄器。

我使用的是這種燒錄器，不過用 CH34* 系列的燒錄器應該都差不多

![](CH341A.png)

燒錄器前半段是用來讀取 SPI BIOS，後半段是 I2C EEPROM，刷 BIOS 的話要插在前半段

Flash 的第一腳要按照圖上所示，放在靠右上的位置，第一腳通常都有一個凹點或是白點絲印做標示

## 軟體

讀取的軟體有很多，還有各種 CLI tools，但為了方便，且 OS 剛好是 Linux，於是決定使用 [IMSProg](https://github.com/bigbigmdm/imsprog)， Windows 上應該也有很多其他種軟體

插上燒錄器後， IMSProg 會顯示 Connected。先用 Detect 讀取 Flash ID，才能使用正確的參數讀取。按 Read 後就可以讀取內容，上方選單可以將讀取到的內容存起來

建議讀取兩次並比較 hash，萬一失敗，可以用備份的內容重新刷回

## BIOS

BIOS 通常都可以在廠商的網站直接下載，少數廠商不會提供 BIOS 的 binary 檔，只給一個 exe，這可能就需要拆 exe 來獲得 binary。雖然 ASUS 有直接提供 binary，不過解壓出來的檔案比 Flash 大了剛好 2 KiB

不過用 Bless Hex Editor 打開後，注意到下載的 BIOS 中間和從 Flash dump 出來的內容有一段相同的標記。

![從 Flash dump 出來的 BIOS](bios_dump.png)

![從 ASUS 下載的 BIOS](bios_download.png)

於是就用這個作為標記，把下載的 BIOS 前段刪除，讓這個標記處在相同位置。刪除完後，剛好檔案大小相同。

用修改過的 BIOS 刷入後，成功開機

![運氣不錯](Z97-C.jpg)
