---
title: Proxmox Network： OVS 與 SDN
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - Proxmox
  - network
  - linux
  - server
excerpt: 分享設定 OVS 和 SDN 的過程中的理解
date: 2024-12-16 21:36:51
updated: 2024-12-16 21:36:51
index_img:
---


# 前言

這篇算是整理我設定 CCNS 的 PVE 集群的一些經驗，還有對於 OVS 與 SDN 的理解

# OVS

Open vSwitch 我記得是在 PVE v6 還是 v7 的時候加入的，如果對於 vmbr 的需求只有傻瓜交換機的功能，那麼使用 OVS 和 Linux Bridge 體驗是相同的，不過如果有使用到 vlan 的功能，那麼我覺得 OVS 的設定比較容易理解，架構類似於一台交換機，不需要去處理 Linux Bridge 的 vlan-aware 之類的選項。

有關 OVS 的概念，我是從這篇文章中參考的，有興趣可以查看( [Proxmox VE + Ceph 超融合项目实战（第五部分：虚拟网络） - Varden - 博客园](https://www.cnblogs.com/varden/p/15206928.html)[^1] )

OVS 創建一個 vmbr 可以理解為建立一個交換機，將一個網口加入 OVS Bridge 中相當於給這個交換機加上一個網路接口，而原本的接口變為 OVS Port。

{% gi 2 2 %}

![](ovs_interface.png)

![](ovs_bridge.png)

{% endgi %}

如果 OVS Port 沒有設定 vlan，則這個接口相當於以下的設定，在 OVS Bridge 沒有 vlan，作為傻瓜交換機時，就是一個 Port，當 OVS Bridge 內有 vlan 時， OVS Port 則是相當於 trunk port，可以直接和其他交換機的 trunk 互連互通，也可設定 trunk vlan filter，只允許特定 vlan。

```CISCO
!
interface GigabitEthernet1/0/1
 switchport trunk native vlan 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
```

{% gi 2 2 %}

![](ovs_port_1.png)

![](ovs_port_2.png)

{% endgi %}

如果 OVS Port 設定了 vlan tag，則接口相當於以下的設定，作為交換機的 access port，也可直接接入其他無 vlan 的交換機。

```CISCO
!
interface GigabitEthernet1/0/1
 switchport access vlan 100
 switchport mode access
!
```

![](ovs_port_3.png)

除了實體的 OVS Port，還可以建立虛擬的 OVS IntPort，相當於在主機上建立了一個虛擬的網卡，並且連接到 OVS Bridge 上，以下就是相當於連接到 OVS Bridge vlan 100 的 access port。

![](ovs_vlan.png)

以上就是 OVS Bridge 的基本設定

##  OVS Bond

OVS 同樣也可以使用 Bond，但是不能與 Linux Bond 混用，需要使用 OVS Bond 建立，結構類似於這樣， OVS Bond 類似於 OVS Port

![](ovs_bond_structure.png)

機器有的網路介面如下，目標是將四個網路介面聚合

![](ovs_bond_interface.png)

首先需要先建立一個空的 OVS Bridge

![](ovs_bond_bridge.png)

接著建立 OVS Bond，選擇剛建立的 OVS Bridge，可以在 OVS options 的地方加上`other_config:lacp-time=fast`

![](ovs_bond.png)

剩下的建立 OVS IntPort 與前面方法相同

---

另一側連接的交換機可以這樣設定來達成鏈路聚合，假設 eno1~4 分別連接到 GigabitEthernet1/0/1 - 4

```CISCO
!
interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface GigabitEthernet1/0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 lacp rate fast
!
interface GigabitEthernet1/0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 lacp rate fast
!
interface GigabitEthernet1/0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 lacp rate fast
!
interface GigabitEthernet1/0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 1 mode active
 lacp rate fast
!
```

這樣就可以成功的與 OVS Bond 達成 LACP 的鏈路聚合

# SDN

SDN (Software-Defined Networkin)，中文為軟體定義網路，我使用他最大的原因是方便在 PVE 中對網路進行權限管理，可以細分每個 vlan 誰能使用，也能將 vlan 變為 VNet，用戶就不用記憶 vlan 的數字是哪個，只要從 VNet 的名字判斷就可以了
可以在 Datacenter 的選單下找到 SDN，首先需要建立 Zone
Proxmox 的 SDN Zone type 有這幾種，一般的使用者應該都是使用 VLAN 即可，設定如下， Bridge 為每個節點上使用 vmbr 的名字

{% gi 2 2 %}

![](sdn_type.png)

![](sdn_vlan_zone.png)

{% endgi %}

接著新增 VNet，這些 VNet 就是可以被虛擬機選擇的網路，設定好 vlan tag 後，機器直接選擇 VNet 就相當於直接接入 Bridge 的 access port，不用再次填寫 vlan tag，注意名字最長只能有8個字元

{% gi 2 2 %}

![](sdn_vnets.png)

![](sdn_vnet.png)

{% endgi %}

接著就可以直接在 vm 中使用這個 VNet 了，還可以針對每個 VNet 設定權限，像是目前就只看得到 General，因為其他 VNet 的權限對此用戶關閉了

![](sdn_vm.png)

## 參考

[^1]: [Proxmox VE + Ceph 超融合项目实战（第五部分：虚拟网络） - Varden - 博客园](https://www.cnblogs.com/varden/p/15206928.html)
[^2]: [Open vSwitch - Proxmox VE](https://pve.proxmox.com/wiki/Open_vSwitch)
[^3]: [Configuring Proxmox VE 8 with an Open vSwitch LACP Bond and VLAN-aware Bridge for easy VLAN assignment](https://syslynx.net/proxmox-open-vswitch-lacp-vlans/)
[^4]: [EtherChannel PAgP LACP 以太通道 - Jan Ho 的網絡世界](https://www.jannet.hk/etherchannel-pagp-lacp-zh-hant/)
[^5]: [Understanding the OpenvSwitch Bonding | hwchiu learning note](https://www.hwchiu.com/docs/2015/openvswitch-bonding)