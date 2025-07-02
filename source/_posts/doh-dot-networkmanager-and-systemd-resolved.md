---
title: DoH、DoT、Networkmanager 與 systemd-resolved 的設定
katex: false
mathjax: false
mermaid: false
excerpt: 分享有關最近調整 DNS 設定的一些經驗
date: 2024-05-17 11:47:27
updated: 2024-06-09 22:24:00
index_img:
categories:
- 教程
tags:
- linux
- network
- dns
---

{% note info %}

2024/6/9更新： 有關 Networkmanager 的自動化腳本

{% endnote %}

# 前言

一開始只是想給在用的筆電加上 DNS over HTTPS 或是 DNS over TLS 而已，後來又逐漸冒出很多奇怪(?)的需求，因此稍微研究了一下 linux 上的 DNS 管理機制。

# 需求

首先是 DNS 使用 DNS over HTTPS/DNS over TLS，但是在特定的網路連接下會自動切換成對應的 DNS，忽略不信任網路下 DHCP 提供的 DNS。

# 實現

## DNS over HTTPS[^1]

這個算是最簡單的一項了，基本只要照著 ArchWiki 的操作執行就好

```shell
# # 確認沒有端口使用localhost:53
# ss -lp 'sport = :domain'
# # 安裝dns-over-https
# pacman -S dns-over-https
# # 啟動dns-over-https服務
# systemctl enable --now doh-client.service
```

設定檔在`/etc/dns-over-https/doh-client.conf`但是基本不用更改，預設的 DoH server 就是 Cloudflare 的

接著只要將 DNS server  改成`127.0.0.1`就完成 DNS over HTTPS 的設定了

```shell
# echo 'nameserver 127.0.0.1' > /etc/resolv.conf
```

## Network manager

但是這樣就會出現一個問題， Network manager 在連上一個新的網路時，如果 IP 是設定使用 DHCP，而非 DHCP(Address only) 或是其他選項的話，啟動連接時將會將新的 nameserver 寫入`/etc/resolv.conf`覆蓋。

![](networkmanager_ipv4_setting.png)

不過在查看`NetworkManager.conf(5)`後可以發現， Network manager 只會在`/etc/NetworkManager/NetworkManager.conf`中沒有設定`dns`這個選項或是設定成`dns=default`時且`/etc/resolv.conf`為一般的文件而非連結檔，才會修改`/etc/resolv.conf`。

讀到這裡，其實就可以知道怎麼解決問題了

```shell
# echo 'nameserver 127.0.0.1' > /etc/resolv-manual.conf
# ln -sf /etc/resolv-manual.conf /etc/resolv.conf
```

~~當然你也可以用`chattr`給`/etc/resolv.conf`指定不可寫入的屬性，不過這樣設定就有點僵硬了~~

{% fold @Network manager 的一些其他行為 %}

如果`dns`選項沒有指定值的話，那麼預設是會使用`default`，除非`/etc/resolv.conf`是個連結到

`/run/systemd/resolve/stub-resolv.conf, /run/systemd/resolve/resolv.conf, /lib/systemd/resolv.conf, /usr/lib/systemd/resolv.conf`的連結檔，這時`dns`選項會使用`systemd-resolved`，如果`/etc/resolv.conf`是個連結到其他地方的連接檔，那麼則是會使用`dns=none`

{% endfold %}

## systemd-resolved[^2]

不過這樣設定的話就會造成一個問題，雖然連接的網路是什麼不會影響 DNS server 的選擇了，但也永遠釘死在`127.0.0.1`的 DNS over HTTPS 上，除非手動修改，但是需求之一就是 Network manager 在切換連接時可以自動修改 DNS，而非每次都要手動修改。

```shell
# # 開啟systemd-resolved，會與resolvconf衝突
# systemctl enable --now systemd-resolved
```

systemd-resolved 會提供部分`resolvconf`的功能，同時，啟用後會提供`resolvectl`。

在啟用服務後會在`127.0.0.53`和`127.0.0.54`上開啟 DNS 服務，當訪問 systemd-resolved 的 DNS server 時，則是會使用在`/etc/systemd/resolved.conf`設定的 DNS 服務器和自身緩存做解析。

{% fold @systemd-resolved 的四種模式 %}

systemd-resolved 有四種模式 stub、static、uplink、foreign，這取決於`/etc/resolv.conf`的狀態

如果`/etc/resolv.conf`是連結到`/run/systemd/resolve/stub-resolv.conf`，那麼 systemd-resolved 就會是 stub 模式

如果`/etc/resolv.conf`是連結到`/lib/systemd/resolv.conf, /usr/lib/systemd/resolv.conf`，那麼 systemd-resolved 就會是 static 模式

如果`/etc/resolv.conf`是連結到`/run/systemd/resolve/stub-resolv.conf`，那麼 systemd-resolved 就會是 uplink 模式

如果`/etc/resolv.conf`是一般檔案或是連接到其他地方，那麼 systemd-resolved 就會是 foreign 模式

這些檔案的內容有興趣可以去翻翻看，不過推薦是使用 stub 模式

{% endfold %}

