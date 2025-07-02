---
title: Asus X99 通病
katex: false
mathjax: false
mermaid: false
excerpt: 分享一下該如何檢測主板的狀態，以及如何修復
date: 2024-05-06 15:45:42
updated: 2024-05-06 15:45:42
index_img: 
categories:
- 教程
- 疑難雜症
tags:
- 3C硬體
---

# 前情提要

簡單來說，當時我正在讓服務器跑從遠端備份回來文件的哈希來驗證完整性，結果回頭一看服務器，發現就關機了。

# 症狀

此時 CPU 右上角的紅色 LED 亮起，按開機鍵都無反應，斷電重新上電紅色 LED 仍然點亮，這個時候我的第一猜測是可能是 CPU 過熱，也許讓他散熱一下就能好了。

在過一陣子重新上電後，紅色 LED 不亮了，現在按開機鍵則變成整台機器的 LED 瞬間閃爍一次，主板偵錯碼(Q-code)也是和 LED 一樣瞬間閃00後，之後按開機鍵也無任何反應，不過將電源供應器重新上電後又會有同樣的行為。

# 偵錯

在經過 Google 搜尋 X99 的這些症狀後，我最後定位到了一個可能的問題——華碩 X99 通病(MOS 擊穿燒 CPU)

{% fold @網路上其他人遇到的狀況 %}

[ASUS X99-DELUXE A-II Failed | TechPowerUp Forums](https://www.techpowerup.com/forums/threads/asus-x99-deluxe-a-ii-failed.262537/)[^1]

[ASUS X99 A-II Code 00 (Not Listed) Red CPU Light On | Overclock.net](https://www.overclock.net/threads/asus-x99-a-ii-code-00-not-listed-red-cpu-light-on.1615149/)[^2]

[X99 A II will not POST!! : r/buildapc](https://www.reddit.com/r/buildapc/comments/5sywuu/x99_a_ii_will_not_post/)[^3]

[Asus X99-A II suddenly shutdown and now giving Q code 00 | Tom's Hardware Forum](https://forums.tomshardware.com/threads/asus-x99-a-ii-suddenly-shutdown-and-now-giving-q-code-00.2915943/)[^4]

{% endfold %}

# 檢測

那麼，對電腦的錯誤有了猜測，接下來就是檢測了

## 主機板

在開始檢測前可以先看底下這篇文章來大致了解主板供電的組成

[【科普】主機板供電能力解析](https://forum.gamer.com.tw/C.php?bsn=60030&snA=512806)[^5]

{% fold @CPU 供電針腳定義 %}

![](cpu_power_pin.png)

{% endfold %}

首先將三用電表打到測通檔(二極體檔)，量測 CPU 供電 12V 和電感下端

12V 對電感下端短路，這邊就基本可以宣告 CPU 死定了，12V 直衝 CPU

{% gi 2 2 %}

![](mb_cpu_12V_test1.jpg)

![](mb_cpu_12V_test2.jpg)

{% endgi %}

同時 12V 也對地短路了

這也可以解釋為什麼之前開機一瞬間燈會閃一下，但之後按開機卻再也沒有反應，直到斷電重來。

主板在開機按下時短接 PS_ON 和 GND，拉低 PS_ON，電源啟動，輸出 12V，但是 CPU 供電短路，電源供應器自動保護斷電，之後也不開啓，直到斷電重置。

{% fold @主機板供電針腳定義 %}

![](mb_power_pin.jpg)

{% endfold %}

現在知道主機板有短路後，就繼續排查哪裡出了問題，首先拆下供電散熱檢查底下的 MOS

{% gi 3 3 %}

![](mb_mos.jpg)

![](mos.jpg)

![](mos_datasheet.png)

{% endgi %}

MOS 的型號是 0812ND，查看 datasheet 可以知道這是一顆集成上下管的 IC。

需要量測的就是這顆 IC 裡的上下管正不正常。

首先量測上管，也就是1腳和2/3/4任一腳(Gate 和 Drain，圓點位置代表第一腳)，發現右數第三顆的阻值不對(左圖)，右圖則是正常的 MOS 應該有的讀值。

{% gi 2 2 %}

![](mos_test_bad.jpg)

![](mos_test_good.jpg)

{% endgi %}

這代表第三顆 MOS 的上管擊穿了直接短路， Vphase(Source)在底下連到電感，12V 直接導到 CPU。

順便量測了所有的下管，結果都是正常。

理論上只要更換壞掉的 MOS 應該就能修好這塊主板，等之後有時間再來修。

## CPU

雖然前面已經猜測這顆 CPU 死定了，不過不看到確切的結果還是不會認命

首先要找到2011-3腳位的定義，不過幸好已經有人幫我們畫出來了

[Intel Socket 2011v3 Land Grid Layout. (Thanks sdgus68!) - computer hardware post - Imgur](https://imgur.com/gallery/intel-socket-2011v3-land-grid-layout-thanks-sdgus68-12PjUu8)[^6]

[LGA2011-3 Land-Layout : r/intel](https://www.reddit.com/r/intel/comments/lpp83e/lga20113_landlayout/)[^7]

我是使用第一份，我覺得那比較直觀

{% fold @第一份使用的參考文件 %}

Imgur 原文有提供下載連結，但是有一個已經失效了，可以從[這裡](http://szarka.ssgg.sk/Vyuka/2014/HighEnd-LGA2011/core-i7-lga2011-3-tmsdg.pdf)觀看，或是在底下下載

[core-i7-lga2011-3-tmsdg.pdf](core-i7-lga2011-3-tmsdg.pdf) 的52頁有針腳編號的定義

[core-i7-lga2011-3-datasheet-vol-1.pdf](core-i7-lga2011-3-datasheet-vol-1.pdf) 的64頁開始有編號對應的用途

{% endfold %}

![](2011-3_layout.png)

量測 V<sub>CC</sub> 和 V<sub>SS</sub> 的阻值，可以理解為供電和 GND，發現短路，徹底給 CPU 判死刑

<video controls width="500">
  <source src="cpu_test.webm" type="video/webm" />
  <source src="cpu_test.mp4" type="video/mp4" />
</video>

## 參考
[^1]: [ASUS X99-DELUXE A-II Failed | TechPowerUp Forums](https://www.techpowerup.com/forums/threads/asus-x99-deluxe-a-ii-failed.262537/)
[^2]: [ASUS X99 A-II Code 00 (Not Listed) Red CPU Light On | Overclock.net](https://www.overclock.net/threads/asus-x99-a-ii-code-00-not-listed-red-cpu-light-on.1615149/)
[^3]: [X99 A II will not POST!! : r/buildapc](https://www.reddit.com/r/buildapc/comments/5sywuu/x99_a_ii_will_not_post/)
[^4]: [Asus X99-A II suddenly shutdown and now giving Q code 00 | Tom's Hardware Forum](https://forums.tomshardware.com/threads/asus-x99-a-ii-suddenly-shutdown-and-now-giving-q-code-00.2915943/)
[^5]: [【科普】主機板供電能力解析 @電腦應用綜合討論 哈啦板 - 巴哈姆特](https://forum.gamer.com.tw/C.php?bsn=60030&snA=512806)
[^6]: [Intel Socket 2011v3 Land Grid Layout. (Thanks sdgus68!) - computer hardware post - Imgur](https://imgur.com/gallery/intel-socket-2011v3-land-grid-layout-thanks-sdgus68-12PjUu8)
[^7]: [LGA2011-3 Land-Layout : r/intel](https://www.reddit.com/r/intel/comments/lpp83e/lga20113_landlayout/)