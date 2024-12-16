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
excerpt: 分享設定OVS和SDN的過程中的理解
date: 2024-12-16 21:36:51
updated: 2024-12-16 21:36:51
index_img:
---


# 前言

這篇算是整理我設定CCNS的PVE集群的一些經驗，還有對於OVS與SDN的理解

# OVS

Open vSwitch我記得是在PVE v6還是v7的時候加入的，如果對於vmbr的需求只有傻瓜交換機的功能，那麼使用OVS和Linux Bridge體驗是相同的，不過如果有使用到vlan的功能，那麼我覺得OVS的設定比較容易理解，架構類似於一台交換機，不需要去處理Linux Bridge的vlan-aware之類的選項。

有關OVS的概念，我是從這篇文章中參考的，有興趣可以查看( [Proxmox VE + Ceph 超融合项目实战（第五部分：虚拟网络） - Varden - 博客园](https://www.cnblogs.com/varden/p/15206928.html)[^1] )

OVS創建一個vmbr可以理解為建立一個交換機，將一個網口加入OVS Bridge中相當於給這個交換機加上一個網路接口，而原本的接口變為OVS Port。

{% gi 2 2 %}

![](ovs_interface.png)

![](ovs_bridge.png)

{% endgi %}

如果OVS Port沒有設定vlan，則這個接口相當於以下的設定，在OVS Bridge沒有vlan，作為傻瓜交換機時，就是一個Port，當OVS Bridge內有vlan時，OVS Port則是相當於trunk port，可以直接和其他交換機的trunk互連互通，也可設定trunk vlan filter，只允許特定vlan。

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

如果OVS Port設定了vlan tag，則接口相當於以下的設定，作為交換機的access port，也可直接接入其他無vlan的交換機。

```CISCO
!
interface GigabitEthernet1/0/1
 switchport access vlan 100
 switchport mode access
!
```

![](ovs_port_3.png)

除了實體的OVS Port，還可以建立虛擬的OVS IntPort，相當於在主機上建立了一個虛擬的網卡，並且連接到OVS Bridge上，以下就是相當於連接到OVS Bridge vlan 100的access port。

![](ovs_vlan.png)

以上就是OVS Bridge的基本設定

##  OVS Bond

OVS 同樣也可以使用Bond，但是不能與Linux Bond混用，需要使用OVS Bond建立，結構類似於這樣，OVS Bond類似於OVS Port

![](ovs_bond_structure.png)

機器有的網路介面如下，目標是將四個網路介面聚合

![](ovs_bond_interface.png)

首先需要先建立一個空的OVS Bridge

![](ovs_bond_bridge.png)

接著建立OVS Bond，選擇剛建立的OVS Bridge，可以在OVS options的地方加上`other_config:lacp-time=fast`

![](ovs_bond.png)

剩下的建立OVS IntPort與前面方法相同

---

另一側連接的交換機可以這樣設定來達成鏈路聚合，假設eno1~4分別連接到GigabitEthernet1/0/1 - 4

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

這樣就可以成功的與OVS Bond達成LACP的鏈路聚合

# SDN

SDN (Software-Defined Networkin)，中文為軟體定義網路，我使用他最大的原因是方便在PVE中對網路進行權限管理，可以細分每個vlan誰能使用，也能將vlan變為VNet，用戶就不用記憶vlan的數字是哪個，只要從VNet的名字判斷就可以了
可以在Datacenter的選單下找到SDN，首先需要建立Zone
Proxmox的SDN Zone type有這幾種，一般的使用者應該都是使用VLAN即可，設定如下，Bridge為每個節點上使用vmbr的名字

{% gi 2 2 %}

![](sdn_type.png)

![](sdn_vlan_zone.png)

{% endgi %}

接著新增VNet，這些VNet就是可以被虛擬機選擇的網路，設定好vlan tag後，機器直接選擇VNet就相當於直接接入Bridge的access port，不用再次填寫vlan tag，注意名字最長只能有8個字元

{% gi 2 2 %}

![](sdn_vnets.png)

![](sdn_vnet.png)

{% endgi %}

接著就可以直接在vm中使用這個VNet了，還可以針對每個VNet設定權限，像是目前就只看得到General，因為其他VNet的權限對此用戶關閉了

![](sdn_vm.png)

## 參考

[^1]: [Proxmox VE + Ceph 超融合项目实战（第五部分：虚拟网络） - Varden - 博客园](https://www.cnblogs.com/varden/p/15206928.html)
[^2]: [Open vSwitch - Proxmox VE](https://pve.proxmox.com/wiki/Open_vSwitch)
[^3]: [Configuring Proxmox VE 8 with an Open vSwitch LACP Bond and VLAN-aware Bridge for easy VLAN assignment](https://syslynx.net/proxmox-open-vswitch-lacp-vlans/)
[^4]: [EtherChannel PAgP LACP 以太通道 - Jan Ho 的網絡世界](https://www.jannet.hk/etherchannel-pagp-lacp-zh-hant/)
[^5]: [Understanding the OpenvSwitch Bonding | hwchiu learning note](https://www.hwchiu.com/docs/2015/openvswitch-bonding)