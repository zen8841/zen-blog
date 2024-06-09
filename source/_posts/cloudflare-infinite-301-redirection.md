---
title: Cloudflare Proxy 無限301轉址
katex: false
mathjax: false
mermaid: false
excerpt: 曾經架TSCCTF網站時遇到的怪事
date: 2024-06-05 14:17:08
updated: 2024-06-05 14:17:08
index_img:
banner_img: cloudflare-infinite-301-redirection/infinite301.png
categories:
- 教程
- 疑難雜症
tags:
- network
- server
- cloudflare
- https
- 301-redirection
---

# 前情提要

我記得這是在架設TSCCTF 2024網站時發生的，時間距離現在有點久了，細節有點忘了，只是這個錯誤的發生還挺有意思就拿來記錄一下。

# 症狀

在開啟Cloudflare Proxy的狀況下，連接網站時會發生如圖的無限301轉址迴圈，最後因為跳轉過多而停止。

![](infinite301.png)

# 發生原因

這是因為在Cloudflare的Proxy設定中選擇了Flexble，而且同時後端服務器監聽的http會返回301轉址到https造成的（也就是在沒有Proxy的情況下連到http會自動使用https連線）。

![](ssl_policy.png)

因為後端返回的return to https一樣會由Cloudflare Proxy處理，再次去訪問後端的http，然後再次返回301造成無限迴圈

![](why_infinite301.png)

# 解決

只需要將Cloudflare的Proxy設定成Full即可，或是將後端服務器的http改成呈現網頁內容

![](ssl_policy_full.png)

![](solve_infinite301.png)



## 參考

[^1]: [Possible bug - HTTP redirect loop when DNS is proxied - Website, Application, Performance / Security - Cloudflare Community](https://community.cloudflare.com/t/possible-bug-http-redirect-loop-when-dns-is-proxied/206612)
[^2]: [ssl - Nginx configuration leads to endless redirect loop - Stack Overflow](https://stackoverflow.com/questions/4616521/nginx-configuration-leads-to-endless-redirect-loop)