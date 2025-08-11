---
title: Proxmox 救援記錄
katex: false
mathjax: false
mermaid: false
excerpt: 更新到 PVE9 的救援記錄
index_img:
categories:
- 教程
- 疑難雜症
tags:
- server
- Proxmox
- network
- OVS
- OpenvSwitch
---

# 前言

前幾天 PVE 9 剛發布時我立刻追上了最新版，但是卻在更新時出了一些問題，導致整個 network stack crash 掉，本文將介紹更新到 PVE 9 時可能會發生的問題、解決與最後的救援方法。

# 問題

在更新到 PVE 9 時(`apt dist-upgrade`)，在更新完 openvswitch 後，`openvswitch-switch.service`會被關閉但沒被重啟，如果 vmbr 使用 OVS 而非 Linux Bridge，就會發生網路斷線的狀況。

Unpack 完 openvswitch 相關的包後過一小段時間就發生斷線

![](update-log.png)

這是後來在另一台機器上更新時，使用 tmux 留下來的記錄，在更新第二台的時候已經知道原因，也提前做好備援，因此沒有要到救援的程度，只需重啟`openvswitch-switch.service`即可恢復。

```shell
# systemctl status openvswitch-switch.service
● openvswitch-switch.service - Open vSwitch
     Loaded: loaded (/usr/lib/systemd/system/openvswitch-switch.service; enabled; preset: enabled)
     Active: active (exited) since XXXX CST; X days ago
 Invocation: 
   Main PID:  (code=exited, status=0/SUCCESS)
   Mem peak: 
        CPU: 

XXXX systemd[1]: Starting openvswitch-switch.service - Open vSwitch...
XXXX systemd[1]: Finished openvswitch-switch.service - Open vSwitch.
XXXX systemd[1]: Stopping openvswitch-switch.service - Open vSwitch...
XXXX systemd[1]: openvswitch-switch.service: Deactivated successfully
XXXX systemd[1]: Stopped openvswitch-switch.service - Open vSwitch...
```

## 更新中斷

在第一台 PVE 進行更新時，發生了上述的問題，而那台剛好是我用來做 PCI Passthrough 的機器，當時為了避免顯卡載入原本 Driver 的 module 而不是載入 vfio module，把所有顯卡相關的 module 都 blacklist 掉了。

這使得這台 PVE 只能通過網路 SSH 操作，哪怕我有留一張亮機卡也沒用，因為 nvidia/nouveau module 被 blacklist，系統不會輸出任何畫面，也沒有 Serial Console 可用，完全喪失對機器的控制。

我等待了一個小時，祈禱他能自己完成更新後強制重啟了它，果然，更新沒有成功，開機後網路沒有恢復，還是無法對它操作，因此就需要下一章的救援了。

# 救援

由於本身的系統已經出問題了，要救援只能靠外部的 live 環境，這裡使用 proxmox 的 ISO，當時是想說內建 Rescue Boot，~~雖然後來發現沒辦法使用~~

將電腦開機到 Proxmox ISO

{% gi 2 2 %}

![](proxmox-iso-1.png)

![](proxmox-iso-2.png)

{% endgi %}

我系統是使用 ZFS Mirror，如果用 Rescue Boot 開機的話會直接報錯

```log
error: no such device: rpool.
ERROR: unable to find boot disk automatically.

Press any key to continue...
```

後來改用 Debug Mode 進行掛載根目錄來救援

選擇 TUI 的 Debug Mode

如果使用的是較舊的 Nvidia 顯卡(我使用 GT720 當亮機卡)，可能會在載入模組的地方就卡死，可以在選擇介面按 e，在 Kernel Parameter 上加上`nomodeset`，按 Ctrl-x 開機，就可以正常開機進入。

![](proxmox-iso-3.png)

在開機後要先`exit`才能進入 live 環境的 terminal，

![](proxmox-iso-4.png)

接著在 terminal 中進行掛載操作

```shell
root@proxmox:/# /sbin/modprobe zfs
root@proxmox:/# mkdir /mypool
root@proxmox:/# zpool import -f -R  /mypool rpool
```

如果有裝蜂鳴器覺得提示音很吵，也可以移除 PC speaker module

```shell
root@proxmox:/# rmmod pcspkr
```

在我這次救援中不需要到需要 chroot 的程度，只要改 module 的設定檔關閉 blacklist，這樣就可以正常接螢幕來維護系統即可。

將設定檔的 blacklist 全部註解掉換回 softdep(參考[在 Proxmox 上進行 PCI-E 直通](/pci-passthrough-with-proxmox/#基礎設定)基礎設定第3步)

離開前要將 ZFS 的 Pool export，這樣原系統才可以正確掛載

```shell
root@proxmox:/# cd /
root@proxmox:/# zpool export rpool
root@proxmox:/# reboot
```

回到原系統後螢幕正常顯示，雖然網路還是沒有恢復，但已經可以操作維護了

發現是更新沒有全部完成，使用 dpkg 繼續完成

```shell
# dpkg --configure -a
```

在更新完成後重新啟動，系統就完全恢復了

## 參考

[^1]: [SOLVED: Proxmox 8.0 Rescue Boot help needed | Proxmox Support Forum](https://forum.proxmox.com/threads/solved-proxmox-8-0-rescue-boot-help-needed.132978/)
[^2]: [Proxmox rescue disk trouble | Proxmox Support Forum](https://forum.proxmox.com/threads/proxmox-rescue-disk-trouble.127585/#post-557888)