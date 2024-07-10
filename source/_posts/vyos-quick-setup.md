---
title: VyOS快速設定
katex: false
mathjax: false
mermaid: false
excerpt: 簡單介紹如何配置VyOS到能通網路
date: 2024-07-10 17:18:57
updated: 2024-07-10 17:18:57
index_img:
categories:
- 教程
tags:
- linux
- network
- router
---

# 前言

之前在配置家中的網路時，曾經嘗試使用VyOS作為路由器的系統，這篇文章簡單記錄一下安裝及配置的過程。

# 下載

VyOS雖然是開源的系統，但是並未對一般用戶提供LTS版本的ISO下載，只能下載Rolling Release版本的（也就是nightly build），不過我試用的感覺覺得沒什麼差別，應該不至於有穩定性上的問題，相較來說只是測試沒有LTS全面的版本，如果對於Rolling Release版本保有疑慮，也可以考慮自行編譯LTS版本，可以參考這篇[文章](https://docs.vyos.io/en/latest/contributing/build-vyos.html)[^1]

# 安裝

在使用ISO開機後，使用預設帳號登入(vyos/vyos)

![](vyos_login.png)

輸入`install image`後按照提示安裝系統

![](vyos_install.png)

VyOS可以同時安裝多個版本，未來若想更新，可以直接使用新的ISO安裝，或是從已安裝好的系統內添加新系統的entry

# 配置

VyOS的配置有點類似cisco的邏輯，使用set和delete兩個指令來新增或刪除設定，可以使用edit和top來進入下層設定或回到頂層，在configure mode下也可使用run來運行一般模式的指令。

VyOS的shell是bash，不過載入了特製的設定檔，可以直接輸入linux中有的指令來使用，如果想使用bash的補全，可以直接執行`bash`進入常規的bash環境。

首先輸入`configure`進入配置模式

## 系統設定

```bash
# 設定hostname
set system host-name <hostname>
# 設定系統時區
set system time-zone Asia/Taipei
# 建立新的使用者
set system login user <user> authentication plaintext-password <password>
# 刪除預設vyos的帳號
delete system login user vyos
# 設定新帳戶的ssh金鑰
set system login user <user> authentication public-keys <user@host> type <ssh-rsa/ssh-ed25519>
set system login user <user> authentication public-keys <user@host> key <text>
```

## IP Address

### wan

```bash
# 設定外部網路連線
set interface ethernet <wan port> address <dhcp/static ip>
set interface ethernet <wan port> description WAN
# 如果外網連線是static IP，則需設定IP
set interface ethernet <wan port> address <x.x.x.x/x>
set protocols static route 0.0.0.0/0 next-hop <gateway ip>
```

如果是使用pppoe撥號可以參考這篇[文章](https://docs.vyos.io/en/latest/configuration/interfaces/pppoe.html)[^2]

### lan

```bash
# 設定內網IP
set interface ethernet <lan port> address <10.x.x.x/24>
set interface ethernet <lan port> description LAN
```

## 服務

### ssh

```bash
# 設定ssh使用的port
set service ssh port 22
# 設定只允許金鑰登入
set service ssh disable-password-authentication
```

### dhcp

```bash
# 進入dhcp設定層級
edit service dhcp-server shared-network-name <text> subnet <10.x.x.x/24>
# 設定gateway
set option default-router <10.x.x.254> # 前面設定lan口的IP
# 設定dns
set option name-server 1.1.1.1
# 設定內網domain，可跳過
set option domain-name xxx.local
# dhcp租約時間(秒)
set lease 86400
# 設定dhcp ip pool
set range 0 start 10.x.x.2
set range 0 stop 10.x.x.199
# subnet id ，不重複即可
set subnet-id 1
# 如果想要設定static dhcp可以參考底下兩行的範例
set static-mapping <text> ip-address <ip>
set static-mapping <text> mac <mac>
top
```

## nat

```bash
# 進入nat設定層級
edit nat source rule 100
# 設定出口設備
set outbound-interface name eth1
# 設定來源IP(內部IP)
set source address 10.x.x.0/24
# 設定nat
set translation address masquerade
top
```

## firewall

```bash
# 進入防火牆設定層級
edit firewall
# 以組別來管理防火牆規則
set group interface-group WAN interface eth1
set group interface-group LAN interface eth0
set group network-group LAN-v4 network 10.x.x.x/24
# 設定全域規則
set global-options state-policy established action accept
set global-options state-policy related action accept
set global-options state-policy invalid action drop
# 設定從外部往內部轉發時的規則
# 設定一條新的鏈，預設行為丟棄封包
set ipv4 name OUTSIDE-IN default-action drop
# 當符合從外部網路向內部IP路由時，跳到OUTSIDE-IN的鏈上
set ipv4 forward filter rule 100 action jump
set ipv4 forward filter rule 100 jump-target OUTSIDE-IN
set ipv4 forward filter rule 100 inbound-interface group WAN
set ipv4 forward filter rule 100 destination group network-group LAN-v4
# 設定路由器本身的INPUT filter
# 當使用tcp訪問端口22時跳到VyOS_MANAGEMENT鏈上
set ipv4 input filter default-action drop
set ipv4 input filter rule 20 action jump
set ipv4 input filter rule 20 jump-target VyOS_MANAGEMENT
set ipv4 input filter rule 20 destination port 22
set ipv4 input filter rule 20 protocol tcp
# 設定一條新的鏈
# 從內部網路來的連線全部允許
set ipv4 name VyOS_MANAGEMENT rule 10 action accept
set ipv4 name VyOS_MANAGEMENT rule 10 inbound-interface group LAN
# 從wan來的連線，當1分鐘連線3次以上則丟棄封包
set ipv4 name VyOS_MANAGEMENT rule 20 action drop
set ipv4 name VyOS_MANAGEMENT rule 20 recent count 3
set ipv4 name VyOS_MANAGEMENT rule 20 recent time minute
set ipv4 name VyOS_MANAGEMENT rule 20 state new
set ipv4 name VyOS_MANAGEMENT rule 20 inbound-interface group WAN
# 允許從wan來的連線
set ipv4 name VyOS_MANAGEMENT rule 21 action accept
set ipv4 name VyOS_MANAGEMENT rule 21 state new
set ipv4 name VyOS_MANAGEMENT rule 21 inbound-interface group WAN
# 允許回應ping
set ipv4 input filter rule 30 action accept
set ipv4 input filter rule 30 icmp type-name echo-request
set ipv4 input filter rule 30 protocol icmp
set ipv4 input filter rule 30 state new
# 允許本機的所有連線
set ipv4 input filter rule 50 action accept
set ipv4 input filter rule 50 source address 127.0.0.0/8
top
```

## 應用及儲存

```bash
# 應用
commit
# 儲存(寫入到開機設定)
save
# 也可簡化為底下
# commit; save
```



## 參考

[^1]: [Build VyOS — VyOS 1.5.x (circinus) documentation](https://docs.vyos.io/en/latest/contributing/build-vyos.html)
[^2]: [PPPoE — VyOS 1.5.x (circinus) documentation](https://docs.vyos.io/en/latest/configuration/interfaces/pppoe.html)
[^3]: [Quick Start — VyOS 1.5.x (circinus) documentation](https://docs.vyos.io/en/latest/quick-start.html)
