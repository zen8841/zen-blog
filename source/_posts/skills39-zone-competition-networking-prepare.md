---
title: 技能競賽 - 資訊網路技術區賽Networking part筆記
katex: false
mathjax: false
mermaid: false
categories:
  - 比賽
tags:
  - network
excerpt: 分享一下區賽準備過程的筆記
date: 2025-04-09 18:31:30
updated: 2025-04-09 18:31:30
index_img:
---


{% note primary %}

這篇大概包括近8年左右的packet tracer解答需要的指令，不保證正確，我個人的筆記而已

會分為三個部分，General Setting同時適用於router、switch、L3 switch，也是基本設定的指令，Switch Setting為在switch上的設定，設定接入等等的內容，Router Setting則是在router上的設定，基本是關於L3路由的內容，L3 switch則自行參考Switch和Router兩個部分的內容

{% endnote %}

{% note info %}

標注*的代表每年都必出的部分

如果沒有特別註明`#`，則默認是在config terminal mode下的指令，config mode下都用`(config)#`開頭

記得沒事就用`wr`(`write`)或是`do wr`(`do write`)來儲存設定

如果忘記指令就一直按?來查指令在哪個底下

遇到要在每個設備上都要設定的指令時，可以直接開記事本一次打完，然後複製到所有設備的terminal上，這樣配置比較快

這篇文章裡如果有不懂的部分幾乎都可以透過關鍵詞+Jan Ho 的網絡世界來查到一個寫cisco介紹寫的很好的人的文章，可以看他更詳細的介紹

{% endnote %}

# General Setting

## *hostname

設定hostname為ALS1

```cisco
(config)# hostname ALS1
```

## domain name

設定domain name為skills39\.tw

```cisco
(config)# ip domain-name skills39.tw
```

## 關閉指令自動解析為域名

```cisco
(config)# no ip domain-lookup
```

## *login

vty的編號為同時能遠端開啟幾個連接，一般都開0-15全開，除非他要求同時連線的最大數量

### ssh

設定ssh需要先設定domain name，他應該會一起提供

建立admin的user，密碼為Sills39，並且要雜湊來儲存

```cisco
(config)# ip ssh version 2
(config)# crypto key generate rsa # rsa金鑰需要生成2048bits，他會問要生幾bits的金鑰，預設是512bits
(config)# username admin secret 0 Skiils39
(config)# line vty 0 15
(config-line)# transport input ssh # 只開啟ssh，如果要同時開啟telnet要用all
(config-line)# login local
```

### telnet

開啟telnet，並使用本機的user進行登入

```cisco
(config)# username admin secret 0 Skiils39
(config)# line vty 0 15
(config-line)# transport input telnet # 只開啟ssh，如果要同時開啟telnet要用all
(config-line)# login local
```

### 登入密碼驗證

登入使用密碼驗證，密碼為Skills39，並進行雜湊，但不在本機建立user

```cisco
(config)# service password-encryption
(config)# line vty 0 15 # 或是line console 0
(config-line)# password Skills39
(config-line)# login
```

### 設定登入逾時

設定不操作2分30秒後登出，並且登入10分鐘後強制登出

```cisco
(config)# line vty 0 15 # 或是line console 0
(config-line)# exec-timeout 2 30
(config-line)# absolute-timeout 10
```

### 登入後自動進入特權模式

```cisco
(config)# line vty 0 15 # 或是line console 0
(config-line)# privilege level 15
```

### 設定log不切斷輸入

```cisco
(config)# line vty 0 15 # 或是line console 0
(config-line)# logging synchronous
```

### user建立

建立admin的user並使用密碼Skills39，特權等級設定最高

```cisco
(config)# username admin privilege 15 secret 0 Skiils39
```

### enable密碼

設定enable密碼為Skills39，並開啟雜湊

```cisco
(config)# enable secret 0 Skills39
(config)# service password-encryption
```

## *IP Setting

{% note info %}

Switch只能在vlan的介面上設定IP，Router可以直接在interface上設定，L3 Switch可以透過使用`no switchport`來像Router一樣直接在介面上設IP

{% endnote %}

```cisco
(config)# interface vlan 10 # 如果是router可能就是interface gigabitethernet 1/0/1
(config-if)# ip address 10.0.0.253 255.255.255.0
(config-if)# no shutdown # 如果介面沒啟動記得要啟動
(config-if)# no switchport # 如果L3 Switch要在介面上設IP要先加這個
(config-if)# exit
(config)# ip default-gateway 10.0.0.254
```

