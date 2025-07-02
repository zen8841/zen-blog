---
title: 小米平板1刷機記錄
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - 3C硬體
  - 刷機
excerpt: 記錄小米平板1刷 LineageOS14+16 的過程
date: 2025-03-05 02:47:56
updated: 2025-03-05 02:47:56
index_img:
---


# 前言

{% note info %}

這篇文章思考了一陣子要不要寫，主要是有關小米平板1代刷機的文章已經很多了，我也是完全按照其他人的教學在走，但是考慮到已經一個多月沒更新了，就簡單記錄一下刷機的流程，不過因為當初沒想要寫文章，所以過程中都沒有記錄圖片

{% endnote %}

前陣子手上有一台親戚給的小米平板1，但是有個問題，他們忘了原本登入的小米帳號密碼，也無法找回，因為有帳號登入，就沒辦法進行重置，於是只能靠刷機換系統解決。

# 刷機流程

## 檔案

需要的檔案有以下這些

- [TosForPSCI-0.1.zip](https://androidfilehost.com/?fid=11410932744536994618)： 底層鏡像，解壓後為 tos.img
- [twrp](https://dl.twrp.me/mocha/)： twrp，下載最新版3.4.0就好
- [mocha_repartition_1_2GB_RahulTheVirus.zip](https://androidfilehost.com/?fid=817906626617957830)： 分區合併的工具， mipad1 上的 os 原本採用 A/B 分區模式，但單一系統只有 1GB 的空間，因此需要將 A/B 分區合併為單一系統，才有 2GB 的空間足夠刷入 LineageOS
- [lineage-14.1-20170911-UNOFFICIAL-mocha.zip](https://cloud.mail.ru/public/HgGS/MPAUYzipj)： LineageOS 14.1的系統，不過好像還有人有編譯[更新的](https://androidfilehost.com/?w=files&flid=255835)，我是覺得這個版本已經沒有遇到問題了
- [lineage-16.0-20200419-UNOFFICIAL-mocha.zip](https://androidfilehost.com/?fid=4349826312261776333)： LineageOS 16的系統

Optional

- [miui_MIPADGlobal_V7.5.2.0.KXFMIDE_c4f7d5f70d_4.4.zip](https://bigota.d.miui.com/V7.5.2.0.KXFMIDE/miui_MIPADGlobal_V7.5.2.0.KXFMIDE_c4f7d5f70d_4.4.zip)： 原裝系統，萬一原機的 recovery 沒了可以刷這個恢復
- [open_gapps-arm-7.1-nano-20220215.zip](https://opengapps.org/?api=7.1&variant=nano)： LineageOS 14.1使用的 gapps，我建議使用 nano 即可，他包含了所有 play store 上下載不到的東西，其他需要在下載就好
- [open_gapps-arm-7.1-nano-20220215.zip](https://github.com/MindTheGapps/9.0.0-arm/releases/tag/MindTheGapps-9.0.0-arm-20230922_081043)： LineageOS 16使用的 gapps

## 步驟

1. 重啟進入系統一

關機後按電源鍵+音量上，開機後放開電源但不放音量鍵，進入原裝的 recovery

使用音量鍵選到重啟這個選項，電源鍵確認，選擇重啟到系統一並確認

2. 進入 fastboot

關機後使用電源鍵+音量下進入，或是在系統內連接 adb，並使用 adb 進入

```shell
$ adb reboot fastboot
```

3. 刷入 tos.img

在電腦上使用`fastboot`刷入

```shell
$ fastboot flash tos tos.img
```

4. 刷入 twrp

```shell
$ fastboot flash recovery twrp-3.4.0-0-mocha.img
```

5. 進入 twrp，並準備好檔案

關機後按電源鍵+音量上，進入新的 recovery 即 twrp

可以使用一個 sd 卡裝前面下載的那些 zip 檔，然後裝到平板上，或是提前將這些 zip 檔複製到平板裡

6. 使用 twrp 做4清

選擇 Wipe(在 twrp 中可以使用觸控) > Advanced Wipe，並把最上面四個分區勾選全清空(ART Cache/System/Cache/Data)

7. 合併分區

選擇 Install，找到 mocha_repartition_1_2GB_RahulTheVirus.zip，然後進行安裝，他會將系統的分區合併

8. 刷入系統

選擇 Install，看要刷14或是16，選擇對應的檔案進行安裝，安裝完成後重啟應該就能看到系統了

9. 刷入 gapps(選擇性)

看前面安裝的是14或16，選擇對應的 zip 使用 twrp 一樣刷入即可，不過我建議是如果沒有需求就沒有必要刷，刷完之後感覺續航有變少，而且發熱、卡頓變嚴重

# 總結

其實刷小米平板是真的挺簡單的，幾個步驟走完就完成了，所以當時才沒打算寫，現在寫出來水文章而已

## 參考

[^1]: [[UNOFFICIAL][14.1][7.1.2][2017-09-11] LineageOS 14.1 for Xiaomi MiPad (mocha) | XDA Forums](https://xdaforums.com/t/unofficial-14-1-7-1-2-2017-09-11-lineageos-14-1-for-xiaomi-mipad-mocha.3557616/)
[^2]: [LineageOS 16.0 | 19.04.2020 | Shield blobs | XDA Forums](https://xdaforums.com/t/lineageos-16-0-19-04-2020-shield-blobs.4015965/)
[^3]: [小米平板1刷lineageOS 16.0 教程 - 哔哩哔哩](https://www.bilibili.com/opus/475928169346359980)
[^4]: [旧MI PAD1重获新生-MultiROM为你部署DotOS1.2和lineageOS14.1 多系统刷机篇[ 小米平板1刷安卓多系统]_安卓平板_什么值得买](https://post.smzdm.com/p/awz7wgop/)