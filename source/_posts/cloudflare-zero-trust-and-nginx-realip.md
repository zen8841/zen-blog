---
title: Cloudflare Zero Trust 與 Nginx 還原真實 IP
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - network
  - server
  - Cloudflare
  - Nginx
  - Zero Trust
  - cloudflared
  - WARP
  - Cloudflare Proxy
excerpt: 簡介與教學 Cloudflare Zero Trust 服務及如何在使用 Nginx 的狀況下還原真實的客戶端 IP
date: 2025-06-30 23:13:34
updated: 2025-06-30 23:13:34
index_img:
banner_img: cloudflare-zero-trust-and-nginx-realip/cloudflare_tunnel_http.webp
---


# 前言

最近在嘗試 Cloudflare Tunnel 的功能，順便整個6月因為考試完全沒產文章出來，就寫一下這篇記錄。

# 簡介

Cloudflare Zero Trust 的一部分前身是來自 Argo Tunnel，現在的 Cloudflare Tunnel 域名還是使用`cfargotunnel.com`，不過我也沒在 Argo Tunnel 的時期使用過這部份服務，所以就不深入了。

Cloudflare Zero Trust 提供了多種服務，在這篇文章中主要是聚焦於內網穿透功能的部分，不過會在簡介中介紹其他功能。這些服務都是基於和 Cloudflare 建立的 Tunnel 來實現， Cloudflare 提供了幾種方式和它建立連接[^1]。

## Tunnel

- cloudflared
- WARP
- WARP Connector
- Magic WAN

### cloudflared

在使用內網穿透功能中主要使用的 Tunnel，比較像是作為服務端，連上 Cloudflare 網路來提供服務，但也不止可以提供內網穿透的功能，可以透過 cloudflared 實現 WARP 提供的部分功能。

### WARP

就是 WARP，前陣子很流行的 " Cloudflare VPN "，主要作用是用來做為客戶端連上 Cloudflare 的網路，不過在設定中也可以開啟選項[^2]，讓 Cloudflare 給你一個 CGNAT 裡的 IP，其他同一個 Zero Trust Team 內的客戶端可以在連上 Cloudflare 網路後碰到開啟 WARP 的客戶端(透過 IPv4 或 IPv6)。

### WARP Connector

在我的理解裡這東西就是在 Linux Router 上安裝的 WARP[^3]，他可以把以他為 Gateway 的網路通通發到 Cloudflare 網路中，相當於整個子網都開了 WARP，看起來還可以作為 Site to Site VPN 來用(但好像用其他的 Tunnel 組合也可以實現)，不過我沒試過這東西，目前還在測試版，開啟了 Magic WAN 的帳戶無法使用 WARP Connector。

![WARP Connector](warp_connector.webp)

### Magic WAN

這個是 Enterprise-only 的功能，所以我更沒用過了，看起來就是提供更多種連接到 Cloudflare 方式的服務，包括使用 GRE、IPsec  和上面提到 WARP、Cloudflare Tunnel 來連到 Cloudflare[^4]， Cloudflare 似乎還有賣專門跑 Magic WAN 的硬體，不過也可以在一般的 x86 中使用。

## 服務

我將 Cloudflare Zero Trust 提供的服務簡單分為兩種

- 可被 Public Network 直接訪問的服務
- 需接入 Cloudflare Network 才能連線的服務

### Public Network Access Service

這部份的服務都是透過 HTTP based 來呈現，也是此篇教學會介紹的部分。

需要使用 Cloudflared 或 WARP Connector 來將服務連線到 Cloudflare Network，由於流量是從內部向 Cloudflare 發起，一般不會被防火牆擋住，因此其中一個用途就是前文提到的可用來做內網穿透，但是只能穿透幾種服務， HTTP(S)、SSH、RDP，後面兩種都是只能使用瀏覽器連線 Cloudflare 網頁來提供 Webshell，或是在網頁上呈現 RDP 的內容。流程大概是像這樣[^5]，服務不一定要開在有 cloudflared(WARP Connector) 的機器上，只要可以被 cloudflared(WARP Connector) 訪問到就可以。

![Cloudflare Tunnel 的代理流程](cloudflare_tunnel_http.webp)

這些服務雖然說是被 Public Network 訪問，但也可在 Cloudflare dash board 設定 access list，讓只有指定的 email 才能透過一次性郵件驗證登入。

### Cloudflare Network Access Service

這部分的服務就比較沒有限定要用哪種方式連到 Cloudflare 前面提到的建立 Tunnel 的方法應該都可以，具體流程如下， client 透過 WARP、cloudflared 或是在 Gateway 上的 WARP Connector、Magic WAN 連上 Cloudflare 網路，就可以存取另一端透過類似方法連線上 Cloudflare 網路的資源(需要在同一個 Zero Trust team 中， WARP 是用前面說的 email 一次性郵件驗證登入)。

![通過 Cloudflare 網路中轉的服務](connect_private_ip.webp)

如果在 Cloudflare dash board 上設定了 Private IP Routing，就可以用 WARP+cloudflared 實現 VPN 的功能， Cloudflare 網路會將設定的 Private IP 路由到內部網路的 cloudflared 上，來連線內部網路的 IP，其他種 Tunnel 組合也可以達成相同的功能。