### sub interface

{% note info %}

如果router連接Switch的介面是走trunk，就需要sub interface來把vlan從介面中拆出來

{% endnote %}

```cisco
(config)# interface gigabitethernet 1/0/1.10 # 我習慣sub interface的id設成vlan的id
(config-subif)# encapsulation dot1q 10 # 使用802.1q的vlan protocal，並且把vlan id為10的vlan分到這個sub interface
(config-subif)# ip address 10.0.0.253 255.255.255.0 # 後面就和一般介面設定一樣
(config-subif)# no shutdown
```

### IPv6 Setting

```cisco
(config)# interface vlan 10 # 如果是router可能就是interface gigabitethernet 1/0/1
(config-if)# ipv6 address X:X:X:X::X/64
(config-if)# no shutdown # 如果介面沒啟動記得要啟動
(config-if)# no switchport # 如果L3 Switch要在介面上設IP要先加這個
(config-if)# exit
(config)# ipv6 route ::/0 X:X:X:X::1 
```

## *DHCP

{% note info %}

發放10.0.0.10-10.0.0.100的IP，gateway為10.0.0.254，dns server為8.8.8.8，lease time一天

{% endnote %}

```cisco
(config)# ip dhcp pool VLAN10 # VLAN10是pool name
(dhcp-config)# network 10.0.0.0 255.255.255.0
(dhcp-config)# defaylt-router 10.0.0.254
(dhcp-config)# dns-server 8.8.8.8
(dhcp-config)# lease 1
(dhcp-config)# exit
(config)# ip dhcp excluded-address 10.0.0.1 10.0.0.9
(config)# ip dhcp excluded-address 10.0.0.101 10.0.0.254
```

{% note warning %}

DHCPv6和DHCP Snopping沒有考過，如果想準備全面一點可以去看怎麼設定

{% endnote %}

## *DHCP relay

DHCP server和Client不在同一個子網中就會需要DHCP relay

```cisco
(config)# interface gigabitethernet 1/0/1
(config-if)# ip helper-address x.x.x.x # DHCP server的IP
```

## *ACL

```cisco
(config)# access-list NUM permit|deny x.x.x.x # NUM=1~99、1300~1999為standard acl，NUM=100~199、2000~2699為extended acl，建議?來看怎麼設
(config)# ip access-list extended|standard NAME
(config-std-nacl)# permit x.x.x.x
(config-ext-nacl)# permit tcp|udp x.x.x.x eq PORTNUM # 建議?來看怎麼設，extended可以設定更詳細的策略，如icmp或tcp/udp的port number match
```

## Time Setting

### NTP

```cisco
(config)# ntp server x.x.x.x # 設定x.x.x.x為ntp server
```

### Manual Setting

```cisco
(config)# clock set hh:mm:ss day month year
```

## Logging

```cisco
(config)# logging host x.x.x.x # 傳送syslog到x.x.x.x
(config)# logging x.x.x.x # 傳送log到x.x.x.x
```

# Switch Setting

## *vlan setting

建立下表的vlan

| vlan10 | vlan20 | vlan99 |
| :----: | :----: | :----: |
|  Mgmt  | Guest  |  Park  |

```cisco
(config)# vlan 10
(config-vlan)# name Mgmt
(config-vlan)# vlan 20
(config-vlan)# name Guest
(config-vlan)# vlan 99
(config-vlan)# name Park
```

## *interface setting

### access port

- gigabitethernet 1/0/1 ~ 1/0/5 接入vlan10
- gigabitethernet 1/0/6 ~ 1/0/10 接入vlan20
- gigabitethernet 1/0/11 ~ 1/0/20 接入vlan99

```cisco
(config)# interface range gigabitethernet 1/0/1-5
(config-if)# switchport mode access
(config-if)# switchport access vlan 10
(config-if)# exit
(config)# int ra gi 1/0/6-10 # 也可以用簡寫
(config-if)# switchport mode access
(config-if)# switchport access vlan 20
(config-if)# exit
(config)# interface range gigabitethernet 1/0/11-20
(config-if)# switchport mode access
(config-if)# switchport access vlan 99
(config-if)# exit
```

### trunk port

gigabitethernet 1/0/24設定為trunk，只允許vlan10,20，native vlan設定為99

```cisco
(config)# interface gigabitethernet 1/0/24
(config-if)# switchport mode trunk
(config-if)# switchport trunk encapsulation dot1q # 在pt裡某些型號的switch可能不用設定，看他有沒有這個選項
(config-if)# switchport trunk allowed vlan 10,20
(config-if)# switchport trunk native vlan 99
```