```shell
# # 使用stub模式
# ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

這個時候使用`resolvectl status`就可以看到在 stub 模式了

這樣目前的設定就是`/etc/resolv.conf` -> `/run/systemd/resolve/stub-resolv.conf`，內容應該是`nameserver 127.0.0.53`然後底下有一些其他選項。

systemd-resolved 會根據 DNS Domain 決定 DNS request 要往哪個 DNS server 詢問，如果沒有設定 DNS Domain 則是使用預設，也就是`/etc/systemd/resolved.conf`裡設定的

這樣只要在 Network Manager 設定 DNS Domain 為`~.`或是想要解析的內部域名，在連上網路時會反映到 systemd-resolved 上，就可以使用特定的 DNS 了

### 設定

修改`/etc/systemd/resolved.conf`設定檔

```conf
[Resolve]
# 上游使用前面設定的DoH server
DNS=127.0.0.1
```

目前系統訪問 DNS 的實際流程就會是`/etc/resolv.conf` -> `127.0.0.53:53` -> `127.0.0.1:53` -> `https://cloudflare-dns.com/dns-query`，但是這樣感覺有點太複雜了，如果只是想要加密 DNS，那麼 DoT 也可達成，而且 systemd-resolved 支援 DoT 。

## DNS over TLS[^2]

只需要修改`/etc/systemd/resolved.conf`即可完成

```conf
[Resolve]
# 使用Cloudflare的DNS
DNS=1.1.1.1
# 如果DNSOverTLS設定為on的話，如果你想使用的其他DNS不支援DoT，那麼systemd-resolved將不會發出詢問
# 使用opportunistic則是會在上游支援時使用，如果不支援則使用原本模式
DNSOverTLS=opportunistic
```

那麼系統訪問 DNS 的實際流程就會變為`/etc/resolv.conf` -> `127.0.0.53:53` -> `1.1.1.1:853`，減少了一個流程，結構變得簡單不少

## Networkmanager 的自動化腳本

前面有提到設定 DNS domain 為`~.`可以讓網路介面的 DNS server 成為全域 DNS，但是在接上 VPN 時，不知道為什麼 systemd-resolve 會不吃 DNS domain 設定為`~.`的網路介面的資訊，`resolvectl`直接顯示這個介面沒有 DNS 資訊。

於是我就換了一種方法來實現，介面的 DNS domain 可以設定為其他值，或不設定，而是在啟動任何介面時檢查我想要更改 DNS domain 的介面是否開啟，如果開啟就直接調用`resolvectl`來更改設定。

設定只需要編輯一個檔案`/etc/NetworkManager/dispatcher.d/set_<interface>_dns_domain.sh`，並給他加上執行權限`chmod +x /etc/NetworkManager/dispatcher.d/set_<interface>_dns_domain.sh`

```bash
#!/bin/sh

if nmcli connection show "<connection name>" | grep 'GENERAL.STATE'; then
    echo "change <interface name> dns domain"
    resolvectl domain <interface name> '~.'
fi
```

`<connection name>`為 Networkmanager 中連線的名字，`<interface name>`則是網路介面的名字。

Networkmanager 的自動化腳本也可以做更多其他的事，像是偵測到物理連接斷開後自動開啟無線網路[^3]。

## 總結

其實這樣做還是會有一個漏洞，如果你不信任的 DHCP server 有設定 DNS domain，那麼 systemd-resolved 一樣會吃到這個設定，解決的方法就是把 Network Manager 的那個網路連接設定為使用 DHCP(Address only) ，但是這樣就要手動設定，有點不爽。

最後總結：如果你真的在一個不信任的網路環境中，那麼我還是推薦你使用 VPN 吧，這篇的 DNS 設定其實幫不了你多少忙。

## 同場加映 -- nsswitch

全名是 name service switch configuration，簡單來說就是配置系統在查詢各項東西的時候要去哪裡查，像是 passwd 就是用戶驗證，設定為使用檔案(也就是/etc/passwd)，或者是使用 systemd 驗證，如果想要串 LDAP 之類的身分驗證，那麼就要在後面加上對應的 Plugin

```conf
passwd: files systemd
group: files [SUCCESS=merge] systemd
shadow: files systemd
gshadow: files systemd

publickey: files

hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns
networks: files

protocols: files
services: files
ethers: files
rpc: files

netgroup: files
```

其中的`hosts`選項就是要怎麼解析 domain 的設定，`mymachines`是檢查內部容器或虛擬機有沒有對應的域名， resolve 是 systemd-resolved，除非 systemd-resolved 無法使用才會進入下一個選項(`[!UNAVAIL=return]`，有四種狀態，有興趣可以去看`nsswitch.conf(5)`)， files 是`/etc/hosts`， myhostname 是本機的 hostname，最後 dns 就是按照`/etc/resolv.conf`的設定去查詢。

## 參考

[^1]: [DNS-over-HTTPS - ArchWiki](https://wiki.archlinux.org/title/DNS-over-HTTPS)
[^2]: [systemd-resolved - ArchWiki](https://wiki.archlinux.org/title/Systemd-resolved#DNS_over_TLS)
[^3]: [用 NM-dispatcher 实现 WiFi 开关的自动控制 - sbw Blog](https://blog.sbw.so/u/nm-dispatcher-auto-switch-between-wifi-ethernet.html)