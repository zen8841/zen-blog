---
title: 如何透過網路列印和掃描
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - linux
  - server
excerpt: 介紹如何建立print server用來遠端列印和掃描
date: 2024-11-19 04:08:21
updated: 2024-11-19 04:08:21
index_img:
---


# 前言

簡單來說就是因為家中的印表機和掃描器放置的位置距離電腦有段距離，又不希望每次需要使用時都需要搬動電腦來連接，因此起了架設伺服器來分享的想法。

# 設置

## Print Server

{% note info %}

建議不要使用性能太差的單片機來架設Print Server，之前我使用過orange pi zero，因為性能太差，導致解碼Ghostscript需要花很多時間，印一頁需要超過一分鐘來渲染。

{% endnote %}

1. 安裝cups

```shell
# apt install cups
# # 底下這些是選擇性安裝，printer-driver-splix是三星舊印表機的驅動，需要安裝對應印表機的驅動，如果預設裡沒有
# apt install cups-bsd cups-tea4cups cups-x2go cups-backend-bjnp printer-driver-splix
```

2. 編輯cups設定檔(`/etc/cups/cupsd.conf`)

```conf
# 加上Port 631，並把Listen localhost:631註解，允許被所有IP訪問
Port 631
# 開啟webinterface
WebInterface Yes

# 在admin的路徑底下都加上Allow @LOCAL，這樣可以使用本機的帳號登入
<Location /admin>
  Order allow,deny
  Allow @LOCAL
</Location>
<Location /admin/conf>
  AuthType Default
  Require user @SYSTEM
  Order allow,deny
  Allow @LOCAL
</Location>
<Location /admin/log>
  AuthType Default
  Require user @SYSTEM
  Order allow,deny
  Allow @LOCAL
</Location>
```

3. 將自己的帳號加入lpadmin群組，才能使用帳號登入

```shell
# usermod -aG lpadmin account_name
```

4. 重啟cups服務即完成架設，`systemctl restart cups.service`
5. 插上印表機，並在網頁中設定印表機

![](cups_config.png)

選擇Add Printer，接著按照指示設定即可，需在最後一步將Share Printer勾起，之後就可以直接使用對應的位置直接進行影印了，不過windows應該可以在同一個Lan底下直接偵測到。

如：http://x.x.x.x:631/printers/Samsung_ML-1710 (使用http因為https的證書沒有簽名，windows預設會拒絕，linux就沒差了)

## Remote scan

在遠端掃描這方面使用的是SANE(Scanner Access Now Easy)，它支援透過網路進行遠端掃描

### 服務端

1. 安裝必要的軟體

```shell
# # libpam-systemd會提供有關掃描器的udev rules，不過應該都已經有裝了
# apt install sane-utils libpam-systemd
# # 選擇性安裝
# # sane-airscan可以連線遠端的掃描器，如果是直接連線可以不管
# # libsane-hpaio可以驅動一些HP的一體機的掃描器(印表機+掃描器)
# # hplip同理，是HP的驅動，如果libsane-hpaio無效可以嘗試，或直接都裝
# apt install libsane-hpaio hplip sane-airscan
```

2. 設定允許使用掃描的IP，編輯設定檔(`/etc/sane.d/saned.conf`)

```conf
# 範例，直接將IP或CIDR直接加進去即可
10.0.0.0/8
```

```shell
# # 重啟服務
# systemctl restart saned.socket
# # 如果服務沒有enable，記得打開
# systemctl enable saned.socket
```

3. 插上掃描器，確認掃描器的權限正確( 如果之前已經插上掃描器，這一步沒看到+號，可以嘗試重新插拔掃描器，讓udev rule偵測到硬體變化 )

```shell
# ls -l /dev/bus/usb/*/*
crw-rw-r--  1 root root 189,   0 Nov 16 21:37 /dev/bus/usb/001/001
crw-rw-r--  1 root root 189,   1 Nov 16 21:37 /dev/bus/usb/001/002
crw-rw-r--  1 root root 189, 128 Nov 16 21:51 /dev/bus/usb/002/001
crw-rw-r--  1 root lp   189, 147 Nov 17 02:10 /dev/bus/usb/002/020
crw-rw-r--+ 1 root lp   189, 148 Nov 17 03:11 /dev/bus/usb/002/021
crw-rw-r--  1 root root 189, 256 Nov 16 21:51 /dev/bus/usb/003/001
```

應該可以看到掃描器對應的usb設備的權限後面有一個+，udev rule會在設備插入的時候加上權限

4. 檢查掃描器是否被sane偵測到

```shell
# # 應該要可以看到有一個設備是插上的掃描器
# sudo -u saned sane-find-scanner
# # 應該要可以看到掃描器被偵測到
# sudo -u saned scanimage -L
device `hpaio:/usb/Deskjet_F4100_series?serial=XXXXXXXX' is a Hewlett-Packard Deskjet_F4100_series all-in-one
```

5. 如果上面的正常被檢查到，那SANE的服務端大概就配置成功了

### 客戶端( Linux )

1. 按照對應的distro安裝sane，(例：`sudo pacman -S sane`)
2. 編輯`/etc/sane.d/dll.conf`，將net的註解取消
3. 編輯`/etc/sane.d/net.conf`，將SANE伺服器的IP加入，`echo x.x.x.x >> /etc/sane.d/net.conf`
4. 使用`scanimage -L`，確認是否成功連線遠端掃描器

### 客戶端( Windows )

Windows上有兩種方案可以使用

- 使用[wiasane](https://github.com/mback2k/wiasane)，他是一個驅動，可以把SANE的遠端掃描器變成Windows下的WIA設備，但是需要自己編譯，目前我沒找到編譯好的執行檔
- 使用[SaneTwain](https://sanetwain.ozuzo.net/)，這個掃描軟體可以直接連線遠端的SANE伺服器，直接填入IP即可開始掃描

## 參考
[^1]: [SaneOverNetwork - Debian Wiki](https://wiki.debian.org/SaneOverNetwork)
[^2]: [终于可以愉快地扫描了：Linux 扫描仪配置与使用攻略 - 少数派](https://sspai.com/post/91396)