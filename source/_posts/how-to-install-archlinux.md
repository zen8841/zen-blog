---
title: ArchLinux安裝指北
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - linux
  - Arch
excerpt: ArchLinux + KDE Plasma + UEFI systemd boot的安裝流程
date: 2025-05-01 18:09:23
updated: 2025-05-01 18:09:23
index_img:
---


# 前言

這篇文章也是應該要很久以前就寫完了，但是因為我個人很懶，所以才拖到現在，剛好最近有CCNS的社課要教學ArchLinux安裝，就順便把文章趕出來。

安裝Arch最好要確保你看得懂ArchWiki上寫什麼，在安裝過程中，我會盡量給出每個地方對應的Wiki Page，如果有能力，最好將Wiki看完，才能對自己的系統有更深的理解，這篇文章主要是跟隨[Installation guide](https://wiki.archlinux.org/title/Installation_guide)[^1]的流程

## 為什麼要使用Arch

我個人認為ArchLinux有三個最主要的優點

- 高度的客製化：從安裝開始就完全由使用者決定裝什麼
- Rolling Release：ArchLinux是滾動式更新，軟體會一直在最新的穩定版本上
- AUR：Arch User Repository，提供其他Arch使用者撰寫的PKGBUILD，可以用來產生不在官方鏡像站裡的軟體包，當然，使用前需要自己看看有沒有人在裡面加料，AUR不對此保證安全

## Arch很容易滾掛？

我使用ArchLinux差不多6年了，目前是沒有遇過更新到開不了機的大問題，不過日用可能隔幾個月會需要解決一些也許是因為軟體更新造成的問題，但是使用其他發行版同樣可能遇到，所以Arch其實並沒有別人想像的不穩定。

# 安裝

我個人安裝Arch不喜歡使用`archinstall`指令，一方面是以前安裝的時候就沒這個指令，另一方面，前面提到我認為Arch最大的優點就是高度客製化，如果使用腳本來安裝，那為什麼不使用其他發行版或是Artix呢？

---

ArchLinux的安裝其實很簡單，就像把大象放進冰箱只需要三個步驟一樣，我將ArchLinux的安裝也分成大致三個步驟(ArchWiki的Installation guide也是分成這三個大步驟[^1])

- 前置準備
  - 連接網路
  - 硬碟分區、格式化和掛載
- 安裝軟體
  - 安裝系統需要的各種軟體
- 系統設定
  - 設定各種系統文件
  - 安裝其他軟體
  - 設定bootloader

## 前置準備

假設現在已經開機到Arch live USB的環境

### 連接網路

#### 有線網路

如果是使用ethernet，並且LAN中有DHCP server，則live USB會在開機後自動獲取IP，或是可以使用`dhcpcd`指令來手動獲取DHCP

使用`ip address`來確認自己是否有獲取IP

#### Wi-Fi

我個人在live USB中喜歡使用`iwctl`來連接無線網路[^2]

```SHELL
# iwctl
[iwd] device list
[iwd] station <device> scan
[iwd] station <device> get-networks
[iwd] station <device> connect <SSID>
[iwd] exit

# # 或是可以使用底下的指令一行來連接，如果知道要連接網路的SSID和密碼
# iwctl --passphrase <passphrase> station <device> connect <SSID>
```

同樣使用`ip address`來確認自己是否有獲取Wi-Fi連接後DHCP配發的IP

#### 測試

隨便ping個東西測試

```shell
# ping archlinux.org
```

#### 設定SSH

這一項並不是必須，但如果網路內有其他電腦，我會習慣在其他電腦上ssh到live USB裡遠端安裝，這樣終端機比較好看，也可以複製貼上

```shell
# # 設定密碼，隨便設就好
# passwd
# # 找到PermitRootLogin，改成PermitRootLogin yes
# vim /etc/ssh/sshd_configs
# # 重啟sshd
# systemctl restart sshd
```

### 設定時間

避免因為時間錯誤對照下載時錯誤

```shell
# timedatectl set-ntp true
```

### 硬碟分區

我個人習慣使用gdisk來進行分區，看個人習慣也可以使用fdisk、parted或是有TUI的cfdisk、cgdisk

#### 分區範例

UEFI需要使用GPT分區表，並且指少需要將`/boot`獨立分區，底下是範例的分區，如果需要安裝很多不同核心或是Early KMS的kernel module，則Boot可以適當放大到1G

|   Name    |  Size  |   Code    |
| :-------: | :----: | :-------: |
|   Boot    | 512MiB |   EF00    |
|   Root    | 60GiB  | 8300/8304 |
| Timeshift | 70GiB  |   8300    |
|   Home    |  $-$   | 8300/8302 |

#### gdisk 指令

在gdisk中會需要用到的指令大概就這些，可以按?來查看所有選項

| Command |              Usage               |
| :-----: | :------------------------------: |
|    p    |    print the partition table     |
|    n    |       add a new partition        |
|    d    |        delete a partition        |
|    c    |    change a partition's name     |
|    t    |  change a partition's type code  |
|    q    |       quit without saving        |
|    w    |          save and exit           |
|   x-z   | zap GPT data structures and exit |

1. 基本上就是先x然後z摧毀所有原本的GPT分區表(如果硬碟之前有裝東西，而且不再需要裡面的資料)

2. 然後n創建新分區，我習慣`boot`在第一個，所以創建一個1G的分區，可以使用+1GiB來指定分區結束的位置，接著Partition type必須填EF00
3. 後面同樣創建root分區，home分區可以是選擇性創建，swap可以使用swapfile或是單獨創建swap分區

4. 接著w儲存分區表並離開

### 格式化

將<>中的內容替換為對應的路徑

```shell
# # EFI分區需使用fat32
# mkfs.fat -F 32 -n Boot /dev/<efi>
# # 其他分區這裡使用ext4示範，也可以換成其他你喜歡的檔案系統，可以不加-L參數，這個只是標示
# mkfs.ext4 -L Root /dev/<root>
# mkfs.ext4 -L Home /dev/<home>
# mkswap /dev/<swap>
```

### 掛載

```shell
# mount /dev/<root> /mnt
# mkdir /mnt/{boot,home}
# mount /dev/<efi> /mnt/boot
# mount /dev/<home> /mnt/home
# swapon /dev/<swap>
```

## 安裝軟體

### 安裝基本系統

前置準備到這裡就結束了，接著是安裝基本系統到`/mnt`(剛剛掛載的根目錄)，在這裡安裝了基本的軟體和編輯器，<intel/amd>看使用的cpu來選擇

```shell
# pacstrap -i  /mnt linux linux-headers linux-firmware base base-devel vim nano <intel/amd>-ucode
```

## 系統設定

### 生成fstab

```shell
# genfstab -U -p /mnt >> /mnt/etc/fstab
```

### chroot進新的系統

```shell
# arch-chroot /mnt
# pacman -Syy
```

### 設定時區

```shell
# ln -sf /usr/share/zoneinfo/Asia/Taipei /etc/localtime
# hwclock --systohc
```

### 設定地區與語言

不需設定鍵盤，預設為美式英語，和台灣常用鍵盤相同

```shell
# # 找到en_US.UTF-8 UTF-8和en_US.UTF-8 UTF-8取消註解
# vim /etc/locale.gen
# locale-gen
# echo LANG=en_US.UTF-8 > /etc/locale.conf
# export LANG=en_US.UTF-8
```

### 設定hostname

```shell
# # 直接填喜歡的機器名稱就好
# vim /etc/hostname
# vim /etc/hosts
```

`/etc/hosts`按照下面的格式來新增，\<hostname>自行替換

```hostname
127.0.0.1   localhost
::1         localhost
127.0.1.1   <hostname>.localdomain   <hostname>
```

### 新增用戶與密碼

```shell
# # 設定root密碼
# passwd
# # 新增用戶，-m為新增家目錄，-G為設定附加群組，預設會建立與用戶名相同的組作為主群組
# useradd -m -G wheel,storage,power -s /bin/bash <username>
# # 設定用戶密碼
# passwd <username>
# 設定sudo，找到 %wheel ALL=(ALL:ALL) ALL 並取消註解
# visudo
```

### 設定固態硬碟修剪(垃圾回收)

如果沒有SSD可以跳過這裡

```shell
# systemctl enable fstrim.timer
```

### 設定系統引導

```shell
# bootctl install
# # 編輯引導設定，同樣<intel/amd>視使用的CPU來修改
# vim /boot/loader/entries/arch.conf
```

```arch.conf
title Arch
linux /vmlinuz-linux
initrd /<intel/amd>-ucode.img
initrd /initramfs-linux.img
```

```shell
# echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/<root_partition>) rw" >> /boot/loader/entries/arch.conf
# mkinitcpio -P
```

到這一步ArchLinux已經安裝完成了，但為了方便，我習慣直接把需要的軟體一起安裝完，而不是等重啟後進系統再裝

### 設定pacman

#### Mirrorlist

```shell
# vim /etc/pacman.d/mirrorlist
```

看自己習慣用哪些鏡像站，目前台灣最快的是TWDS，但我有在維護CCNS的鏡像，所以優先使用

```mirrorlist
Server = https://archlinux.ccns.ncku.edu.tw/archlinux/$repo/os/$arch
Server = https://mirror.archlinux.tw/ArchLinux/$repo/os/$arch
```

或是可以用速度或一些條件來決定

##### Rate Speed

```shell
# pacman -S pacman-contrib
# rankmirrors -t /etc/pacman.d/mirrorlist
```

##### Reflector

```shell
# pacman -S reflector
# # 從HK、JP、TW、US的鏡像站中挑選速度前10快的，他也有其他條件可以篩選
# reflector --verbose -c HK -c JP -c TW -c US -f 10 --sort rate --save /etc/pacman.d/mirrorlist
```

#### 開啟32bit的支持

```shell
# # 找到下面的並取消註解
# vim /etc/pacman.conf
```

```pacman.conf
[multilib]
Include = /etc/pacman.d/mirrorlist
```

#### 開啟Archlinux CN

開啟Archlinux CN Repository，有一些常用的AUR已經被編譯好了，看你信不信任他

```shell
# vim /etc/pacman.conf
# vim /etc/pacman.d/mirrorlist.archlinuxcn
```

```pacman.conf
[archlinuxcn]
Include = /etc/pacman.d/mirrorlist.archlinuxcn
```

```mirrorlist.archlinuxcn
Server = https://archlinux.ccns.ncku.edu.tw/archlinuxcn/$arch
Server = https://mirror.twds.com.tw/archlinuxcn/$arch
```

```shell
# pacman -S archlinuxcn-keyring
```

### 安裝顯卡驅動

這部份最好看每個對應的wiki page，這邊只是我很久以前整理出來節省時間的

#### Intel[^3]

```shell
# pacman -S mesa lib32-mesa xf86-video-intel vulkan-intel intel-media-driver intel-gpu-tools
# xf86-video-intel 可不安裝，可能導致部分問題
```

#### Nvidia[^4]

```shell
# # 或是nvidia可以換成nvidia-dkms，nvidia是針對linux這個kernel編譯的，如果想使用其他核心需改用nvidia-dkms
# pacman -S mesa nvidia nvidia-utils lib32-nvidia-utils libglvnd lib32-libglvnd opencl-nvidia lib32-opencl-nvidia nvidia-settings
# # 開啟early module load
# vim /etc/mkinitcpio.conf
# mkinitcpio -P
# vim /boot/loader/entries/arch.conf
```

```mkinitcpio.conf
MODULES=(nvidia nvidia-modeset nvidia-uvm nvidia-drm)
```

```arch.conf
options root=... nvidia_drm.modeset=1
```

##### hooks

使用nvidia-dkms可跳過設定hook

```shell
# mkdir /etc/pacman.d/hooks
# vim /etc/pacman.d/hooks/nvidia.hooks
```

```nvidia.hooks
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia
Target=linux

[Action]
Description=Update Nvidia module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

#### AMD[^5]

```shell
# pacman -S mesa lib32-mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon
```

### 安裝AUR Helper

我習慣的AUR Helper是yay，他的用法和pacman一模一樣

```shell
# su <username>
#cd
# git clone https://aur.archlinux.org/yay.git
# cd yay
# makepkg -si
# cd ..
# rm -rf yay
```

### 安裝桌面環境

基本的應該就這些吧

```shell
# pacman -S plasma xorg-xinit alsa-utils bluez-utils 
# pacman -S --asdeps xorg-server xorg-twm xterm xorg-apps pipewire-alsa pipewire pipewire-pulse pipewire-jack
```

#### 安裝字體

看自己要裝哪些

```shell
# yay -S ttf-inconsolata nerd-fonts awesome-terminal-fonts ttf-nerd-fonts-symbols-2048-em ttf-nerd-fonts-symbols-2048-em-mono nerd-fonts-3270 noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra ttf-dejavu ttf-droid ttf-hack ttf-liberation ttf-opensans
```

#### 安裝輸入法

```shell
# pacman -S fcitx5-im fcitx5-rime
```

#### 其他

我個人還有額外裝很多東西，不過那些等到下一篇文章：ArchLinux的客製化再說吧，這篇只是針對安裝ArchLinux已經夠長了

### 啟動服務

```shell
# systemctl enable NetworkManager
# systemctl enable avahi-daemon
# systemctl enable bluetooth
# systemctl enable cups
# systemctl enable sddm
# systemctl enable paccache.timer
```

### 重新啟動

```shell
# # 離開chroot
# exit
# umount -R /mnt
# reboot
```

## 參考

[^1]: [Installation guide - ArchWiki](https://wiki.archlinux.org/title/Installation_guide)
[^2]: [iwd - ArchWiki](https://wiki.archlinux.org/title/Iwd#iwctl)
[^3]: [Intel graphics - ArchWiki](https://wiki.archlinux.org/title/Intel_graphics)
[^4]: [NVIDIA - ArchWiki](https://wiki.archlinux.org/title/NVIDIA)
[^5]: [AMDGPU - ArchWiki](https://wiki.archlinux.org/title/AMDGPU)
