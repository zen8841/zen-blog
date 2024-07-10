---
title: Wireguard的keepalive設定
katex: false
mathjax: false
mermaid: false
excerpt: 如何在只有單端能設定PersistentKeepalive下維持連線
date: 2024-07-10 23:28:37
updated: 2024-07-10 23:28:37
index_img:
categories:
- 教程
- 疑難雜症
tags:
- network
- wireguard
---

# 問題

在使用wireguard時會發現一個問題，當連線長時間靜默時，wireguard會自動切斷連線，這時就無法連線到對方。wireguard有提供一個對應的設定值`PersistentKeepalive`，會隔設定的秒數向對方發送心跳包，但如果只在單端設定，wireguard會因為對方長時間沒有回應，認為對方斷線，從而切斷連線。

但是有時候server端自己無法控制，剛好server端又沒有加上`PersistentKeepalive`的設定，此時就要用一些其他的方法來維持連線了。

# 解法

解法其實很簡單，定時向對面發送封包即可，可以將以下指令新增到crontab，記得新增到可以免密碼使用`wg-quick`的user上，或者新增到root的，我是使用* * * * *(每分鐘發送一次ping，如果斷線則重啟對應的wireguard連線)

```bash
ping -c1 -W5 <peer's wireguard IP> 1>/dev/null 2>/dev/null || (sudo wg-quick down <config name> ; sudo wg-quick up <config name>)
```

