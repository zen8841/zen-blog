---
title: 在Proxmox上進行PCI-E直通
katex: false
mathjax: false
mermaid: false
excerpt: 分享有關直通GTX650的時候遇到的問題，和一些看到其他疑難雜症的整理
date: 2024-11-04 23:39:33
updated: 2024-11-04 23:39:33
index_img:
categories:
- 教程
tags:
- linux
- Proxmox
- pci passthrough
- virtual machine
---

# 前言

簡單來說，PCI Passthrough就是將物理機(Host)上的PCI(E) device從Host上隔離並直通給客戶機(Client)的技術，對於Client來說，就會認為他真的插著一個PCI device。

直通也不一定是直通PCI device，也有可能是直通透過SR-IOV或是其他技術虛擬化出來的資源，不過這就需要硬體及Driver的支持，相對來說會更複雜。

因為每個Host的配置都不同，每個Host的PCI直通設定都不完全相同，有時候甚至PCIE device插在哪個PCIE插槽可能都會有影響，可能別人能用的解決方法你試就沒用，堪稱玄學。而直通顯卡到Windows Client更是PCI直通中最麻煩的設定，大概率會碰到更多的問題，不同Host的解決方法還不一定一樣，我對此的評價只有：玄學中的玄學。

# 設置

設定這部分我會大致分成四個區塊，第一部分是PCI Passthrough一定要做的設定，第二、三部分是我遇到的兩個問題及對我有用的解法，第四部分則是我整理的其他問題可能適用的方法

## 基礎設定

這部份的設定是PCI Passthrough都要進行的設定，如果運氣很好，硬體的組合剛好，那可能只需要這部份的設定就成功直通了。

---

1. 加入kernel parameter來開啟IOMMU

使用Intel CPU的話要加的參數是`intel_iommu=on`(雖然PVE的核心編譯的時候有打開`CONFIG_INTEL_IOMMU_DEFAULT_ON`，但仍然需要設定)

AMD CPU則不需要加參數，只要檢測到支持就會開啟IOMMU

不管使用什麼CPU都建議加上`iommu=pt`，如果硬體支援則可以提高性能

視自己使用的bootloader來進行更改

{% note info %}

如果是在PVE7後使用UEFI安裝，則bootloader會使用systemd-boot，反之，使用BIOS安裝則是使用Grub，但如果是原本使用Grub開機並升級上來的則會繼續使用Grub

{% endnote %}

- systemd-boot

編輯`/etc/kernel/cmdline`，前面會有一些其他參數，要加的參數直接放在後面，如下

```cmdline
quiet intel_iommu=on
```

編輯完後使用`pve-efiboot-tool refresh`來更新bootloader

- Grub

編輯`/etc/default/grub`

找到`GRUB_CMDLINE_LINUX`或是`GRUB_CMDLINE_LINUX_DEFAULT`，可以將參數加在雙引號之間，只要選擇其中一個加即可

```grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
GRUB_CMDLINE_LINUX="intel_iommu=on"
```

編輯完後使用`update-grub`來更新bootloader

---

修改完後重新開機，可以使用以下指令來確認是否成功開啟IOMMU(可以等到後面設定完再一起重新開機)

```shell
# dmesg | grep -e DMAR -e IOMMU
```

應該要出現包含一行`DMAR: IOMMU enabled`的內容

```shell
# dmesg | grep 'remapping'
```

視不同CPU的廠商，會有不同內容，有出現其中一行就OK

- `AMD-Vi: Interrupt remapping enabled`
- `DMAR-IR: Enabled IRQ remapping in x2apic mode`

---

2. 開機載入`vfio`module

編輯`/etc/modules`，加入以下內容

```modules
vfio
vfio_iommu_type1
vfio_pci
```

也許在其他教程還會多一個`vfio_virqfd`，不過在kernel6.2後這個模組不再需要加了，已經包含在`vfio`內了

