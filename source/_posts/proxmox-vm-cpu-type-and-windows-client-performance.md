---
title: Proxmox中CPU類型對Windows客戶機記憶體性能影響
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - server
  - Proxmox
excerpt: 記錄有關自己的測試結果
date: 2025-05-21 15:24:10
updated: 2025-05-21 15:24:10
index_img:
---


# 前言

前陣子看到了[小林家的白渃](https://blog.bairuo.net/2025/03/03/%e5%85%b3%e4%ba%8eproxmox%e4%b8%adcpu%e7%b1%bb%e5%9e%8b%e5%af%b9windows%e5%ae%a2%e6%88%b7%e6%9c%ba%e7%9a%84%e6%80%a7%e8%83%bd%e5%bd%b1%e5%93%8d/)發布的文章[^1]，根據他的測試，windows客戶機的記憶體性能損失是因為cpu type設為host的原因，我自己按照了類似的流程做了測試，不過得出的結論略有不同，本文為測試結果的記錄。

# 測試

根據小林家的白渃的測試結果，windows客戶機記憶體性能下降的原因是host提供的某些cpu flag可能觸發了windows的安全機制，導致性能下降，因此我也是從cpu flag上開始著手。

## 測試方法

我使用另外一台linux的客戶機來獲得對應cpu type提供的flag，使用以下指令取得。

```shell
$ lscpu | grep Flags | sed -e 's/Flags:                                //g' | tr ' ' '\n' | sort > host-flags
```

這樣只要diff就可以快速得知不同cpu type對支援flag的區別

```shell
$ diff haswell-notsx-flags haswell-notsx-IBRS-flags
24a25,26
> ibpb
> ibrs
```

### 修改PVE提供的CPU flag

編輯`/etc/qemu-server/<id>.conf`，加上`args`這行，host代表使用的cpu model，後面的+、-為在此model上加上或減掉的cpu flag，加上`args: -cpu`後會覆蓋掉cpu設定的部分，使用`qm showcmd <id> --pretty`來查看qemu啟動時的參數，args的參數會被加在最後覆蓋前面設定的參數

```conf
args： -cpu host,-arch-capabilities,-flush-l1d,-pdcm,-pdpe1gb,-ss,-ssbd,-stibp,-tsc_adjust,-umip,-vmx,-vnmi
```

可使用的flag可以在[qemu文檔](https://qemu-project.gitlab.io/qemu/system/qemu-cpu-models.html)[^2]中找到

## 結果

我使用的CPU為2690v3，為Haswell架構，經過測試能使用的CPU model為Haswell-noTSX和Haswell-noTSX-IBRS，無法使用Haswell和Haswell-IBRS，推測是xeon缺少消費級的部分指令集，其中Haswell-noTSX和Haswell-noTSX-IBRS的差別為Haswell-noTSX-IBRS多了兩個和IRBS相關的flag，更接近host的flag因此主要是用這個model和host測試

首先先測試基本model的記憶體性能對比

{% gi 5 3 2 %}

![](host.png)

![](haswell-notsx.png)

![](haswell-notsx-IRBS.png)

![](max.png)

![](x86-64-v2-aes.png)

{% endgi %}

左上：host，中上：haswell-notsx，右上：haswell-notsx-IRBS

左下：max，右下：x86-64-v2-aes

可以看到max和host都有記憶體性能的問題

---

在他的文章總結中認為是md-clear和flush-l1d這兩個flag導致OS啟動漏洞修補影響性能，因此直接測試host在分別減掉這兩個flag後有沒有影響

{% gi 2 2 %}

![](host-no-md-clear.png)

![](host-no-flush-l1d.png)

{% endgi %}

左：無md-clear flag，右：無flush-l1d flag

可以看到記憶體性能還是有問題

---

直接測試host減掉所有比haswell-notsx-IRBS多的flag和把haswell-notsx-IRBS加上所有能加的flag

host減掉所有能減掉的flag後只比haswell-notsx-IRBS多三個flag

```diff
5d4
< arch_perfmon
54d52
< ssbd
60d57
< stibp
```

haswell-notsx-IRBS加上所有能加的flag後只比host少2個flag

```diff
5a6
> arch_perfmon
47a49
> pdcm
```

{% gi 2 2 %}

![](host-no-every.png)

![](haswell-notsx-IRBS-plus-every.png)

{% endgi %}

左：host減掉所有能減的flag，右：haswell-notsx-IRBS加上所有能加的flag

可以看到host的model記憶體性能還是有問題，haswell-notsx-IRBS加上所有flag後還是沒有問題，因此我認為可能和cpu漏洞沒有很大的關係，不然就是和pcdm有關(能源管理部分)

---

最後我還是使用他文章中提供的解決方法，設定一個custom的cpu model，專門用來給windows的虛擬機使用

編輯`/etc/pve/virtual-guest/cpu-models.conf`

```conf
cpu-model: windows-host
    flags +arch-capabilities;+flush-l1d;+md-clear;+pdcm;+pdpe1gb;+ss;+ssbd;+stibp;+tsc_adjust;+umip;+vmx
    phys-bits host
    hidden 0
    hv-vendor-id proxmox
    reported-model Haswell-noTSX-IBRS
```

性能測試如下，沒什麼大問題

![](custom.png)

## 參考

[^1]: [关于Proxmox中CPU类型对Windows客户机的性能影响 - 小林家的白渃小林家的白渃](https://blog.bairuo.net/2025/03/03/%e5%85%b3%e4%ba%8eproxmox%e4%b8%adcpu%e7%b1%bb%e5%9e%8b%e5%af%b9windows%e5%ae%a2%e6%88%b7%e6%9c%ba%e7%9a%84%e6%80%a7%e8%83%bd%e5%bd%b1%e5%93%8d/)
[^2]: [QEMU / KVM CPU model configuration — QEMU  documentation](https://qemu-project.gitlab.io/qemu/system/qemu-cpu-models.html)