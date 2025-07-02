---
title: 在 EdgeOS 和 RouterOS 之間建立 GRE over IPsec tunnel
katex: false
mathjax: false
mermaid: false
excerpt: 介紹如何在兩個系統間建立安全的 GRE tunnel
date: 2024-04-25 00:04:43
updated: 2024-09-01 16:15:02
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

{% note info %}

2024/9/1更新： 更新 GRE over IPsec transport mode 的設定

{% endnote %}

# 介紹

一般來說建立 GRE over IPsec tunnel 有三種方法[^1]

- GRE over IPsec transport mode
- GRE over IPsec tunnel mode
- Virtual Tunnel Interface (VTI)

不過 RouterOS 好像到現在都還不支持 VTI，因此本文中不會討論

![](ipsec_enc.jpg)

## GRE over IPsec transport mode

GRE over IPsec transport mode 簡單來說就是建立一個 GRE tunnel，並設定 IPsec transport mode 保護在兩個 peer 中傳輸的封包內容。

這種方式缺點是兩邊都必須有 public IP，不能位於 NAT 後方。

## GRE over IPsec tunnel mode

GRE over IPsec tunnel mode 則是先在兩個 peer 間建立 IPsec tunnel，接著使用 IPsec tunnel 中的 IP 去建立 GRE tunnel，此時 GRE tunnel 的整個封包都會被 ESP 給包住，因此達到加密的效果。

這種方式建立的 tunnel 封包對多一個 IPsec tunnel 的 IP header， MTU 會比較低，如果可以，使用 transport mode 會是更好的選擇。

## IPsec

在設定 IPsec 的過程中會看到很多參數像 sha1、aes256、DH group14、modp2048 或是其他類似的，這些參數在不同設定間有些重複，有些又不同，很容易讓人搞混，其實 IPsec 的主要部分按照我的理解可以分成 4 個部分

- Phase 1 設定： 負責身分驗證，成立才會進行 Phase 2
  - 在 EdgeOS 中這部份的設定叫 ike-group
  - 在 RouterOS 中這部份的設定是 Profile
- Phase 2 設定： 負責加密的設定
  - 在 EdgeOS 中這部份的設定叫 esp-group
  - 在 RouterOS 中這部份的設定是 Proposals
- Peer 設定： 定義對方的端點之類的資訊
  - 在 EdgeOS 中這部份的設定和 Policy 一起包在 site-to-site/peer 裡面
  - 在 RouterOS 中這部份的設定在 Peers 和 Identities 裡
- Policy 設定： 定義要對哪些封包做 IPsec 加密/解密
  - 在 EdgeOS 中這部份的設定和 Peer 一起包在 site-to-site/peer 裡面
  - 在 RouterOS 中這部份的設定是 Policy