編輯完後使用`update-initramfs -u -k all`來更新initramfs

3. 設定顯卡隔離

使用指令來確認要直通的PCI device是不是單獨在一個iommu group裡。

有些主板的南橋分配會不同，導致插在南橋引出的PCIE上會和其他設備分在同一組，解決方式就是換個插槽，或是插到直連CPU的PCIE上

```shell
# pvesh get /nodes/{nodename}/hardware/pci --pci-class-blacklist ""
```

輸出大概這樣

```txt
┌──────────┬────────┬──────────────┬────────────┬────────┬────────────────────────────────────────────────────
│ class    │ device │ id           │ iommugroup │ vendor │ device_name
╞══════════╪════════╪══════════════╪════════════╪════════╪════════════════════════════════════════════════════
│ 0x010601 │ 0x8d62 │ 0000:00:11.4 │         21 │ 0x8086 │ C610/X99 series chipset sSATA Controller [AHCI mode
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
│ 0x010601 │ 0x8d02 │ 0000:00:1f.2 │         27 │ 0x8086 │ C610/X99 series chipset 6-Port SATA Controller [AHC
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
│ 0x020000 │ 0x15a1 │ 0000:00:19.0 │         24 │ 0x8086 │ Ethernet Connection (2) I218-V
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
│ 0x020000 │ 0x1639 │ 0000:01:00.0 │         28 │ 0x14e4 │ NetXtreme II BCM5709 Gigabit Ethernet
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
│ 0x020000 │ 0x1639 │ 0000:01:00.1 │         28 │ 0x14e4 │ NetXtreme II BCM5709 Gigabit Ethernet
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
│ 0x030000 │ 0x0fc6 │ 0000:02:00.0 │         29 │ 0x10de │ GK107 [GeForce GTX 650]
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
│ 0x030000 │ 0x1049 │ 0000:03:00.0 │         30 │ 0x10de │ GF119 [GeForce GT 620 OEM]
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
│ 0x040300 │ 0x8d20 │ 0000:00:1b.0 │          0 │ 0x8086 │ C610/X99 series chipset HD Audio Controller
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
│ 0x040300 │ 0x0e1b │ 0000:02:00.1 │         29 │ 0x10de │ GK107 HDMI Audio Controller
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
│ 0x040300 │ 0x0e08 │ 0000:03:00.1 │         30 │ 0x10de │ GF119 HDMI Audio Controller
├──────────┼────────┼──────────────┼────────────┼────────┼────────────────────────────────────────────────────
...
```

一般來說一張顯卡會有至少兩個設備，顯示和聲卡兩部分，如果是有typec的卡還會有更多的設備

將要直通的設備的vendor id和device id記起來，我要直通GK107(GTX650)，因此就是`10de:0fc6`和`10de:0e1b`

編輯`/etc/modprobe.d/vfio.conf`來設定vfio參數，ids後面加的就是要直通的設備id，後面softdep是讓核心在載入這些模組前優先載入vfio，避免被driver的模組搶先，這邊只有NVIDIA的driver module，如果使用intel或是amd的顯卡需要改為對應的模組

```conf
options vfio-pci ids=10de:0fc6,10de:0e1b
softdep nouveau pre: vfio-pci
softdep nvidia pre: vfio-pci
softdep nvidiafb pre: vfio-pci
softdep nvidia_drm pre: vfio-pci
softdep drm pre: vfio-pci
```

如果不想用softdep，也可以直接把driver的module給blacklist，編輯`/etc/modprobe.d/blacklist.conf`，這邊也只有blacklist NVIDIA的driver module，如果使用intel或是amd的顯卡需要改為對應的模組

```conf
blacklist nouveau
blacklist i2c_nvidia_gpu
blacklist snd_hda_intel
blacklist snd_hda_codec_hdmi
blacklist nvidia*
```

