---
title: 如何建立 Proxmox 的鏡像站
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - Proxmox
  - server
  - linux
excerpt: 記錄一下 CCNS 的 Proxmox 鏡像站是如何建立的
date: 2024-11-24 03:47:10
updated: 2024-11-24 03:47:10
index_img:
---


# 前言

Proxmox 官方提供的鏡像只有 http://download.proxmox.com、https://download.proxmox.com 只包含了部分的 repository，並且沒有提供 rsync、ftp 等連接方式，因為鏡像中的軟連結都沒辦法從 http/https 中下載到，如果直接鏡像那麼就會完整的下載，會浪費容量在重複且相同版本的軟體上。( 像 debian/ubuntu/archlinux 之類的鏡像，軟體包的檔案可能會放在集中的資料夾中( 像是 debian 的/pool )，不同大版本的資料夾中只有指向對應軟體包文件的軟連結 )

最開始我是使用`apt-mirror`來進行鏡像的工作，但就和前面提到的，從 http 進行鏡像沒辦法得知軟連結之類的訊息，在今年8月左右重整 CCNS 的 Proxmox 鏡像的過程中，我使用了`proxmox-offline-mirror`來替換`apt-mirror`進行鏡像( 具體原因後文會介紹 )，當然也不一定要使用這種方式來鏡像， [twds](https://mirror.twds.com.tw) 就是使用 [tunasync](https://github.com/tuna/tunasync) ( 北京清華開發的一套鏡像站工具之一 )。

# Proxmox-offline-mirror

會選擇此工具的最大原因就是他擁有去重的功能，用它下載的檔案結構如下，軟體包的文件都儲存在`.pool`底下，並且計算了 hash，在各個 repository 中只有指向文件的軟連結，如果有相同版本的同一個軟體包在不同的 repository 中，就不會重新下載( 因為 hash 一樣 )，而是直接新建軟連結。

```shell
# tree -L 2 -a /path/to/mirror/proxmox/download/
/path/to/mirror/proxmox/download/
├── .pool
│   ├── .lock
│   ├── sha256
│   └── sha512
├── pve_bookworm
│   └── 2024-11-21T19:14:17Z
│       └── dists
│           └── bookworm
│               ├── InRelease
│               ├── pve-no-subscription
│               ├── pvetest
│               ├── Release
│               └── Release.gpg
├── pve_bullseye
│   └── 2024-11-21T19:16:40Z
│       └── dists
│           └── bullseye
│               ├── InRelease
│               ├── pve-no-subscription
│               ├── pvetest
│               ├── Release
│               └── Release.gpg
└── pve_buster
    └── 2024-11-21T19:19:34Z
        └── dists
            └── buster
                ├── InRelease
                ├── pve-no-subscription
                ├── pvetest
                ├── Release
                └── Release.gpg
```

## 設定

使用 proxmox-offline-mirror 需要設定，設定檔預設使用`/etc/proxmox-offline-mirror.cfg`，不過可以使用`--config`改變使用的設定檔。

設定鏡像可以使用指令直接新增，或是使用互動式新增

- 互動式，需注意不要新增 Debian 的鏡像

```shell
# proxmox-offline-mirror setup --config /etc/proxmox-offline-mirror.cfg
Initializing new config.

Select Action:
   0) Add new mirror entry
   1) Add new subscription key
   2) Quit
Choice ([0]): 0
Guided Setup ([yes]):
Select distro to mirror
   0) Proxmox VE
   1) Proxmox Backup Server
   2) Proxmox Mail Gateway
   3) Proxmox Ceph
   4) Debian
Choice: 0
Select release
   0) Bookworm
   1) Bullseye
   2) Buster
Choice ([0]): 0
Select repository variant
   0) Enterprise repository
   1) No-Subscription repository
   2) Test repository
Choice ([0]): 1
Should missing Debian mirrors for the selected product be auto-added ([yes]): no
Enter mirror ID ([pve_bookworm_no_subscription]):
Enter (absolute) base path where mirrored repositories will be stored ([/var/lib/proxmox-offline-mirror/mirrors/]): /path/to/mirror/proxmox/download/
Should already mirrored files be re-verified when updating the mirror? (io-intensive!) ([yes]):
Should newly written files be written using FSYNC to ensure crash-consistency? (io-intensive!) ([yes]):
Config entry 'pve_bookworm_no_subscription' added
Run "proxmox-offline-mirror mirror snapshot create --config 'test.cfg' 'pve_bookworm_no_subscription'" to create a new mirror snapshot.

Existing config entries:
mirror 'pve_bookworm_no_subscription'

Select Action:
   0) Add new mirror entry
   1) Add new medium entry
   2) Add new subscription key
   3) Quit
Choice ([0]): 3
```

- 指令

```shell
# proxmox-offline-mirror config mirror add --config /etc/proxmox-offline-mirror.cfg --architectures amd64 --architectures all --base-dir /path/to/mirror/proxmox/download/ --id pve_bookworm_no_subscription --ignore-errors false --key-path /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg --repository 'deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription' --sync true --verify true
```

這兩種方式產生的設定檔相同，如下

- 需注意`key-path`的位置是否有對應的 gpgkey，如果沒有可以到 [http://download.proxmox.com/debian/](http://download.proxmox.com/debian/) 下載對應的 key，`proxmox-ve-release-6.x.gpg`是對應 buster，往前同理。
- 設定檔中的`sync`代表是否要使用 fsync 寫入檔案
- 設定檔中的`verify`代表是否在鏡像的時候要不要檢查存在的檔案是否正確( 重新下載並計算 hash 檢查 )，或是默認存在的檔案正確

```cfg
mirror: pve_bookworm_no_subscription
        architectures amd64
        architectures all
        base-dir /path/to/mirror/proxmox/download/
        ignore-errors false
        key-path /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg
        repository deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
        sync true
        verify true
```

如果想要同時鏡像測試版的 repository，使用互動式的新增方式會再創一個`pve_bookworm_test`的條目，可以直接修改設定檔來新增要鏡像的 repository，只是要確認設定檔是否正確。

修改後的設定檔。可以直接在`repository`後方多新增想要鏡像的 repository，也將`id`修改為整個版本的集合名。

```cfg
mirror: pve_bookworm
        architectures amd64
        architectures all
        base-dir /path/to/mirror/proxmox/download/
        ignore-errors false
        key-path /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg
        repository deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription pvetest
        sync true
        verify true
```

---

從 buster 向後算，截至目前(2024/11/23)， Proxmox 提供的所有鏡像包括這些

```txt
# proxmox-offline-mirror config mirror list --config /etc/proxmox-offline-mirror.cfg
┌───────────────────────┬───────────────────────────────────────────────────────────────────────────────────────┬───────────────────────────────────┬────────┬──────┐
│ ID                    │ repository                                                                            │ base-dir                          │ verify │ sync │
╞═══════════════════════╪═══════════════════════════════════════════════════════════════════════════════════════╪═══════════════════════════════════╪════════╪══════╡
│ ceph-luminous_buster  │ deb http://download.proxmox.com/debian/ceph-luminous buster main test                 │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ ceph-nautilus_buster  │ deb http://download.proxmox.com/debian/ceph-nautilus buster main test                 │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ ceph-octopus_bullseye │ deb http://download.proxmox.com/debian/ceph-octopus bullseye main test                │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ ceph-octopus_buster   │ deb http://download.proxmox.com/debian/ceph-octopus buster main test                  │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ ceph-pacific_bullseye │ deb http://download.proxmox.com/debian/ceph-pacific bullseye main test                │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ ceph-quincy_bookworm  │ deb http://download.proxmox.com/debian/ceph-quincy bookworm main no-subscription test │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ ceph-quincy_bullseye  │ deb http://download.proxmox.com/debian/ceph-quincy bullseye main test                 │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ ceph-reef_bookworm    │ deb http://download.proxmox.com/debian/ceph-reef bookworm no-subscription test        │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pbs-client_bookworm   │ deb http://download.proxmox.com/debian/pbs-client bookworm main test                  │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pbs-client_bullseye   │ deb http://download.proxmox.com/debian/pbs-client bullseye main test                  │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pbs-client_buster     │ deb http://download.proxmox.com/debian/pbs-client buster main test                    │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pbs_bookworm          │ deb http://download.proxmox.com/debian/pbs bookworm pbs-no-subscription pbstest       │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pbs_bullseye          │ deb http://download.proxmox.com/debian/pbs bullseye pbs-no-subscription pbstest       │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pbs_buster            │ deb http://download.proxmox.com/debian/pbs buster pbs-no-subscription pbstest         │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pmg_bookworm          │ deb http://download.proxmox.com/debian/pmg bookworm pmg-no-subscription pmgtest       │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pmg_bullseye          │ deb http://download.proxmox.com/debian/pmg bullseye pmg-no-subscription pmgtest       │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pmg_buster            │ deb http://download.proxmox.com/debian/pmg buster pmg-no-subscription pmgtest         │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pve_bookworm          │ deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription pvetest       │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pve_bullseye          │ deb http://download.proxmox.com/debian/pve bullseye pve-no-subscription pvetest       │ /path/to/mirror/proxmox/download/ │      1 │    1 │
├───────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┼───────────────────────────────────┼────────┼──────┤
│ pve_buster            │ deb http://download.proxmox.com/debian/pve buster pve-no-subscription pvetest         │ /path/to/mirror/proxmox/download/ │      1 │    1 │
└───────────────────────┴───────────────────────────────────────────────────────────────────────────────────────┴───────────────────────────────────┴────────┴──────┘
```

# 同步腳本

從上面 proxmox-offline-mirror 下載的目錄結構可以看到這並不是正確的可以讓`apt`下載的鏡像目錄結構，因此需要一個腳本包裝並產生正確的目錄結構並軟連結到下載的目錄。

腳本大致分為三個部分

1. 最前面的部分是設定鏡像的位置，並檢查是否有其他同步正在進行及鏡像位置的資料夾是否存在，在每週一的00:00~01:59執行時，會使用 verify=true 的設定檔，日常的同步不進行完全檢查

存儲的結構如下， download 是儲存 proxmox-offline-mirror 下載的位置， logs 儲存日誌， packages 則是鏡像的根目錄

```shell
# tree -L 1 /path/to/mirror/proxmox
/mirrors/proxmox/
├── download
├── logs
├── packages
│   ├── debian
│   ├── images
│   └── iso
└── syncrepo.lck
```

2. 中間的部分負責 proxmox-offline-mirror 的下載及軟連結的工作，由於設定檔的`id`都是設定成`repository_version`的格式，因此可以很方便的知道目錄是該軟連結到哪裡
3. 最後一部分則是直接使用`wget`從 [http://download.proxmox.com/debian/](http://download.proxmox.com/debian/) 下載 CT 的 Template 和 ISO

最後只需要定時執行腳本並定時清理日誌即可完成鏡像的架設

```bash
#!/bin/bash

mirror_base_path="/path/to/mirror/proxmox"
log_path="$mirror_base_path/logs"
mirror_dir="$mirror_base_path/download"
symlink_dir="$mirror_base_path/packages/debian"
lock="$mirror_base_path/syncrepo.lck"
proxmox_offline_mirror_config="/etc/proxmox-offline-mirror.cfg"


# lock
exec 9>"${lock}"
flock -n 9 || echo "$(date): Another sync is running!" >> "$log_path/lock.log"
flock -n 9 || exit

[ ! -d "${mirror_dir}" ] && mkdir -p "${mirror_dir}"
[ ! -d "${symlink_dir}" ] && mkdir -p "${symlink_dir}"
[ ! -d "${log_path}/proxmox-offline-mirror" ] && mkdir -p "${log_path}/proxmox-offline-mirror"
[ ! -d "${log_path}/wget" ] && mkdir -p "${log_path}/wget"

if [[ "$(date +%u%H)" = "100" || "$(date +%u%H)" = "101" ]]; then
    proxmox_offline_mirror_config="/etc/proxmox-offline-mirror.cfg_verify"
    echo "verify..."
fi

echo "Start Syncing Proxmox Mirror at $(date)"

for dir in "${mirror_dir}"/*; do
    echo
    if ! [[ -d ${dir} ]]; then
        echo "${dir} not a directory"
        continue
    fi
    dir_name=$(basename "$dir")
    if [[ "$dir_name" = "lost+found" || "$dir_name" = ".pool" ]]; then
        continue
    fi
    echo "Start Syncing $dir_name"
    echo "Start Syncing $dir_name at $(date)" >> "$log_path/proxmox-offline-mirror/proxmox-offline-mirror.$dir_name.$(date +%Y%m%d).log"
    /usr/bin/proxmox-offline-mirror mirror snapshot create $dir_name --config $proxmox_offline_mirror_config >> "$log_path/proxmox-offline-mirror/proxmox-offline-mirror.$dir_name.$(date +%Y%m%d).log" 2>&1

    if [ $? -ne 0 ]; then
        rm -rfv $mirror_dir/$dir_name/*.tmp
        echo "$dir_name Sync Error, Watch proxmox-offline-mirror.$dir_name.time.log for more detail"
        continue
    fi

    if [[ -z $(ls -lA $dir | grep '^d') ]]; then
        echo "Skip $dir_name, no directory inside"
        continue
    fi
    rm -rfv $mirror_dir/$dir_name/*.tmp

    echo "Start Linking $dir_name"

    repo_name=$(echo $dir_name | cut -d _ -f 1)
    repo_version=$(echo $dir_name | cut -d _ -f 2)
    latest_subdir=$(ls -td "$dir"/*/ | head -n 1)
    latest_subdir_name=$(basename $latest_subdir)
    symlink_target_prefix=$(echo $latest_subdir | cut -d / -f 4-6)
    ln -sfnv "../../../../$symlink_target_prefix/dists/$repo_version" "$symlink_dir/$repo_name/dists/"
    if [ $? -ne 0 ]; then
        echo "$dir_name Link Failed, Use snapshot $latest_subdir_name"
        continue
    fi
    echo "$dir_name Link Success, Use snapshot $latest_subdir_name"
    echo "Removing snapshot"
    for subdir in "${dir}"/*; do
        subdir_name=$(basename $subdir)
        if [[ "$subdir_name" = "$latest_subdir_name" ]]; then
            continue
        fi
        /usr/bin/proxmox-offline-mirror mirror snapshot remove $dir_name $subdir_name --config $proxmox_offline_mirror_config || echo "Remove snapshot $dir_name $subdir_name failed"
    done
    echo "Removing snapshot complete"
    echo "Finish Syncing $dir_name"
done

echo
echo "Snapshot garbage collection"
/usr/bin/proxmox-offline-mirror mirror gc --config $proxmox_offline_mirror_config

echo
echo "Start download ISO, CT template"
echo
echo "Start download ISO, CT template at $(date)" >> "$log_path/wget/wget.$(date +%Y%m%d).log"
/usr/bin/wget -N --mirror --convert-links --show-progress --recursive --no-host-directories -e robots=off -R "index.html*" -X '/temp,/debian' -P $mirror_base_path/packages/ http://download.proxmox.com/ >> "$log_path/wget/wget.$(date +%Y%m%d).log" 2>&1 && echo "Download ISO, CT template Success" || echo "Download ISO, CT template Failed, Watch wget log for more detail"
echo
echo "Finish Syncing Proxmox Mirror"
```

```cron
0 */2 * * * sleep "$(( RANDOM \% 600 ))" && bash /opt/mirrorscript/proxmox_syncrepo.sh >> "/path/to/mirror/proxmox/logs/proxmox-mirror.$(date +\%Y\%m\%d).log"  2>&1
0 2 * * * /usr/bin/find /path/to/mirror/proxmox/logs -type f -mtime +30 -delete
```



## 參考

[^1]: [Command Syntax — Proxmox Offline Mirror 0.6.7 documentation](https://pom.proxmox.com/command-syntax.html)
