---
title: FreeIPA 設定檔導致的 SSH 錯誤
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
  - 疑難雜症
tags:
  - FreeIPA
  - linux
  - server
  - ssh
excerpt: 舊版 FreeIPA 產生的設定檔並未隨著更新改變導致的 SSH 錯誤
date: 2025-08-29 01:01:24
updated: 2025-08-29 01:01:24
index_img:
banner_img:
---


# 前言

事情要說回到前兩週更新 Proxmox 集群時，在更新完成兩個節點後，有使用者回報發生無法打開虛擬機的錯誤，仔細查看後發現是 SSH 的錯誤，在其他節點的 Web UI 開啟另一個節點上虛擬機的 Console 會發生錯誤，不過當時猜測可能是因為集群沒有完整更新完成，部分節點仍停留在 PVE 8，因此暫時忽略。

在徹底更新完成整個集群後，又發現了有關 Migrate 的錯誤，遷移虛擬機時通通發生以下錯誤。

```shell
******************************************************************************
Your system is configured to use the obsolete tool sss_ssh_knownhostsproxy.
Please read the sss_ssh_knownhosts(1) man page to learn about its replacement.
******************************************************************************

Connection closed by UNKNOWN port 65535

TASK ERROR: command '/usr/bin/ssh -e none -o 'BatchMode=yes' -o 'HostKeyAlias=xxx' -o 'UserKnownHostsFile=/etc/pve/nodes/xxx/ssh_known_hosts' -o 'GlobalKnownHostsFile=none' root@10.x.x.x pvecm mtunnel -migration_network 10.x.x.x/24 -get_migration_ip' failed: exit code 255
```

# 問題