做完這些設定後重新開機，並執行`dmesg | grep vfio`如果有看到輸出，那應該就是正確啟動了

---

如果順利的話，只需要做完這些設定、安裝一個虛擬機，並加入PCI device就大功告成了

![](add_pci_device.png)

建議把PCI-Express勾起來，如果是直通顯卡則Primary GPU也建議勾起來

此時把虛擬機打開應該可以在直通的顯卡連接的螢幕上看到畫面，並且console不再有顯示，等到啟動到系統並安裝驅動，如果都沒問題，那就算是成功了

如果沒有成功，可以嘗試我底下提供的其他解法

## 問題1：顯卡無顯示

問題的具體症狀大概包括這幾個，直通的顯卡沒有輸出任何內容，在Windows Client上可以看到GTX650的裝置，也可以打上驅動，但是會有code 43的error

由於我使用的NVIDIA driver版本>465，所以不是因為驅動發現是虛擬機才顯示code43(在465的版本前，如果驅動發現是虛擬機會報code 43 error，需要給vBIOS打patch，並隱藏虛擬機特徵才能通過)

在這個問題上我多設定了幾個地方，但是具體是哪些設定有起效果，我也不知道，反正這個問題解決了

因為在確認remapping的時候看到`dmesg | grep 'remapping'`的輸出還有一行`x2apic: IRQ remapping doesn't support X2APIC mode`，因此kernel parameter加上了`intremap=no_x2apic_optout`

1. 給kernel parameter加上`video=efifb:off`和`intremap=no_x2apic_optout`[^1]，改完需重啟
2. 在`options vfio-pci ids=10de:0fc6,10de:0e1b`後加上`disable_vga=1`
3. 更換vBIOS(顯卡的BIOS)

由於顯卡直通會需要顯卡有支援UEFI的vBIOS，可以使用這些指令來把目前顯卡的vBIOS dump出來

```shell
# # 0000:02:00.0是pci device的位置
# cd /sys/bus/pci/devices/0000:01:00.0/
# echo 1 > rom
# cat rom > /tmp/image.rom
# # 如果rom顯示沒有這個檔案，無法dump出來，可能重新啟動可以
# echo 0 > rom
```

然後使用rom-parser來驗證

```shell
# git clone https://github.com/awilliam/rom-parser
# cd rom-parser
# make
# ./rom-parser /tmp/image.rom
```

輸出應該要像這樣，需要有顯示type 3 (EFI)

```txt
Valid ROM signature found @0h, PCIR offset 190h
        PCIR: type 0 (x86 PC-AT), vendor: 10de, device: 0fc6, class: 030000
        PCIR: revision 0, vendor revision: 1
Valid ROM signature found @fc00h, PCIR offset 1ch
        PCIR: type 3 (EFI), vendor: 10de, device: 0fc6, class: 030000
        PCIR: revision 3, vendor revision: 0
                EFI: Signature Valid, Subsystem: Boot, Machine: X64
        Last image
```