### port security

設定連接終端的介面只能連接固定的一個設備，並且會自動記錄接入的mac，如果有違規的新設備要記錄並停止網路

```cisco
(config)# interface range gigabitethernet 1/0/1-10
(config-if)# switchport port-security maximum 1
(config-if)# switchport port-security mac-address sticky
(config-if)# switchport port-security violation restrict # 如果設定protect則是只drop不記錄，shutdown則是直接關閉介面，需要重新登入啟用
```

### bdpuguard

阻止接入終端的介面被私自接switch，並設定快速啟動接口，不等待STP收斂

```cisco
(config)# interface range gigabitethernet 1/0/1-10
(config-if)# spannging-tree portfast
(config-if)# spanning-tree bdpuguard enable
```

### 關閉介面

將Park vlan的介面關閉

```cisco
(config)# interface range gigabitethernet 1/0/11-20
(config-if)# shutdown
```

## *STP

題目有可能要求更換switch上使用的STP算法，如果要求是CISCO私有的協定代表要使用pvst，如果要求要使用IEEE制定的協議則是rapid-pvst

```cisco
(config)# spanning-tree mode rapid-pvst
```

### 設定root bridge

有時候題目會要求說某個vlan要優先走哪台switch，這就是需要設定那個swtich為那個vlan的root bridge

可以透過手動設定priority來指定，但通常不會這樣做

```cisco
(config)# spanning-tree vlan 10 priority 36864 # priority需要是4096的倍數
```

更常用的方法是下面這種，但是需要確保他和其他的switch有連接的介面或是在同一個bridge裡，他會得知其他switch在這個vlan的priority，並自動把自己設成低一個單位的priority(priority越低越優先)

```cisco
(config)# spanning-tree vlan 10 root primary # 備援的switch則設為secondary
```

### 縮短收斂時間

主要使用uplinkfast和backbonefast兩個指令

```cisco
(config)# spanning-tree uplinkfast # 不能在root bridge上設定
(config)# spanning-tree backbonefast # 在所有switch上都要設定才有用
```

## *Link Aggregation

題目不會直接說要用哪種方式做鏈路聚合，如果題目說不經過任何協商代表使用static on mode；如果題目說要用IEEE制定的標準或是802.3ax或ad，代表就是使用LACP；如果題目說使用CISCO私有的協定則是使用PAgP

我這邊只設定其中一邊，兩邊使用的port channel number不同沒有關係，並且port channel做為trunk介面

使用gigabitethernet 1/0/21、22和另一台switch連接，並作為trunk

```cisco
(config)# interface range gigabitethernet 1/0/21-22
(config-if)# channel-group 1 on # on 代表static on mode,active或passive代表LACP，desirable或auto代表PAgP，但我習慣兩邊都設主動模式(active或desirable)
(config-if)# switchport mode trunk
(config-if)# switchport trunk encapsulation dot1q # 在pt裡某些型號的switch可能不用設定，看他有沒有這個選項
(config-if)# switchport trunk allowed vlan 10,20
(config-if)# exit
(config)# interface port-channel 1
(config-if)# switchport mode trunk
(config-if)# switchport trunk encapsulation dot1q # 在pt裡某些型號的switch可能不用設定，看他有沒有這個選項
(config-if)# switchport trunk allowed vlan 10,20
```

## vtp

題目有可能會要求設定vtp，但是他可能不會直接講說哪台要做server或client(參考47屆的Networking題目)，他有可能要求說某台的vlan設定要存在哪裡

vtp server mode： vlan設定存在flash

vtp client mode： vlan設定存在ram

vtp transparent mode： vlan設定存在nvram，這個模式可以當成關閉vtp的感覺，如果題目這樣要求

### server

```cisco
(config)# vtp mode server
(config)# vtp domain skills39.tw # server和client的domain和password要一樣
(config)# vtp password Skills39
```

### client

```cisco
(config)# vtp mode client
(config)# vtp domain skills39.tw
(config)# vtp password Skills39
```

### transparent

```cisco
(config)# vtp mode transparent
```

### version

設定vtp版本為2，1有一些風險

```cisco
(config)# vtp version 2
```

## cdp

關閉思科設備發現協議，並阻止所有的介面發送cdp封包

```cisco
(config)# no cdp run
(config)# interface range gigabitethernet 1/0/1-24
(config-if)# no cdp enable
```