其實這個錯誤相當好解決。只要把上面兩排星號間的錯誤訊息在丟到 Google，就能立刻找到兩篇文章，一篇是對 [SSSD 的 Bug report](https://bugzilla.redhat.com/show_bug.cgi?id=2305856)[^1]，一篇是[抱怨 Debian 上的 FreeIPA 會導致更新後 SSH 出錯](https://bgstack15.ddns.net/blog/posts/2025/02/04/freeipa-on-devuan-and-sss-ssh-knownhosts-proxy/)[^2]，這兩篇文章都提到了是`sss_ssh_knownhostsproxy`導致的問題。

在 SSSD 更新後(Proxmox/Debian 大版本更新)，`sss_ssh_knownhostsproxy`這個指令被廢棄，轉向使用新的指令`sss_ssh_knownhosts`，但舊的指令仍然存在，只會回傳錯誤。

```error
******************************************************************************
Your system is configured to use the obsolete tool sss_ssh_knownhostsproxy.
Please read the sss_ssh_knownhosts(1) man page to learn about its replacement.
******************************************************************************
```

之所以會使用到`sss_ssh_knownhostsproxy`指令是因為在`ipa-client-install`時會建立設定檔`/etc/ssh/sshd_config.d/04-ipa.conf`，其中就包括了透過 SSSD 查詢 ssh knownhost 的設定，參考 [FreeIPA 設定與客戶端安裝過程 - 使用 FreeIPA-Client 驗證](/freeipa-install-and-client-setup/#%E4%BD%BF%E7%94%A8-FreeIPA-Client-%E9%A9%97%E8%AD%89)。

```04-ipa.conf
# IPA-related configuration changes to ssh_config
#
PubkeyAuthentication yes
# disabled by ipa-client update
# GlobalKnownHostsFile /var/lib/sss/pubconf/known_hosts
#VerifyHostKeyDNS yes

# assumes that if a user does not have shell (/sbin/nologin),
# this will return nonzero exit code and proxy command will be ignored
Match exec true
        ProxyCommand /usr/bin/sss_ssh_knownhostsproxy -p %p %h
```

# 解決

解決方法其實很簡單，就是把`ProxyCommand`的設定修正一下和註解`GlobalKnownHostsFile`就好

```shell
# sed -E --in-place=.orig 's/^(GlobalKnownHostsFile 
\/var\/lib\/sss\/pubconf\/known_hosts)$/# disabled by ipa-client 
update\n# \1/' /etc/ssh/sshd_config.d/04-ipa.conf
# sed -E --in-place=.orig 's/(ProxyCommand 
\/usr\/bin\/sss_ssh_knownhostsproxy -p \%p \%h)/# replaced by ipa-client 
update\n    KnownHostsCommand \/usr\/bin\/sss_ssh_knownhosts \%H/' 
/etc/ssh/sshd_config.d/04-ipa.conf
```

修正後的內容如下

```04-ipa.conf
# IPA-related configuration changes to ssh_config
#
PubkeyAuthentication yes
# disabled by ipa-client update
# GlobalKnownHostsFile /var/lib/sss/pubconf/known_hosts
#VerifyHostKeyDNS yes

# assumes that if a user does not have shell (/sbin/nologin),
# this will return nonzero exit code and proxy command will be ignored
Match exec true
        # replaced by ipa-client update
    KnownHostsCommand /usr/bin/sss_ssh_knownhosts %H
```

## 深入

雖然問題很容易解決，不過我很好奇，為什麼更新會導致這種錯誤。

在第二篇文章中，有提到 [#9536(FreeIPA 的 Issue 9536)](https://pagure.io/freeipa/issue/9536)，並且[4.12的 release note](https://www.freeipa.org/release-notes/4-12-0.html)也有提到這個 Issue，而且特別說明在更新版本時會通過一種機制自動升級設定檔。

> Deprecated sss_ssh_knownhostsproxy in favor of sss_ssh_knownhosts. With this update, if /usr/bin/sss_ssh_knownhosts is present, it will be used instead of /usr/bin/sss_ssh_knownhostsproxy. We implemented a mechanism to apply this change when upgrading from older versions, and downgrading from newer versions.

我翻了一下 Issue 的內容，找到了 [a41e5e2](https://pagure.io/freeipa/c/a41e5e2a244f8fa2edfd7db1e821d8b0f3bbd997) 這個 Commit，在這個 Commit 中修改了 spec file，增加了判斷 SSSD 是否升級的邏輯，如果升級了，就把設定檔用`sed`取代成新的樣式，反之取代成舊的樣式。

但問題是， spec file 是 RPM 打包時的設定檔案之一，它只適用於 RPM-based system，而 Proxmox/Debian 是 DEB-based system， DEB 是用 deb control file 來做這種設定，但是打包時卻沒有將這部份的邏輯增加到 deb control 中。

![DEB 的設定結構](deb_control.png)

目前先提了一個 Debian Bug report，看之後打包人員是否會納入修正吧，[\#1111396](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1111396)

## 參考

[^1]: [2305856 – sss_ssh_knownhosts man page does not contain information about replacements](https://bugzilla.redhat.com/show_bug.cgi?id=2305856)
[^2]: [FreeIPA on Devuan and sss ssh knownhosts proxy | Knowledge Base](https://bgstack15.ddns.net/blog/posts/2025/02/04/freeipa-on-devuan-and-sss-ssh-knownhosts-proxy/)
[^3]: [Issue #9536: Client configuration of ssh: Replace sss_ssh_knownhostsproxy with sss_ssh_knownhosts - freeipa - Pagure.io](https://pagure.io/freeipa/issue/9536)
[^4]: [FreeIPA 4.12.0 — FreeIPA  documentation](https://www.freeipa.org/release-notes/4-12-0.html)
[^5]: [Commit - freeipa - a41e5e2a244f8fa2edfd7db1e821d8b0f3bbd997 - Pagure.io](https://pagure.io/freeipa/c/a41e5e2a244f8fa2edfd7db1e821d8b0f3bbd997)
[^6]: [#1111396 - freeipa-client: /etc/ssh/ssh_config.d/04-ipa.conf create before 4.12 make ssh error - Debian Bug report logs](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1111396)