如果只有type 0，就需要去找對應的有UEFI支援的vBIOS，可以嘗試在[Tech Powerup](https://www.techpowerup.com/vgabios/)上搜尋，只是我在上面找到的vBIOS也是沒有EFI支援的vBIOS，後來是在[微星的論壇](https://forum-en.msi.com/index.php?threads/uefi-gop-bios-for-card-n650-1gd5-ocv1.269258/)上找到有人提供有EFI支援的vBIOS

我找到對應的vBIOS後使用`nvflash`刷入顯卡上，可以從[Tech Powerup](https://www.techpowerup.com/download/nvidia-nvflash/)上下載`nvflash`

接著在虛擬機的設定中指定vBIOS為rom file

將剛剛的vBIOS移到`/usr/share/kvm/`底下

```shell
# mv /tmp/image.rom /usr/share/kvm/vbios.rom
```

編輯`/etc/pve/qemu-server/105.conf`，數字是要直通的虛擬機的vmid，在`hostpci0`後面加上`romfile`，如果不設定romfile，則開機的時候不會顯示，直到進入系統

```conf
hostpci0: 0000:02:00,pcie=1,x-vga=1,romfile=vbios.rom
```

經過這些設定並重新開機後，問題1解決了，症狀變為問題2

## 問題2：無法安裝驅動

具體的症狀大概是只要Windows Client不裝驅動，那麼就可以正常開機，但是使用連在顯卡上的螢幕裝驅動會裝到一半直接黑屏死機，如果是使用RDP遠程連入裝驅動，雖然可以成功安裝，但是開機到進入系統前就會直接死機重鎧(只能開機到轉圈的地方，猜測是載入驅動的時候內核的問題)

經過一些googling後在[UNRAID的論壇](https://forums.unraid.net/topic/131013-windows-10-vm-crashesbreaks-during-nvidia-driver-install-for-gpu-passthrough/)[^2]和[Proxmox的論壇](https://forum.proxmox.com/threads/gpu-passthrough-issue.109074/)[^3]上找到了和我相同問題的提問，給出的解法加上一些kernel parameter後在Windows Client中使用安全模式安裝驅動，並將顯卡的MSI(Message Signaled Interrupts)打開，我嘗試後成功解決問題，直通成功。

1. kernel parameter加上`video=efifb:off video=vesafb:off video=simplefb:off`，之前已經加過了`video=efifb:off`，所以實際只加`video=vesafb:off video=simplefb:off`，更改完需重啟
2. 將顯卡的vBIOS打上patch，我不確定這一步有沒有影響，這一步一般來說是用來規避NVIDIA舊版本驅動對虛擬機的限制

使用HEX Editor打開剛剛dump出來或是下載的vBIOS文件，找到55AA這兩個值，並且他後面應該會有VIDEO的文字，將55AA前面的內容全部刪掉

![](patch_vBIOS.png)

3. 開啟虛擬機，並進入安全模式

按下Meta+R，輸入`msconfig`，並在開機的選項中把安全啟動勾起來，這樣之後開機都會進入安全模式，直到把安全啟動取消

![](safe_mode_1.png)

![](safe_mode_2.png)

4. 如果之前有嘗試安裝NVIDIA驅動則下載[DDU](https://www.guru3d.com/download/display-driver-uninstaller-download/)，來徹底刪除驅動並重啟虛擬機
5. 在安全模式中安裝NVIDIA驅動
6. 將顯卡開啟MSI

可以參考[這篇文章](https://forums.guru3d.com/threads/windows-line-based-vs-message-signaled-based-interrupts-msi-tool.378044/)[^4]來確認目前設備的中斷形式是哪一種，以及如何手動開啟MSI

要開啟MSI可以先嘗試使用[MSI_util](https://www.mediafire.com/file/ewpy1p0rr132thk/MSI_util_v3.zip/file)，如果有在列表中看到顯卡，就只需要把後面的勾勾起來就好

不過我的顯卡沒有出現在設備列表中，所以我使用更改註冊表來開啟MSI

首先開啟`regedit`，並進入`\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Enum\PCI\`

在這個項目底下找到對應顯卡對應的子機碼，前面有提到顯卡的id是`10de:0fc6`，因此進入`VEN_10DE&DEV_0FC6&SUBSYS_8A961462&REV_A1\`，他底下應該只有一個子機碼，在裡面找到`Device Parameters\Interrupt Management\`，如果底下沒有`MessageSignaledInterruptProperties`的子機碼，就直接建立即可，並在這個機碼中建立一個DWORD，名稱為**MSISupported**，值設定為1即開啟MSI

註：更改註冊表有風險，最好先進行備份

![](open_msi.png)

我在完成開啟MSI，並重開虛擬機後就成功直通顯卡了，且沒有任何錯誤

## 其他疑難雜症

這邊大概會簡略介紹我看到的其他問題之解法，但不一定適用，建議參考文章[^5][^6][^7]和搜尋進行評估

- code 43

在NVIDIA驅動465版本之前，它會檢查機器是不是虛擬機，如果是，即使直通成功，驅動也會報錯誤代碼43

對應的解決方法就是隱藏虛擬機和修補vBIOS，修補vBIOS的方式前面已經提過了，這裡就不重提了

隱藏虛擬機可以透過在虛擬機的設定檔加上參數[^8]

```conf
cpu: host,hidden=1,flags=+pcid
args: -cpu 'host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX,kvm=off'
```

或是可以在網路上搜尋其他隱藏的方法並去嘗試，不過現在應該不需要這部份了，前幾年NVIDIA放開對於虛擬機的限制，現在不會因為虛擬機而報錯code43

- 開啟專用IOMMU group

如果IOMMU並沒有給要直通的設備單獨分組可以嘗試給kernel parameter加上`pcie_acs_override=downstream`，不過wiki[^6]也有提到這是最後的選擇，可能會有風險，VM可以讀取所有Proxmox Host的記憶體

## 參考
[^1]: [[SOLVED] - x2apic: IRQ remapping doesn't support X2APIC mode | Proxmox Support Forum](https://forum.proxmox.com/threads/x2apic-irq-remapping-doesnt-support-x2apic-mode.118349/)
[^2]: [Windows 10 VM crashes/breaks during Nvidia driver install for GPU passthrough. - VM Engine (KVM) - Unraid](https://forums.unraid.net/topic/131013-windows-10-vm-crashesbreaks-during-nvidia-driver-install-for-gpu-passthrough/)
[^3]: [[SOLVED] - GPU Passthrough issue | Proxmox Support Forum](https://forum.proxmox.com/threads/gpu-passthrough-issue.109074/)
[^4]: [Windows: Line-Based vs. Message Signaled-Based Interrupts. MSI tool. | guru3D Forums](https://forums.guru3d.com/threads/windows-line-based-vs-message-signaled-based-interrupts-msi-tool.378044/)
[^5]: [PCI passthrough via OVMF - ArchWiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
[^6]: [PCI Passthrough - Proxmox VE](https://pve.proxmox.com/wiki/PCI_Passthrough)
[^7]: [PCI(e) Passthrough - Proxmox VE](https://pve.proxmox.com/wiki/PCI(e)_Passthrouhgh)
[^8]: [PCI passthrough Error code 43 | Proxmox Support Forum](https://forum.proxmox.com/threads/pci-passthrough-error-code-43.56462/)
[^9]: [[TUTORIAL] - PCI/GPU Passthrough on Proxmox VE 8 : Installation and configuration | Proxmox Support Forum](https://forum.proxmox.com/threads/pci-gpu-passthrough-on-proxmox-ve-8-installation-and-configuration.130218/)
[^10]: [Ultimate Beginner's Guide to Proxmox GPU Passthrough](https://gist.github.com/KasperSkytte/6a2d4e8c91b7117314bceec84c30016b)
[^11]: [PCI通透 - HackMD](https://hackmd.io/@davidho9713/pve_pci_passthrough)
[^12]: [PVE虚拟机安装流程（推荐） · 游戏串运营教程 · 看云](https://www.kancloud.cn/baoge/gamecc/1291968#N43_113)
[^13]: [Proxmox显卡直通 - Noodlefighter's Wiki](https://wiki.noodlefighter.com/%E8%AE%A1%E7%AE%97%E6%9C%BA/%E8%99%9A%E6%8B%9F%E6%8A%80%E6%9C%AF/proxmox%E6%98%BE%E5%8D%A1%E7%9B%B4%E9%80%9A/)
[^14]: [Proxmox VE 直通显卡方案及解决N卡Code43 - moper 工作技术展示](https://blog.moper.net/2909.html)