剛剛提到的那些參數就是在 Phase 1 和 Phase 2 設定中會碰到的，其實只要記住一個原則就是兩邊的設定要相同即可，這樣連線應該就能正常起來，而 DH group/modp/pfs 其實是同樣的東西，不過叫法會不同，可以查到[對應表](https://www.watchguard.com/help/docs/help-center/en-US/Content/en-US/Fireware/bovpn/manual/diffie_hellman_c.html)

# 操作

網路拓樸如下，請自行替換對應的 IP

|              | **public IP(WAN IP)** | **tunnel interface IP** |   **內部網路**   |
| :----------: | :-------------------: | :---------------------: | :--------------: |
| **RouterOS** |      203.0.113.1      |       10.2.1.1/30       | 192.168.1.254/24 |
|  **EdgeOS**  |      203.0.113.2      |       10.2.1.2/30       | 192.168.2.254/24 |

![](network_topology.png)

## GRE over IPsec transport mode

EdgeOS 的部分是參考 DN42 發布的這篇文章[^2]，在 Ubiquiti Help Center 中並沒有 IPsec transport mode 的相關文件

### EdgeOS

EdgeOS 部分的設定我使用設定檔來呈現，和指令是對應的

```conf
interfaces {
    tunnel tun0 {
        address 10.2.1.2/30
        description TO-ROS
        encapsulation gre
        local-ip 203.0.113.2
        mtu 1400
        multicast disable
        remote-ip 203.0.113.1
        ttl 255
    }
}
vpn {
    ipsec {
        auto-firewall-nat-exclude disable
        esp-group ESP-AES256-SHA1-DH14-TRANSPORT {
            compression disable
            lifetime 3600
            mode transport
            pfs dh-group14
            proposal 1 {
                encryption aes256
                hash sha1
            }
        }
        ike-group IKE-AES256-SHA1-DH14 {
            lifetime 28800
            proposal 1 {
                dh-group 14
                encryption aes256
                hash sha1
            }
        }
        ipsec-interfaces {
            interface eth1
        }
        site-to-site {
            peer 203.0.113.1 {
                authentication {
                    mode pre-shared-secret
                    pre-shared-secret ****************
                }
                connection-type initiate
                default-esp-group ESP-AES256-SHA1-DH14-TRANSPORT
                ike-group IKE-AES256-SHA1-DH14
                local-address 203.0.113.2
                tunnel 0 {
                    allow-nat-networks disable
                    allow-public-networks disable
                    esp-group ESP-AES256-SHA1-DH14-TRANSPORT
                    protocol gre
                }
            }
        }
    }
}
```

### RouterOS

**IP -> IPsec -> Profiles -> new**

Profile 是 IKE phase1 協商的參數，基本上設定都可以和 EdgeOS 的部分對應到

![](transport_mode/ipsec_new_profile.png)

**IP -> IPsec -> Peers -> new**

![](transport_mode/ipsec_new_peer.png)

**IP -> IPsec -> Identities -> new**

這裡我是使用 PreshareKey 來驗證對方，雖然使用 RSA key 或證書來驗證更安全，但是 EdgeOS 那邊的 RSA key 格式是 plain RSA 格式，要和 PEM 格式相互轉換很麻煩

![](transport_mode/ipsec_new_indentities.png)

**IP -> IPsec -> Proposals -> new**

![](transport_mode/ipsec_new_proposal.png)

**IP -> IPsec -> Policies -> new**

Protocal 47 就是代表 GRE

![](transport_mode/ipsec_new_polices_general.png)

![](transport_mode/ipsec_new_polices_action.png)

**Interfaces -> GRE Tunnel -> new**

因為另一邊 MTU 設 1400 這邊也設同樣的值，不過不一定要 1400，可以稍大一點像 1420 ，因為 GRE header 佔 24Byte， ESP header 佔 50～60Byte，大概 1420 也可以

這裡的 local IP 和 remote IP 直接填 Public IP 就行， Policy 會攔截對應的封包加上 IPsec

![](transport_mode/gre_new_interface.png)

**IP -> Addresses -> new**

![](transport_mode/gre_interface_ip.png)

這樣就完成設定了，可以嘗試看看 ping 對方的 GRE tunnel 的 IP 了

## GRE over IPsec tunnel mode

在這部份中，將會在兩個 peer 上建立 loopback interface， IPsec tunnel 將設定成只會轉發對方 loopback interface 的 IP

下表是示範時使用的 IP，如果要使用其他 IP，請替換對應的位置

|              | **loopback interface IP** |
| :----------: | :-----------------------: |
| **RouterOS** |        10.1.1.1/32        |
|  **EdgeOS**  |        10.1.1.2/32        |

### IPsec tunnel
此部份大部分參考這些文件[^3][^4]，但在此之上做了一些小修改

#### RouterOS

**IP -> IPsec -> Profiles -> new**

Profile 是 IKE phase1 協商的參數

![](tunnel_mode/ipsec_new_profile.png)

{% fold info @更改 EdgeOS 端 Encryption 的參數 %}
如果想更改 EdgeOS 端 Encryption 的參數，要在這裡修改對應的參數

![](tunnel_mode/ipsec_profile_encryption_change.png)

{% endfold %}

**IP -> IPsec -> Peers -> new**

![](tunnel_mode/ipsec_new_peer.png)

**IP -> IPsec -> Identities -> new**

![](tunnel_mode/ipsec_new_indentities.png)

先不用在意底下的警告，等最後再回來確認就好

**IP -> IPsec -> Proposals -> new**

![](tunnel_mode/ipsec_new_proposal.png)

{% fold info @更改 EdgeOS 端 DH Group 的參數 %}
如果想更改 EdgeOS 端 DH Group 的參數，要在這裡修改對應的參數

![](tunnel_mode/ipsec_proposal_dhgroup_change.png)

{% endfold %}

**建立 loopback interface**

由於 RouterOS 沒有 loopback interface 一類的設備[^5][^6]，因此創建 loopback interface 的方法會是建立一個空的 bridge，然後給予這個 bridge 一個 IP

**Bridge -> Bridge -> new**

![](tunnel_mode/new_bridge_general.png)

![](tunnel_mode/new_bridge_stp.png)

其他設定保持默認就好

**IP -> Addresses -> new**

![](tunnel_mode/loopback_interface_ip.png)

**IP -> IPsec -> Policies -> new**

![](tunnel_mode/ipsec_new_polices_general.png)

![](tunnel_mode/ipsec_new_polices_action.png)

如果要轉發多個子網的話 Action 部分的 Level 要設定為 unique，不過此次設定只需要轉發兩端 router 的 IP 就好

**IP -> Firewall -> NAT -> new**

![](tunnel_mode/firewall_nat_rule_general.png)

![](tunnel_mode/firewall_nat_rule_action.png)

要將此條規則移至最上方

#### EdgeOS

**創建 loopback interface**

```shell
set interfaces loopback lo address 10.1.1.2/32
```

**設定 IPsec Site to Site VPN**

VPN 頁面 -> IPsec Site-to-Site

![](tunnel_mode/edgeos_ipsec_vpn.png)

完成這些設定後，兩端的 IPsec tunnel 應該就架好了，可以嘗試互相 ping 看看對方的 loopback inerface IP，使用 RouterOS ping 時記得要指定來源 IP 為 loopback inerface IP

在 IPsec 的 Active Peers 和 Installed SAs 應該就能看到東西出現了

### GRE tunnel

#### RouterOS

**Interfaces -> GRE Tunnel -> new**

![](tunnel_mode/gre_new_interface.png)

設定成功後應該會顯示 mtu 的資訊

![](tunnel_mode/gre_interface_status.png)

**IP -> Addresses -> new**

![](tunnel_mode/gre_interface_ip.png)

#### EdgeOS[^7]

```shell
configure
set interfaces tunnel tun0 local-ip 10.1.1.2
set interfaces tunnel tun0 remote-ip 10.1.1.1
set interfaces tunnel tun0 encapsulation
set interfaces tunnel tun0 encapsulation gre
set interfaces tunnel tun0 address 10.2.1.2/30
commit ; save
```

完成這些設定後，在兩個 peer 間就建立了 GRE over IPsec tunnel

## 設定路由

完成 tunnel 的設定後要加上路由才能讓內網的連通到另一個內網

### RouterOS

**IP -> Routes -> new**

![](tunnel_mode/gre_new_routes.png)

### EdgeOS

```shell
configure
set protocols static interface-route 192.168.1.0/24 next-hop-interface tun0
commit ; save
```

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