# Router Setting

## *FHRP 第一跳冗餘協定(router備援)

{% note info %}

應該只會考HSRP，PT裡好像不支援VRRP和GLBP的設定

設定都是在連接子網的介面上，子網10.0.0.0/24，gateway:10.0.0.254

{% endnote %}

### HSRP

```cisco
(config)# interface gigabitethernet 1/0/1
(config-if)# stanby 10 ip 10.0.0.254 # 互為備援的router使用的stanby id要相同(不同vlan要換不同id，我習慣用vlan id當stanby id)
(config-if)# stanby 10 priority 110 # 預設priority為100，最高的priority會成為active的router
(config-if)# stanby 10 preempt # 當自己的priority更高時，即使是新上線的也自動成為active
(config-if)# stanby 10 track gigabitethernet 1/0/2 20 # 當gigabitethernet 1/0/2 shutdown時自動減20的priority，這樣其他priority 100的router就會成為active
(config-if)# stanby 10 timers 1 3 # 設定hello time和hold time，hello time為每隔幾秒發送封包聯絡其他router，當超過hold time沒收到hello封包就默認其他router下線
(config-if)# stanby 10 authentication md5 key-string Skills39 # 設定使用md5作為驗證資訊，密碼為Skills39
```

### VRRP

{% note info %}

設定的概念和HSRP接近，就不再介紹

{% endnote %}

```cisco
(config)# interface gigabitethernet 1/0/1
(config-if)# vrrp 10 ip 10.0.0.254
(config-if)# vrrp 10 priority 110
(config-if)# vrrp 10 preempt
```

### GLBP

{% note info %}

設定的概念和HSRP接近，就不再介紹

{% endnote %}

```cisco
(config)# interface gigabitethernet 1/0/1
(config-if)# glbp 10 ip 10.0.0.254
(config-if)# glbp 10 priority 110
(config-if)# glbp 10 preempt
(config-if)# glbp 10 timers 1 3
(config-if)# glbp 10 authentication md5 key-string Skills39
```

## *NAT

{% note info %}

假設連接internet的IP為1.1.1.1-1.1.1.10/24

{% endnote %}

### 有多個外部IP可供轉換

```cisco
(config)# ip access-list standard INSIDE_NET permit 10.0.0.0 0.0.0.255
(config)# ip nat pool INTERNET 1.1.1.1 1.1.1.10 255.255.255.0
(config)# ip nat inside source list INSIDE_NET pool INTERNET overload # overload代表做PAT，如果不加代表做1by1 NAT
(config)# interface gigabitethernet 1/0/2
(config-if)# ip nat outside
(config-if)# exit
(config)# interface gigabitethernet 1/0/1
(config-if)# ip nat inside
```

### 只有一個IP

```cisco
(config)# ip access-list standard INSIDE_NET permit 10.0.0.0 0.0.0.255 # 這裡是wildcard不是netmask
(config)# ip nat inside source list INSIDE_NET interface gigabitethernet 1/0/2 overload # overload代表做PAT，如果不加代表做1by1 NAT
(config)# interface gigabitethernet 1/0/2
(config-if)# ip nat outside
(config-if)# exit
(config)# interface gigabitethernet 1/0/1
(config-if)# ip nat inside
```

## *Serial

### HDLC

{% note info %}

CISCO在Serial介面上預設使用的協定，如果題目沒說要用PPP，大概就是這個，直接給介面上IP就好，或是可能需要改clock rate

{% endnote %}

```cisco
(config)# interface serial 1/0/1
(config-if)# ip address 1.1.1.1 255.255.255.0
(config-if)# clock rate 2015232
```

### PPP

```cisco
(config)# interface serial 1/0/1
(config-if)# ip address 1.1.1.1 255.255.255.0
(config-if)# encapsulation ppp
(config-if)# ppp authentication chap # 還有另一種驗證方式PAP，但是PAP的用戶和密碼是明文，所以應該不會考，如果沒說要驗證，也不用設定這個
(config-if)# exit
(config)# username ISP password Skills39 # 假設連接的router hostname為ISP，設定密碼為Slills39，對面的router也要用你的hostname來新建用戶驗證
```

