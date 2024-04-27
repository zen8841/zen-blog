---
title: 在EdgeOS和RouterOS之間建立GRE over IPsec tunnel
katex: false
mathjax: false
mermaid: false
excerpt: 介紹如何在兩個系統間建立安全的GRE tunnel
date: 2024-04-25 00:04:43
updated: 2024-04-25 00:04:43
index_img:
categories:
- 教程
tags:
- network
- UBNT
- Ubiquiti
- EdgeOS
- Mikrotik
- RouterOS
- GRE over IPsec
- IPsec
---

# 介紹

一般來說建立GRE over IPsec tunnel有三種方法[^1]

- GRE over IPsec transport mode
- GRE over IPsec tunnel mode
- Virtual Tunnel Interface (VTI)

不過RouterOS好像到現在都還不支持VTI，因此本文中不會討論

![](ipsec_enc.jpg)

## GRE over IPsec transport mode

GRE over IPsec transport mode簡單來說就是建立一個GRE tunnel，並設定IPsec transport mode保護在兩個peer中傳輸的封包內容，ESP只會保護Data段的內容，因此IP Header之類的資訊還是可以被得知

這種方式缺點是萬一兩邊的IPsec並沒有成功建立、或是連線中斷，由於GRE tunnel是直接使用對方的public IP連線，有可能GRE tunnel會直接在public internet中裸奔，且兩邊都必須有public IP，不能位於NAT後方。

## GRE over IPsec tunnel mode

GRE over IPsec tunnel mode則是先在兩個peer間建立IPsec tunnel，接著使用IPsec tunnel中的IP去建立GRE tunnel，此時GRE tunnel的整個封包都會被ESP給包住，因此達到加密的效果。

在IPsec連線未成功建立或中斷時，GRE tunnel將無法通訊，這既是缺點也是優點，可以防止通訊裸露在public internet中，但是由於過了兩個tunnel，因此mtu會較低，可能會對速度造成影響

# 操作

網路拓樸如下，請自行替換對應的IP

|              | **public IP(WAN IP)** | **tunnel interface IP** |   **內部網路**   |
| :----------: | :-------------------: | :---------------------: | :--------------: |
| **RouterOS** |      203.0.113.1      |       10.2.1.1/30       | 192.168.1.254/24 |
|  **EdgeOS**  |      203.0.113.2      |       10.2.1.2/30       | 192.168.2.254/24 |

![](network_topology.png)

## GRE over IPsec transport mode

待更新

也許可以先參考DN42發布的這篇文章[^2]研究看看，Ubiquiti Help Center中並沒有IPsec transport mode的相關文件

## GRE over IPsec tunnel mode

在這部份中，將會在兩個peer上建立loopback interface，IPsec tunnel將設定成只會轉發對方loopback interface 的IP

下表是示範時使用的IP，如果要使用其他IP，請替換對應的位置

|              | **loopback interface IP** |
| :----------: | :-----------------------: |
| **RouterOS** |        10.1.1.1/32        |
|  **EdgeOS**  |        10.1.1.2/32        |

### IPsec tunnel
此部份大部分參考這些文件[^3][^4]，但在此之上做了一些小修改

#### RouterOS

##### IP -> IPsec -> Profiles -> new

Profile 是IKE phase1協商的參數

![](routeros/ipsec_new_profile.png)

{% fold info @更改EdgeOS端Encryption的參數 %}
如果想更改EdgeOS端Encryption的參數，要在這裡修改對應的參數

![](routeros/ipsec_profile_encryption_change.png)

{% endfold %}

##### IP -> IPsec -> Peers -> new

![](routeros/ipsec_new_peer.png)

##### IP -> IPsec -> Identities -> new

![](routeros/ipsec_new_indentities.png)

先不用在意底下的警告，等最後再回來確認就好

##### IP -> IPsec -> Proposals -> new

![](routeros/ipsec_new_proposal.png)

{% fold info @更改EdgeOS端DH Group的參數 %}
如果想更改EdgeOS端DH Group的參數，要在這裡修改對應的參數

![](routeros/ipsec_proposal_dhgroup_change.png)

{% endfold %}

##### 建立loopback interface

