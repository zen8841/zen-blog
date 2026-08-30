---
title: zswap 與 zram 的設定方式與參數調整
katex: false
mathjax: false
mermaid: false
categories:
  - Tutorial
tags:
  - Linux
  - Arch
excerpt: 介紹在 ArchLinux 上設定這兩者的方式
date: 2026-08-30 20:32:27
updated: 2026-08-30 20:32:27
index_img:
banner_img:
---


# 前言

前陣子看了 Chris Down 的[這篇文章](https://chrisdown.name/2026/03/24/zswap-vs-zram-when-to-use-what.html)，決定把自己用了很久的 zram 換成 zswap，順便記錄一下兩者的設定方式

{% note info %}
如無必要，建議不要使用 zram，詳見 Chris Down 的文章。zswap 實際上就包含了使用 zram 作為 swap 的概念，且 zswap 與記憶體管理子系統關係更密切，可以實現類似於加上緩存的三層結構

不使用 zram 作為 swap 最主要的原因，是 zram 的 swap 優先級總是較高。一開始產生的 cold page 會優先寫入 zram，持續佔用壓縮後的記憶體空間。等到記憶體出現壓力時，由於 zram 已滿，又會將較熱的頁面寫入實體 swap，使原先想實現的多層 swap 架構失敗。(不過我個人是沒遇到過啦，我的 zram 開的很激進，和實體 RAM 1:1，所以實際上從來沒有填滿 zram 過)
{% endnote %}

# zswap

Arch 官方的核心都預設開啟了 zswap，只需要按自己需求調整參數即可，我建議是調`max_pool_percent`即可

可以調整的參數有幾個：

- shrinker_enabled: 預設開啟，當 zswap 中有大量冷數據時，會將這些冷數據寫入實體 swap，節省記憶體

- max_pool_percent: 預設為20，代表 zswap 壓縮後的資料最多可以佔用記憶體的幾 %。我建議可以提高到 40～60。我本身使用 zram 的時候就比較激進，平均壓縮比我猜測應該2.5x，開40%與實體 RAM 為 1:1 的 zram 應該差不多(還在測試中)

- compressor: 預設 zstd，壓縮算法，若要修改，建議在 lz4 與 zstd 之間二選一。建議不用改，要改還需要修改 initramfs 的設定(在 mkinitcpio 的 modules 中提早載入，或是其他的設定檔，看使用的 initramfs 生成器)，不然改這個參數會變成先用 zstd 開機，開機後才改

- accept_threshold_percent: 預設為90，為了避免在記憶體高佔用的時候 zswap 收縮，把數據擠到 RAM 內讓壓力更大，當 zswap 達到容量上限後，必須等到使用量下降至設定的百分比，才會繼續接收寫入 zswap 的頁面。

## 調整方式

這些參數都需要設定在 kernel cmdline 上

systemd-boot:

`/boot/loader/entries/arch.conf`

```conf
options zswap.max_pool_percent=40
```

grub:

`/etc/default/grub`

```grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet ... zswap.max_pool_percent=40"
```

---

如果其他 distro 的 kernel 沒有預設開啟 zswap，可以使用 `zswap.enabled=1` 啟用。

不過需要確認 kernel 是否已編譯 zswap 支援，不然可能需要用`modules-load.d`去載入(大部分 distro 應該都已經編譯進核心了)

可以用指令確認目前的核心是否有編譯並預設開啟

```shell
$ zgrep -E "CONFIG_ZSWAP" /proc/config.gz
```

`CONFIG_ZSWAP=y`代表有編譯進核心，`CONFIG_ZSWAP_DEFAULT_ON=y`代表預設開啟

---

[如何 disable writeback](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Disable_zswap_writeback_to_use_the_swap_space_only_for_hibernation): 簡單來說就是不寫入實體 swap，實體 swap 僅用於 hibernate，或是在啟用 zswap 時作為後端 swap 空間

## 其他參數

我還有調整一些 sysctl 中 vm 的相關參數，參考 zram 的建議設定進行調整

`/etc/sysctl.d/99-vm-zswap-paRAMeters.conf`

```conf
vm.swappiness = 100
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
#vm.page-cluster = 0
```

考慮到是使用 SSD，`swappiness`設為100，如果是使用 HDD 做 swap，可以設為預設的60應該不錯

`watermark_boost_factor`和`watermark_scale_factor`和 zram 的建議值一樣，因為 zswap 和 zram 都是在記憶體中壓縮數據

注釋掉了`page-cluster`回到預設的3，因為會有實際寫入實體 swap 的狀況，一次讀寫較大的 chunk

# zram

需要先載入 zram module

`/etc/modules-load.d/zram.conf`

```conf
zram
```

建立 zram 的方式有很多種，我比較喜歡使用 udev 規則建立，其他方式可以看 [wiki 上的介紹](https://wiki.archlinux.org/title/zram)

`/etc/udev/rules.d/99-zram.rules`

```conf
KERNEL=="zram0", ATTR{comp_algorithm}="lz4", ATTR{disksize}="4G", RUN="/usr/bin/mkswap /dev/zram0", TAG+="systemd"
```

將 zram 掛載為 swap

`/etc/fstab`

```conf
# /dev/zram0
/dev/zram0  none    swap    defaults,pri=16383  0   0
```

sysctl 參數調整

`/etc/sysctl.d/99-vm-zram-paRAMeters.conf`

```conf
vm.swappiness = 180
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
vm.page-cluster = 0
```

## 參考

[^1]: [zswap - ArchWiki](https://wiki.archlinux.org/title/Zswap)
[^2]: [zram - ArchWiki](https://wiki.archlinux.org/title/zram)