PPPoE應該不會考，有興趣可以看[Jan Ho的網站](https://www.jannet.hk/point-to-point-protocol-ppp-zh-hant/)

## *Routing

{% note info %}

基本是一定會考EIGRP或OSPF其中一個

{% endnote %}

### Static route

```cisco
(config)# ip route a.a.a.0 255.255.255.0 x.x.x.x # a.a.a.0/24的路由往x.x.x.x的router
```

### EIGRP

```cisco
(config)# router eigrp ASNUM # 題目會指定要用的AS number，同一個AS number才能成為neighbor
(config-router)# network 10.0.0.0 0.0.0.255 # 如果沒有auto-summary要自己指定wildcard，或是可以用下面這樣的設定
(config-router)# network 10.0.0.0
(config-router)# auto-summary
(config-router)# neighbor 10.0.0.252 gigabitethernet1/0/1 # 如果介面支援multicast，前面的network宣告後就會自動鄰接，不用手動宣告neighbor
(config-router)# redistibute static # 把static route發布出去，如果題目要求分發預設路由，就在static route加上預設路由和這個設定即可
(config-router)# redistibute connected # 把連接就介面的路由發布出去
```

#### Authentication

```cisco
(config)# key chain EIGRP
(config-keychain)# key 1
(config-keychain)# key-string Skills39
(config-keychain)# exit
(config)# interface gigabitethernet 1/0/1
(config-if)# ip authentication key-chain eigrp 1 EIGRP
(config-if)# ip authentication mode eigrp 1 md5
```

#### Timer

一樣有兩個timer，hello time和hold time，每隔hello time會發送封包，超過hold time沒收到就認為neighbor下線，預設值是5/15，縮短路由改變時間為1/5

```cisco
(config)# interface gigabitethernet 1/0/1
(config-if)# ip hello-interval eigrp ASNUM 1
(config-if)# ip hold-time eigrp ASNUM 5
```

#### passive-interface

關閉不需要介面的eigrp封包發送，只讓需要的介面發送封包

```cisco
(config)# router eigrp ASNUM
(config-router)# passive-interface default
(config-router)# no passive-interface gigabitethernet 1/0/1
```

### OSPF

```cisco
(config)# router ospf OSPFID # 題目會指定OSPF id，同一個OSPF id才可以成為neighbor
(config-router)# router-id x.x.x.x # 需要指定router id，題目應該也會給，如果沒有就用介面的IP，這只是辨識用的
(config-router)# network 10.0.0.0 0.0.0.255 area 0 # 在backbone area(id為0)宣告路由，題目也有可能要求在其他zone宣告路由，改最後的id就好
(config-router)# neighbor 10.0.0.252 # 如果介面支援multicast，前面的network宣告後就會自動鄰接，不用手動宣告neighbor
(config-router)# redistibute static # 發布靜態路由，但OSPF不會發布預設路由，預設路由要用下面的指令
(config-router)# default-information originate # 發布預設路由
```

#### Authentication

```cisco
(config)# router ospf OSPFID
(config-router)# area 0 authentication # area 0 需要驗證
(config-router)# exit
(config)# interface gigabitethernet 1/0/1
(config-if)# ip ospf authentication
(config-if)# ip ospf authentication-key xxxx
```

或是使用md5來驗證

```cisco
(config)# router ospf OSPFID
(config-router)# area 0 authentication message-digest # area 0 需要驗證
(config-router)# exit
(config)# interface gigabitethernet 1/0/1
(config-if)# ip ospf message-digest-key 0 md5 Skills39 # 不同area要設定不同的值
```

#### Timer

一樣有兩個timer，hello time和dead time，每隔hello time會發送封包，超過dead time沒收到就認為neighbor下線，預設值是10/40，縮短路由改變時間為1/4

```cisco
(config)# interface gigabitethernet 1/0/1
(config-if)# ip ospf hello-interval 1
(config-if)# ip ospf dead interval 4
```

#### passive-interface

和EIGRP的定義一樣

```cisco
(config)# router ospf OSPFID
(config-router)# passive-interface default
(config-router)# no passive-interface gigabitethernet 1/0/1
```

#### cost

改變計算cost的reference bandwidth為1G，每個OSPF router都要改

```cisco
(config)# router ospf OSPFID
(config-router)# auto-cost reference-bandwidth 1000
```

# 總結

分區賽Networking會考的差不多就這些，基本就是有*的題目和一些其他小項，分數佔比也不高，我個人是練了2屆的題目而已，其他屆的題目就看看除了必出部分以外出了什麼記一下，而且這屆題目變簡單，不確定之後會不會越來越簡單，主要的決戰還是在OS part

## 參考

[^1]: [Jan Ho 的網絡世界](https://www.jannet.hk/home-zh-hant/)