由於RouterOS沒有loopback interface一類的設備[^5][^6]，因此創建loopback interface的方法會是建立一個空的bridge，然後給予這個bridge一個IP

###### Bridge -> Bridge -> new

![](routeros/new_bridge_general.png)

![](routeros/new_bridge_stp.png)

其他設定保持默認就好

###### IP -> Addresses -> new

![](routeros/loopback_interface_ip.png)

##### IP -> IPsec -> Policies -> new

![](routeros/ipsec_new_polices_general.png)

![](routeros/ipsec_new_polices_action.png)

如果要轉發多個子網的話Action部分的Level要設定為unique，不過此次設定只需要轉發兩端router的IP就好

##### IP -> Firewall -> NAT -> new

![](routeros/firewall_nat_rule_general.png)

![](routeros/firewall_nat_rule_action.png)

要將此條規則移至最上方

#### EdgeOS

##### 創建loopback interface

```shell
set interfaces loopback lo address 10.1.1.2/32
```

##### 設定IPsec Site to Site VPN

VPN頁面 -> IPsec Site-to-Site

![](edgeos_ipsec_vpn.png)

完成這些設定後，兩端的IPsec tunnel應該就架好了，可以嘗試互相ping看看對方的loopback inerface IP，使用RouterOS ping時記得要指定來源IP為loopback inerface IP

在IPsec 的Active Peers和Installed SAs應該就能看到東西出現了

### GRE tunnel

#### RouterOS

##### Interfaces -> GRE Tunnel -> new

![](routeros/gre_new_interface.png)

設定成功後應該會顯示mtu的資訊

![](routeros/gre_interface_status.png)

##### IP -> Addresses -> new

![](routeros/gre_interface_ip.png)

##### IP -> Routes -> new

![](routeros/gre_new_routes.png)

#### EdgeOS[^7]

```shell
configure
set interfaces tunnel tun0 local-ip 10.1.1.2
set interfaces tunnel tun0 remote-ip 10.1.1.1
set interfaces tunnel tun0 encapsulation
set interfaces tunnel tun0 encapsulation gre
set interfaces tunnel tun0 address 10.2.1.2/30
set protocols static interface-route 192.168.1.0/24 next-hop-interface tun0
commit ; save
```

完成這些設定後，在兩個peer間就建立了GRE over IPsec tunnel

## 參考

[^1]: [ GRE Over IPsec for Secure Tunneling : VyOS Support Portal ](https://support.vyos.io/support/solutions/articles/103000096326-gre-over-ipsec-for-secure-tunneling)
[^2]: [howto/EdgeOS GRE IPsec Example](https://dn42.eu/howto/EdgeOS-GRE-IPsec-Example)
[^3]: [How to create an IPsec VPN between Unifi USG and Mikrotik firewalls](https://github.com/iisti/how-to-usg-mikrotik-ipsec-vpn)
[^4]: [EdgeRouter to MikroTik IPSec VPN](https://docs.google.com/document/d/13WTT3wZgejNWP0EPNeKQ24XXCVFM3g521doVrzeHqtg/edit)
[^5]: [Dummy or Loopback interface](https://forum.mikrotik.com/viewtopic.php?t=108227)
[^6]: [loopback interface in mikrotik](https://forum.mikrotik.com/viewtopic.php?t=187390)
[^7]: [EdgeRouter - GRE Tunnel ](https://help.ui.com/hc/en-us/articles/205231690-EdgeRouter-GRE-Tunnel)
[^8]: [Manual:IP/IPsec - MikroTik Wiki](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec)
[^9]: [EdgeRouter - EoGRE Layer 2 Tunnel](https://help.ui.com/hc/en-us/articles/204961754-EdgeRouter-EoGRE-Layer-2-Tunnel)
[^10]: [GRE over IPSec tunnel](https://community.ui.com/questions/GRE-over-IPSec-tunnel/7f1bc5c4-ba65-4c6f-a8f5-7c2fd3135693)
[^11]: [EdgeRouter - Modifying the Default IPsec Site-to-Site VPN](https://help.ui.com/hc/en-us/articles/216771078-EdgeRouter-Modifying-the-Default-IPsec-Site-to-Site-VPN#3)