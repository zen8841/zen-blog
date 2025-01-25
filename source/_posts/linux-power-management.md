---
title: Linux上的電源管理
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - linux
  - power
excerpt: 記錄如何在Linux上進行電源管理來達成省電及延長電池續航
date: 2025-01-26 00:18:11
updated: 2025-01-26 00:18:11
index_img:
---


# 前言

這篇文章主要是偏向我對於網路上關於Linux電源管理的內容的整理，以及我自己在筆電上實作的結果，如果想要更加了解可以查看文章中的參考資料。

我的筆電型號是Swift SFX14-41G，CPU為5800U，因此在底下的實作中有針對只能在AMD與Ryzen上進行的電源管理設定。

# CPU

如果不算GPU的話，一個系統中CPU大概率是最耗電的部分，因此大部分的電源管理都是針對CPU來進行，像是限制運行頻率之類的，CPU如果可以省電，那麼將可以給電池續航帶來顯著的提升。

CPU的省電主要是靠調節CPU運行的頻率，而Linux中調整CPU頻率依靠CPUFreq子系統，CPUFreq定義了`Scaling governors `和`Scaling drivers`兩個選項，可以透過改變這兩個選項來改變CPU的調度，更詳細的內容，可以查看[Archwiki](https://wiki.archlinux.org/title/CPU_frequency_scaling)[^1]裡的介紹。

`Scaling governors `和`Scaling drivers`的功能如下

- Scaling governors： 調度策略，決定CPU頻率要跑哪個頻率，並傳遞給Scaling drivers執行，大部分的driver都定義了四種調度器，每種driver提供的調度策略雖然名稱相同，但調度並不完全一樣，大部分情況下ondemand或schedutil可以滿足多數需求
  - performance： 始終維持CPU最大頻率
  - powersave： 始終維持CPU最小頻率
  - ondemand： 在負載時使用高頻率，閒置時逐漸降低到最低
  - schedutil： 根據CPU使用率調整頻率，且調速快
- Scaling drivers： 調度驅動，直接與CPU交互，將governors決定的頻率傳到CPU，調整CPU的頻率

## Scaling drivers

因為我使用AMD的CPU，這部份就只會提AMD CPU可使用的調度器，完整列表可參考[Archwiki](https://wiki.archlinux.org/title/CPU_frequency_scaling#Scaling_drivers)[^1]

在AMD平台上可以使用的driver有3種，`acpi_cpufreq`、`amd_pstate`和`amd_pstate_epp`，其中`amd_pstate`又有三種模式，分別為`passive`、`guided`和`active`，設定為`active`模式時使用的driver為`amd_pstate_epp`，可以在kernel command line parameters中設定選擇哪一種driver的哪種模式[^2]

- `amd_pstate=disable`使用的driver為`acpi_cpufreq`，此driver為核心中CPUFreq負責，使用ACPI提供的CPU能效等級來調度，好處是基本所有的CPU都可使用此driver，但是調度較粗糙，無法達到精細的調度來省電
- `amd_pstate=active`使用的driver為`amd_pstate_epp`，在此模式下由CPPC(Collaborative Power and Performance Control)決定運行頻率，只提供performance與powersave兩種governor，driver將governor與EPP(energy performance preference)的值提供給CPPC來決定頻率，由於需要CPU內有CPPC，因此只有zen2後支援此driver；這個driver除了可以調整governor，還可以額外設定EPP，可以理解成在governor的電源指示上做微調，有4種EPP，性能和耗電從高到低分別是performance、balance_performance、balance_power、power，這個部分就按需選擇即可
- `amd_pstate=guided`使用的driver為`amd_pstate`的`guided`模式，此模式同樣使用CPPC，但是可以在OS內設定最高及最低性能，CPPC會在範圍之間自動決定運行頻率，同樣的，因為對CPPC有需求，只有zen2後可以使用此driver
- `amd_pstate=passive`使用的driver為`amd_pstate`的`passive`模式，此模式下由driver來決定運行頻率，類似`acpi_cpufreq`，但可能會好一點

目前大部分的核心編譯設定`CONFIG_X86_AMD_PSTATE_DEFAULT_MODE=3`，即核心在沒有設定時預設使用`amd_pstate_epp`，可以在`/proc/config.gz`裡的Config中找到目前運行核心的編譯設定。

核心編譯設定的定義可以在kernel的源始碼中找到，在[drivers/cpufreq/Kconfig.x86](https://github.com/torvalds/linux/tree/master/drivers/cpufreq/Kconfig.x86)中`config X86_AMD_PSTATE_DEFAULT_MODE`底下有定義設定數值對應預設使用的driver

- 1 -> Disabled
- 2 -> Passive
- 3 -> Active (EPP)
- 4 -> Guided

---

至於這幾種driver要選擇哪一種的問題，我參考了[phoronix的測試](https://www.phoronix.com/review/amd-pstate-epp-ryzen-mobile)[^3]，`amd-pstate-epp`的表現很好，在CPU閒置時他可以達到接近`acpi_freq`在`powersave` governors的耗電量，但是在有需求時它可以自動boost到較高的頻率，而不是像其他driver的`powersave` governor，只會保持在最低頻率，幾乎不可使用。我個人的體驗覺得`amd-pstate-epp`的表現不錯，相比於`passive`或`guided`在續航上有較好的表現。

{% fold info @一些其他參考資料 %}

[AMD P-State and AMD P-State EPP Scaling Driver Configuration Guide : r/linux](https://www.reddit.com/r/linux/comments/15p4bfs/amd_pstate_and_amd_pstate_epp_scaling_driver/)

[AMD Posts P-State Linux Patches For New "Guided Autonomous Mode" - Phoronix](https://www.phoronix.com/news/AMD-P-State-Guided-Auto)

[In case anyone wants to use the AMD pstate frequency scaling driver : r/linux_gaming](https://www.reddit.com/r/linux_gaming/comments/15hjq8l/in_case_anyone_wants_to_use_the_amd_pstate/)

{% endfold %}

## Setting

可以透過操作`/sys`底下的檔案來改變kernel設定的driver與governor

### driver

```shell
# echo <driver> > /sys/devices/system/cpu/amd_pstate/status
```

\<driver>的值可以為active/guided/passive/disable

### governor

```shell
# echo <governor> > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

\<governor>的值可以是`/sys/devices/system/cpu/cpu*/cpufreq/scaling_available_governors`中提供的選項

### EPP

```shell
# echo <EPP> > /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference
```

\<EPP>的值可以是`/sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference`中提供的選項

### frequencies

```shell
# # 設定CPU以指定頻率運作
# cpupower frequency-set -f <clock_freq>
# # 設定CPU頻率上限
# cpupower frequency-set -u <clock_freq>
# # 或是使用/sys操作
# echo <clock_freq> > /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
# #設定CPU頻率下限
# cpupower frequency-set -d <clock_freq>
# # 或是使用/sys操作
# echo <clock_freq> > /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
```

### boost

設定是否開啟turbo boost

```shell
# # 開啟turbo boost
# echo 1 > /sys/devices/system/cpu/cpufreq/boost # amd/intel
# echo 0 > /sys/devices/system/cpu/intel_pstate/no_turbo # intel
# # 關閉turbo boost
# echo 0 > /sys/devices/system/cpu/cpufreq/boost # amd/intel
# echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo # intel
```

# 工具

前面章節介紹了調整CPU省電的方法，但是如果每次拔下插頭都要做一堆設定總是不方便的，雖然可以自己寫一個腳本被udev rules觸發或是寫一個daemon監控acpi電源事件，不過目前linux上已經有很多成熟的電源管理專案了，沒有必要重複造輪子。

目前linux常見的電源管理方案有以下幾種

- tlp： 大部分人的推薦，省電設定包含了CPU、無線、USB、GPU等等，幾乎所有設備的省電設定都可設置，而且可以使用tlpui來用GUI進行設定，可以設定一套接AC時的設定和使用電池時的設定，可以在離電後自動切換，也可手動切換模式，堪稱電源管理工具的瑞士刀，只是需要良好的設定。
- power-profiles-daemon： 一個較新的電源管理工具，只針對CPU做幾項設定，可使用tlp重現相同的功能[^4] (後文會說明)，ppd會內置3種配置power、balanced、performance，優點是不需配置，就可以達成不差的效果，雖然只針對CPU做省電設定(前文有提到大部分耗電是CPU造成的)，而且對桌面環境有很好的支持，KDE Plasma和Gnome都可以直接在Taskbar上進行三種配置的切換。
- tuned： 和tlp相似，有豐富的省電選項可以設置，這款工具由RetHat負責維護，因此RH系的Linux大部分都是使用這個，Fedora在41後預設安裝tuned而不是ppd[^5]；使用人數較tlp少，但最近tuned推出了tuned-ppd這個提供ppd的兼容層，可以達成和ppd一樣對桌面環境很好的支持，同時又有和tlp一樣豐富的功能，也許我之後會嘗試這個工具。
- auto-cpufreq： 也是近幾年討論比較多的工具，它一樣只針對CPU做設定，調整CPU的頻率，根據它的文檔，這個工具可以根據工作負載自動決定CPU的頻率來達到省電的功能，我個人是覺得CPU內的CPPC單元應該可以有更好的效果啦，畢竟判斷工作負載同樣需要算力，有興趣可以自己測試看看。
- thermald： 這個工具和省電其實沒有很大的關係，它主要關注發熱，可以設定達到溫度自動降頻，好處是和前面幾個工具都不衝突，前面幾個工具一般都是建議只裝其中一個，畢竟每個工具都是用同樣的操作方式進行省電設定，如果幾個都在跑大概率會衝突。
- powertop： 這個工具其實也和省電沒有什麼關係，主要是它可以查看目前系統CPU的耗電量、idle、C狀態之類的，不過他也有提供`--auto-tune`選項，不過這個選項的省電設定會很激進，可能會影響系統的使用，我一般只有在需要極致省電的時候才會在開啟了tlp下再執行`sudo powertop --auto-tune`

在tlp、ppd、tuned和auto-cpufreq這4個工具中，我曾經使用過tlp和ppd，目前是又從ppd換回tlp了，大概可以多提升幾十分鐘的續航，畢竟多了一些外圍設備的省電；我個人建議筆電一定要進行省電設定，如果不想查文檔，可以直接裝ppd即可，如果不做電源管理，可能windows下能用6小時，Linux下只剩2~3小時，哪怕只有ppd，也可以把續航拉回正常的水平

{% fold info @一些關於電源管理工具的討論 %}

[Tuned-ppd vs tlp - Fedora Discussion](https://discussion.fedoraproject.org/t/tuned-ppd-vs-tlp/141203/3)

[Confused about power saving: TLP vs tuned vs powertop, etc : r/Fedora](https://www.reddit.com/r/Fedora/comments/p7uz33/confused_about_power_saving_tlp_vs_tuned_vs/)

[TLP or Power-Profiles for Battery Life - General system / Newbie - EndeavourOS](https://forum.endeavouros.com/t/tlp-or-power-profiles-for-battery-life/37921)

[[TRACKING] PPD v TLP for AMD Ryzen 7040 - Framework Laptop 13 / Linux - Framework Community](https://community.frame.work/t/tracking-ppd-v-tlp-for-amd-ryzen-7040/39423)

[tlp vs. power-profiles-daemon : r/Fedora](https://www.reddit.com/r/Fedora/comments/qpaa4g/tlp_vs_powerprofilesdaemon/)

這些討論都只是參考，如果想要達成最好的省電，還是要在自己的電腦上做測試，我是懶得做精細的測試了，就tlp用著就很好

{% endfold %}

## TLP

這段主要會講我對tlp在使用電池下的設定，我設定在連接AC下是什麼省電都關閉，抱持最大性能的運行

在使用電池時我設定tlp的CPU設定與ppd的power模式下相同，並做了一些修改，ppd的三種模式的設定相當於tlp中的`PLATFORM_PROFILE_ON_AC/BAT`、`CPU_ENERGY_PERF_POLICY_ON_AC/BAT`和`CPU_BOOST_ON_AC/BAT`[^4]

ppd的performance設定等於

```conf
PLATFORM_PROFILE_ON_AC/BAT=performance
CPU_ENERGY_PERF_POLICY_ON_AC/BAT=performance
CPU_BOOST_ON_AC/BAT=1
```

balanced

```conf
PLATFORM_PROFILE_ON_AC/BAT=balanced
CPU_ENERGY_PERF_POLICY_ON_AC/BAT=balance_performance
CPU_BOOST_ON_AC/BAT=0
```

power

```conf
PLATFORM_PROFILE_ON_AC/BAT=low-power
CPU_ENERGY_PERF_POLICY_ON_AC/BAT=power
CPU_BOOST_ON_AC/BAT=0
```

我個人在使用電池下，有關處理器的設定為

```conf
CPU_DRIVER_OPMODE_ON_BAT=active
CPU_SCALING_GOVERNOR_ON_BAT=powersave
CPU_ENERGY_PERF_POLICY_ON_BAT=power
CPU_BOOST_ON_BAT=0
```

可以視自己的機器不同來做調整，tlpui裡都有對每個選項做相關設定

CPU以外的部分我基本沒有更改什麼設定，如果有發現像是USB斷線或是網路/音訊設備閒置後斷線，可以把那部份的省電關掉即可

## other Tools

- cpupower： 前面就有用到的工具，可以設定CPU使用的governors、運行的頻率，也可以查看目前CPU調度相關的資訊，`cpupower frequency-info`
- turbostat： 用來查看CPU每顆核心boost的狀態，也會顯示C狀態的比例，一般不太會需要這個工具
- lm_sensors： 統整電腦內各種感測器的狀態，可以獲得各種零部件的溫度或是風扇轉速

## Ryzen only Tools

- ryzenadj： 可以理解為等同Intel XTU的工具，如果BIOS有開放設置，可以在OS內設定CPU的功耗、溫度牆或是超頻，只是我電腦BIOS沒有開放這些選項，所以我無法使用
- zenpower3/zenpower： 提供zenpower的dkms模組，可以更精確的獲得CPU內感測器的數值，如電壓、電流等
- zenmonitor3/zenmonitor： 將zenpower的資訊列表呈現在GUI上的工具

![](zenmonitor3.png)

- amdctl： 纇似ryzenadj，同樣無法使用
- zenstate： 調電壓曲線的工具，好像也可以超頻，同樣無法使用
- ryzen-ppd： 一個用來幫助ryzen省電的工具，但我跑不動，看起來也沒什麼人用
- ryzen_smu： 一個CPU內SMU(System Management Unit)驅動的核心模組

不過AMD沒有公布SMU可以使用的指令，目前只有Matisse和Vermeer的指令有被試出來，在ryzen_smu的[github](https://github.com/leogx9r/ryzen_smu/blob/master/docs/rsmu_commands.md)[^6]上有給出，其他處理器系列能使用的指令只有Test和GetSMUVersion這兩個，我個人測試5800U也只有這兩個指令會有反應

```shell
# # test，Arg0輸入數字X，回傳X+1
# # 先隨便寫入一些數值
# printf '%0*x' 48 142000 | fold -w 2 | tac | tr -d '\n' | xxd -r -p | tee /sys/kernel/ryzen_smu_drv/smu_args
# cat /sys/kernel/ryzen_smu_drv/smu_args | xxd
00000000: b02a 0200 0000 0000 0000 0000 0000 0000  .*..............
00000010: 0000 0000 0000 0000                      ........
# # 執行SMU指令
# printf '\x01' | tee /sys/kernel/ryzen_smu_drv/rsmu_cmd
# # 查看回傳，發現第一個參數+1
# cat /sys/kernel/ryzen_smu_drv/smu_args | xxd
00000000: b12a 0200 0000 0000 0000 0000 0000 0000  .*..............
00000010: 0000 0000 0000 0000                      ........


# # GetSMUVersion，無參數，回傳小端排序的版本
# # 執行SMU指令
# printf '\x02' | tee /sys/kernel/ryzen_smu_drv/rsmu_cmd
# # 查看回傳
# cat /sys/kernel/ryzen_smu_drv/smu_args | xxd
00000000: 002c 4000 0000 0000 0000 0000 0000 0000  .,@.............
00000010: 0000 0000 0000 0000                      ........
# # 回傳002c40，十進位為0 44 64，因為是小端排序，所以SMU正確版本為64.44.0
```

## 參考

[^1]: [CPU frequency scaling - ArchWiki](https://wiki.archlinux.org/title/CPU_frequency_scaling)
[^2]: [amd-pstate CPU Performance Scaling Driver — The Linux Kernel  documentation](https://docs.kernel.org/admin-guide/pm/amd-pstate.html)
[^3]:[Ryzen Mobile Power/Performance With Linux 6.3's New AMD P-State EPP Driver - Phoronix](https://www.phoronix.com/review/amd-pstate-epp-ryzen-mobile)
[^4]: [power-profiles-daemon — TLP 1.7.0 documentation](https://linrunner.de/tlp/faq/ppd.html)
[^5]: [Changes/TunedAsTheDefaultPowerProfileManagementDaemon - Fedora Project Wiki](https://fedoraproject.org/wiki/Changes/TunedAsTheDefaultPowerProfileManagementDaemon)
[^6]: [ryzen_smu/docs/rsmu_commands.md at master · leogx9r/ryzen_smu](https://github.com/leogx9r/ryzen_smu/blob/master/docs/rsmu_commands.md)