Cloudflare Zero Trust 還可以設定對 WARP 客戶端的權限，可以設定讓 WARP 無法被客戶端手動關閉，所有流量通過 Cloudflare，方便企業級用戶對工作用裝置進行流量審計。

# 教學

## 建立 Tunnel

在 Cloudflare dash board 上開啟 Zero Trust 服務後，便會有一個自己的 Zero Trust team，選擇網路 > Tunnel 後就可以開始建立 Tunnel

{% gi 3 2 1 %}

![](create_tunnel_1.png)

![](create_tunnel_2.png)

![](create_tunnel_3.png)

{% endgi %}

可以取一個可以識別 cloudflared 位置的名字

![](create_tunnel_4.png)

我個人喜歡用 Docker 架設 cloudflared，可以視自己情況選擇合適自己的平台

![](create_tunnel_5.png)

## 設定穿透

到這裡 Tunnel 已經建立成功了，最後這裡便是設定要穿透哪些服務的部分，可以之後再繼續補充設定。

![](create_tunnel_6.png)

上排的子網域和主機名稱就是可以被 Public Network 訪問到的端點，需要至少有一個在 Cloudflare 上的網域。

下排的服務則是穿透的目標，需要是可以被 cloudflared 碰到的位置，像是這個示範就是把在 localhost 443 port 的 HTTPS 穿透出去，讓他可以在`https://test.zenwen.tw`被訪問到。

服務的類型也有前面說的 SSH、RDP 之類的類型選擇，可以自行嘗試。

### 代理到 Nginx

如果是想穿透到內部的 Nginx 上，為了要拿到正確的 TLS certificate，需要在 TLS 的部分設定要代理的域名，這樣 Nginx 才會返回正確的證書和網頁，如果是代理 HTTP 或是設定忽略證書，可以改為設定 HTTP 設定中的 HTTP 主機標頭(Host Header)的部分

![](create_tunnel_nginx.png)

## Nginx real IP

如果使用了 Cloudflare 提供的 Proxy 或是 Zero Trust 的內網穿透服務，則內網的 Nginx 會收到的 IP 就會是 Cloudflare CDN 或 安裝有 cloudflared 機器的 IP，如果想要在 log 或其他記錄中記載真實的客戶端 IP，就需要使用`ngx_http_realip_module`[^7]。

![RealIP 的工作方式](restore_realip.webp)

這個模組的使用很簡單，只有3種指令[^8]

- set_real_ip_from： 設定當收到哪些 IP 發來請求時要使用`real_ip_header`內的 IP 來替換來源 IP
- real_ip_header： 設定真實 IP 是放在哪個 Header 裡，一般的反向代理會放`X-Real-IP`或`X-Forwarded-For`，不過在 Cloudflare 這個案例中也可以使用`CF-Connecting-IP`
- real_ip_recursive： 當 real_ip_header 裡有多個 IP 時是否要遞歸查詢直到找到不在 set_real_ip_from 的 IP 清單中

將這些指令放入`nginx.conf`的 http 區段中即可

```conf
http {
    include /etc/nginx/global/cloudflared_realip.conf;
    include /etc/nginx/global/cloudflare_v4_realip.conf;
    include /etc/nginx/global/cloudflare_v6_realip.conf;
    real_ip_header CF-Connecting-IP;
    #real_ip_header X-Forwarded-For;
    real_ip_recursive on;
}
```

在`cloudflared_realip.conf`中填入`set_real_ip_from <cloudflared IP>;`，來將 cloudflared 傳來的請求使用 realip 模組。

Cloudflare CDN 的 IP 需要定期更新，分別在 https://www.cloudflare.com/ips-v4 和 https://www.cloudflare.com/ips-v6 ，我使用 crontab 來在每週一自動更新。

```cron
0 0 * * 1 /bin/curl -s -w '\n' https://www.cloudflare.com/ips-v4 | /bin/awk '{print "set_real_ip_from " $1 ";"}' > /etc/nginx/global/cloudflare_v4_realip.conf
0 0 * * 1 /bin/curl -s -w '\n' https://www.cloudflare.com/ips-v6 | /bin/awk '{print "set_real_ip_from " $1 ";"}' > /etc/nginx/global/cloudflare_v6_realip.conf
```

## 參考

[^1]: [Private networks · Cloudflare Zero Trust docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/private-net/)
[^2]: [Create private networks with WARP-to-WARP · Cloudflare Zero Trust docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/private-net/warp-to-warp/)
[^3]: [WARP Connector · Cloudflare Zero Trust docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/private-net/warp-connector/)
[^4]: [Overview · Cloudflare Magic WAN docs](https://developers.cloudflare.com/magic-wan/)
[^5]: [Cloudflare Tunnel · Cloudflare Zero Trust docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
[^6]: [Connect private networks · Cloudflare Zero Trust docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/private-net/cloudflared/)
[^7]: [Restoring original visitor IPs · Cloudflare Support docs](https://developers.cloudflare.com/support/troubleshooting/restoring-visitor-ips/restoring-original-visitor-ips/#nginx-1)
[^8]: [Module ngx_http_realip_module](https://nginx.org/en/docs/http/ngx_http_realip_module.html)
