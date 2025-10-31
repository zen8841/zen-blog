---
title: FreeIPA 設定與客戶端安裝過程
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - linux
  - server
  - FreeIPA
  - DNS
excerpt: FreeIPA 伺服器端的架設與細部設定及客戶端的安裝與連線
date: 2024-12-23 01:40:00
updated: 2024-12-23 01:40:00
index_img:
---


# 前言

社團原本使用的帳號系統是搭在一個黑群輝虛擬機上的 LDAP 服務，只是之前機房搬遷後虛擬機的黑群輝直接無法開機了，後來是靠其他位置的 LDAP 備份恢復。所以就不太想把這個重要的服務放在黑群輝上，才開始了帳號系統的遷移， FreeIPA 提供了類似於 Linux 中 AD 的功能；而這篇文章主要記錄 FreeIPA 的安裝過程和客戶端該如何設定。

# 服務端安裝

在服務端系統的選擇上我使用 Fedora，主要是 FreeIPA 在 Fedora 上有最新的版本，支援也比較全面，畢竟是 RedHat 支持的專案。

首先需要設定 FreeIPA Server 使用的 hostname 與 domain，例如`freeipa.example.com`，網域為`example.com`，將`freeipa.example.com`寫入`/etc/hostname`，並在`/etc/hosts`中將`freeipa.example.com`解析到自己的 IP，假設 Server 的 IP 為10.0.0.1則加上最後一行來進行解析

```hosts
# Loopback entries; do not change.
# For historical reasons, localhost precedes localhost.localdomain:
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
# See hosts(5) for proper format and other examples:
# 192.168.1.10 foo.example.org foo
# 192.168.1.13 bar.example.org bar
10.0.0.1        freeipa.example.com freeipa
```

接著安裝 Freeipa，網域名稱按照自己的需求更改，我安裝時同時安裝了 bind9 ，同時安裝 DNS 未來配置 kerberos 會較為方便[^1][^2]，不過如果使用存在的域名，安裝腳本會檢查 NS 記錄有沒有指到機器上的 IP，所以如果沒辦法這樣設定可以使用`.internal`域名，將不會檢查 NS 記錄

```shell
# dnf install freeipa-server freeipa-server-dns
# ipa-server-install --mkhomedir
The log file for this installation can be found in /var/log/ipaserver-install.log
==============================================================================
This program will set up the IPA Server.
Version 4.12.1

This includes:
  * Configure a stand-alone CA (dogtag) for certificate management
  * Configure the NTP client (chronyd)
  * Create and configure an instance of Directory Server
  * Create and configure a Kerberos Key Distribution Center (KDC)
  * Configure Apache (httpd)
  * Configure SID generation
  * Configure the KDC to enable PKINIT

To accept the default shown in brackets, press the Enter key.

Do you want to configure integrated DNS (BIND)? [no]: yes

Enter the fully qualified domain name of the computer
on which you're setting up server software. Using the form
<hostname>.<domainname>
Example: master.example.com


Server host name [freeipa.example.com]:


Warning: skipping DNS resolution of host freeipa.example.com
The domain name has been determined based on the host name.

Please confirm the domain name [example.com]:

The kerberos protocol requires a Realm name to be defined.
This is typically the domain name converted to uppercase.

Please provide a realm name [EXAMPLE.COM]:
Certain directory server operations require an administrative user.
This user is referred to as the Directory Manager and has full access
to the Directory for system management tasks and will be added to the
instance of directory server created for IPA.
The password must be at least 8 characters long.

Directory Manager password:
Password (confirm):

The IPA server requires an administrative user, named 'admin'.
This user is a regular system account used for IPA server administration.

IPA admin password:
Password (confirm):

Checking DNS domain example.com., please wait ...
Invalid IP address fe80::be24:11ff:fe69:8783 for freeipa.example.com: cannot use link-local IP address fe80::be24:11ff:fe69:8783
Do you want to configure DNS forwarders? [yes]:
The following DNS servers are configured in systemd-resolved: 1.1.1.1
Do you want to configure these servers as DNS forwarders? [yes]:
All detected DNS servers were added. You can enter additional addresses now:
Enter an IP address for a DNS forwarder, or press Enter to skip:
DNS forwarders: 1.1.1.1
Checking DNS forwarders, please wait ...
Do you want to search for missing reverse zones? [yes]:
Reverse record for IP address x.x.x.x lready exists
Trust is configured but no NetBIOS domain name found, setting it now.
Enter the NetBIOS name for the IPA domain.
Only up to 15 uppercase ASCII letters, digits and dashes are allowed.
Example: EXAMPLE.


NetBIOS domain name [EXAMPLE]:

Do you want to configure chrony with NTP server or pool address? [no]:

The IPA Master Server will be configured with:
Hostname:       freeipa.example.com
IP address(es): x.x.x.x
Domain name:    example.com
Realm name:     EXAMPLE.COM

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=EXAMPLE.COM
Subject base: O=EXAMPLE.COM
Chaining:     self-signed

BIND DNS server will be configured to serve IPA domain with:
Forwarders:       1.1.1.1
Forward policy:   only
Reverse zone(s):  No reverse zone

Continue to configure the system with these values? [no]: yes
The following operations may take some minutes to complete.
Please wait until the prompt is returned.

Adding [x.x.x.x freeipa.example.com] to your /etc/hosts file
Disabled p11-kit-proxy
Synchronizing time
No SRV records of NTP servers found and no NTP server or pool address was provided.
Using default chrony configuration.
Attempting to sync time with chronyc.
Time synchronization was successful.
Configuring directory server (dirsrv). Estimated time: 30 seconds
  [1/42]: creating directory server instance
Validate installation settings ...
Create file system structures ...
Perform SELinux labeling ...
Create database backend: dc=example,dc=com ...
Perform post-installation tasks ...
  [2/42]: adding default schema
  [3/42]: enabling memberof plugin
  [4/42]: enabling winsync plugin
  [5/42]: configure password logging
  [6/42]: configuring replication version plugin
  [7/42]: enabling IPA enrollment plugin
  [8/42]: configuring uniqueness plugin
  [9/42]: configuring uuid plugin
  [10/42]: configuring modrdn plugin
  [11/42]: configuring DNS plugin
  [12/42]: enabling entryUSN plugin
  [13/42]: configuring lockout plugin
  [14/42]: configuring graceperiod plugin
  [15/42]: configuring topology plugin
  [16/42]: creating indices
  [17/42]: enabling referential integrity plugin
  [18/42]: configuring certmap.conf
  [19/42]: configure new location for managed entries
  [20/42]: configure dirsrv ccache and keytab
  [21/42]: enabling SASL mapping fallback
  [22/42]: restarting directory server
  [23/42]: adding sasl mappings to the directory
  [24/42]: adding default layout
  [25/42]: adding delegation layout
  [26/42]: creating container for managed entries
  [27/42]: configuring user private groups
  [28/42]: configuring netgroups from hostgroups
  [29/42]: creating default Sudo bind user
  [30/42]: creating default Auto Member layout
  [31/42]: adding range check plugin
  [32/42]: creating default HBAC rule allow_all
  [33/42]: adding entries for topology management
  [34/42]: initializing group membership
  [35/42]: adding master entry
  [36/42]: initializing domain level
  [37/42]: configuring Posix uid/gid generation
  [38/42]: adding replication acis
  [39/42]: activating sidgen plugin
  [40/42]: activating extdom plugin
  [41/42]: configuring directory to start on boot
  [42/42]: restarting directory server
Done configuring directory server (dirsrv).
Configuring Kerberos KDC (krb5kdc)
  [1/11]: adding kerberos container to the directory
  [2/11]: configuring KDC
  [3/11]: initialize kerberos container
  [4/11]: adding default ACIs
  [5/11]: creating a keytab for the directory
  [6/11]: creating a keytab for the machine
  [7/11]: adding the password extension to the directory
  [8/11]: creating anonymous principal
  [9/11]: starting the KDC
  [10/11]: configuring KDC to start on boot
  [11/11]: enable PAC ticket signature support
Done configuring Kerberos KDC (krb5kdc).
Configuring kadmin
  [1/2]: starting kadmin
  [2/2]: configuring kadmin to start on boot
Done configuring kadmin.
Configuring ipa-custodia
  [1/5]: Making sure custodia container exists
  [2/5]: Generating ipa-custodia config file
  [3/5]: Generating ipa-custodia keys
  [4/5]: starting ipa-custodia
  [5/5]: configuring ipa-custodia to start on boot
Done configuring ipa-custodia.
Configuring certificate server (pki-tomcatd). Estimated time: 3 minutes
  [1/32]: configuring certificate server instance
  [2/32]: stopping certificate server instance to update CS.cfg
  [3/32]: backing up CS.cfg
  [4/32]: Add ipa-pki-wait-running
  [5/32]: secure AJP connector
  [6/32]: reindex attributes
  [7/32]: exporting Dogtag certificate store pin
  [8/32]: disabling nonces
  [9/32]: set up CRL publishing
  [10/32]: enable PKIX certificate path discovery and validation
  [11/32]: authorizing RA to modify profiles
  [12/32]: authorizing RA to manage lightweight CAs
  [13/32]: Ensure lightweight CAs container exists
  [14/32]: Enable lightweight CA monitor
  [15/32]: Ensuring backward compatibility
  [16/32]: startin  [17/32]: configure certmonger for renewals
  [18/32]: requesting RA certificate from CA
  [19/32]: publishing the CA certificate
  [20/32]: adding RA agent as a trusted user
  [21/32]: configure certificate renewals
  [22/32]: Configure HTTP to proxy connections
  [23/32]: updating IPA configuration
  [24/32]: enabling CA instance
  [25/32]: importing IPA certificate profiles
  [26/32]: migrating certificate profiles to LDAP
  [27/32]: adding default CA ACL
  [28/32]: adding 'ipa' CA entry
  [29/32]: Recording random serial number state
  [30/32]: Recording HSM configuration state
  [31/32]: configuring certmonger renewal for lightweight CAs
  [32/32]: deploying ACME service
Done configuring certificate server (pki-tomcatd).
Configuring directory server (dirsrv)
  [1/3]: configuring TLS for DS instance
  [2/3]: adding CA certificate entry
  [3/3]: restarting directory server
Done configuring directory server (dirsrv).
Configuring ipa-otpd
  [1/2]: starting ipa-otpd
  [2/2]: configuring ipa-otpd to start on boot
Done configuring ipa-otpd.
Configuring the web interface (httpd)
  [1/22]: stopping httpd
  [2/22]: backing up ssl.conf
  [3/22]: disabling nss.conf
  [4/22]: configuring mod_ssl certificate paths
  [5/22]: setting mod_ssl protocol list
  [6/22]: configuring mod_ssl log directory
  [7/22]: disabling mod_ssl OCSP
  [8/22]: adding URL rewriting rules
  [9/22]: configuring httpd
  [10/22]: setting up httpd keytab
  [11/22]: configuring Gssproxy
  [12/22]: setting up ssl
  [13/22]: configure certmonger for renewals
  [14/22]: publish CA cert
  [15/22]: clean up any existing httpd ccaches
  [16/22]: enable ccache sweep
  [17/22]: configuring SELinux for httpd
  [18/22]: create KDC proxy config
  [19/22]: enable KDC proxy
  [20/22]: starting httpd
  [21/22]: configuring httpd to start on boot
  [22/22]: enabling oddjobd
Done configuring the web interface (httpd).
Configuring Kerberos KDC (krb5kdc)
  [1/1]: installing X509 Certificate for PKINIT
Done configuring Kerberos KDC (krb5kdc).
Applying LDAP updates
Upgrading IPA:. Estimated time: 1 minute 30 seconds
  [1/10]: stopping directory server
  [2/10]: saving configuration
  [3/10]: disabling listeners
  [4/10]: enabling DS global lock
  [5/10]: disabling Schema Compat
  [6/10]: starting directory server
  [7/10]: upgrading server
  [8/10]: stopping directory server
  [9/10]: restoring configuration
  [10/10]: starting directory server
Done.
Restarting the KDC
dnssec-validation yes
Configuring DNS (named)
  [1/12]: generating rndc key file
  [2/12]: adding DNS container
  [3/12]: setting up our zone
  [4/12]: setting up our own record
  [5/12]: setting up records for other masters
  [6/12]: adding NS record to the zones
  [7/12]: setting up kerberos principal
  [8/12]: setting up LDAPI autobind
  [9/12]: setting up named.conf
created new /etc/named.conf
created named user config '/etc/named/ipa-ext.conf'
created named user config '/etc/named/ipa-options-ext.conf'
created named user config '/etc/named/ipa-logging-ext.conf'
  [10/12]: setting up server configuration
  [11/12]: configuring named to start on boot
  [12/12]: changing resolv.conf to point to ourselves
Done configuring DNS (named).
Restarting the web server to pick up resolv.conf changes
Configuring DNS key synchronization service (ipa-dnskeysyncd)
  [1/7]: checking status
  [2/7]: setting up bind-dyndb-ldap working directory
  [3/7]: setting up kerberos principal
  [4/7]: setting up SoftHSM
  [5/7]: adding DNSSEC containers
  [6/7]: creating replica keys
  [7/7]: configuring ipa-dnskeysyncd to start on boot
Done configuring DNS key synchronization service (ipa-dnskeysyncd).
Restarting ipa-dnskeysyncd
Restarting named
Updating DNS system records
Configuring SID generation
  [1/8]: adding RID bases
  [2/8]: creating samba domain object
  [3/8]: adding admin(group) SIDs
  [4/8]: updating Kerberos config
'dns_lookup_kdc' already set to 'true', nothing to do.
  [5/8]: activating sidgen task
  [6/8]: restarting Directory Server to take MS PAC and LDAP plugins changes into account
  [7/8]: adding fallback group
  [8/8]: adding SIDs to existing users and groups
This step may take considerable amount of time, please wait..
Done.
Configuring client side components
This program will set up IPA client.
Version 4.12.1

Using existing certificate '/etc/ipa/ca.crt'.
Client hostname: freeipa.example.com
Realm: EXAMPLE.COM
DNS Domain: example.com
IPA Server: freeipa.example.com
BaseDN: dc=example,dc=com

Configured /etc/sssd/sssd.conf
Systemwide CA database updated.
Adding SSH public key from /etc/ssh/ssh_host_ed25519_key.pub
Adding SSH public key from /etc/ssh/ssh_host_ecdsa_key.pub
Adding SSH public key from /etc/ssh/ssh_host_rsa_key.pub
SSSD enabled
Configured /etc/openldap/ldap.conf
Configured /etc/ssh/ssh_config
Configured /etc/ssh/sshd_config.d/04-ipa.conf
Configuring example.com as NIS domain.
Client configuration complete.
The ipa-client-install command was successful

==============================================================================
Setup complete

Next steps:
        1. You must make sure these network ports are open:
                TCP Ports:
                  * 80, 443: HTTP/HTTPS
                  * 389, 636: LDAP/LDAPS
                  * 88, 464: kerberos
                  * 53: bind
                UDP Ports:
                  * 88, 464: kerberos
                  * 53: bind
                  * 123: ntp

        2. You can now obtain a kerberos ticket using the command: 'kinit admin'
           This ticket will allow you to use the IPA tools (e.g., ipa user-add)
           and the web user interface.

Be sure to back up the CA certificates stored in /root/cacert.p12
These files are required to create replicas. The password for these
files is the Directory Manager password
The ipa-server-install command was successful
```

Fedora 預設有開啟防火牆，所以需要開啟對應的端口，`freeipa-4`服務已經包括所有 FreeIPA 使用的端口了，如果前面沒有設定 dns 也可以不用開啟 dns 服務

```shell
# firewall-cmd --add-service=freeipa-4
# firewall-cmd --add-service=freeipa-4 --permanent
# firewall-cmd --add-service=dns
# firewall-cmd --add-service=dns --permanent
```

## DNS

若要使用 FreeIPA 的 DNS 作為整個網路的 DNS 則需要允許讓網路內的客戶端可以進行遞回查詢(recursion)

編輯`/etc/named/ipa-ext.conf`，加入這段來定義允許的客戶端，我設定為允許所有的 Private IP，視情況可以進行收縮

```conf
acl "trusted_network" {
    localnets;
    localhost;
    192.168.0.0/16;
    172.16.0.0/12;
    10.0.0.0/8;
};
```

接著編輯`/etc/named/ipa-options-ext.conf`來允許定義的客戶端查詢

```conf
allow-recursion { trusted_network; };
allow-query-cache { trusted_network; };
```

最後重新載入`named`即可

```shell
# systemctl reload named
```

## 拒絕匿名查詢

FreeIPA 安裝完後，它的 LDAP 預設是可以被沒有驗證的用戶查詢到部分資訊，可以將其設定為`rootdse`，而非`off`拒絕所有匿名查詢，因為當客戶端要加入網域時，需要知道`rootdse`的資訊[^3]

首先編輯更改用的 ldif

```ldif
dn: cn=config
changetype: modify
replace: nsslapd-allow-anonymous-access
nsslapd-allow-anonymous-access: rootdse
```

接著將使用 ldif 檔案修改 LDAP Server 的設定， ldif 的檔名為`disable_anon_bind.ldif`

```shell
# ldapmodify -x -D "cn=Directory Manager" -W -H ldap:// -ZZ -f disable_anon_bind.ldif
```

## 一些自定義的設定

在使用`ipa`指令前需要先取得 kerberos 的 ticket，輸入設定的 admin 密碼，即可獲得

```shell
# kinit admin
```

使用`config-mod`可以修改 FreeIPA 預設的一些設定

```shell
# ipa config-mod --defaultshell=/bin/bash
# ipa config-mod --homedirectory=/rhome
```

使用`user-add`與`group-add-member`來新增 FreeIPA 用戶

```shell
# echo password | ipa user-add someone --password --first=some --last=one --email=someone1@example.com --email=someone2@example.com --gecos=someone
# ipa group-add-member somegroup --users=someone
```

---

設定完成後就可以使用瀏覽器進入 FreeIPA 網頁管理介面

![](login.png)

# 客戶端

## 使用 LDAP 驗證

如果是需要配置讓 Linux 可以使用 FreeIPA 中的帳號進行登入推薦使用底下的方式，而不是直接透過 LDAP 驗證，使用 FreeIPA-Client 可以有更細化的權限控制

要讓應用程式使用 FreeIPA 的帳號驗證，需要建立應用程式使用的 bind user 來對 LDAP 查詢，在 FreeIPA 中可以建立 System Accounts 專門用作 bind user[^4]

可以使用 ldif 來增加 System Accounts，但我更推薦使用 [freeipa-sam](https://github.com/noahbliss/freeipa-sam)[^5]的腳本來新增

有了 bind user 後就可以按自己的 需求及應用程式設定 LDAP 的方式填入設定，即能完成 LDAP 的設定

以下為一些 FreeIPA 的 LDAP 中的資料結構，在設定時可以參考

- base dn: dc=example,dc=com

- user location: uid=someone,cn=users,cn=accounts,dc=example,dc=com
- group location: cn=group1,cn=groups,cn=accounts,dc=example,dc=com

- user classes: person
- user name attribute: uid
- email attribute: mail
- group classes: posixGroup
- group name attribute: cn
- user filter: memberOf=cn=group1,cn=groups,cn=accounts,dc=example,dc=com
- group filter: (|(cn=group1)(cn=group2))

## 使用 FreeIPA-Client 驗證

使用 FreeIPA-Client 來讓系統可以使用 FreeIPA 中的帳號來進行登入的話，可以使用 FreeIPA 中的 HBAC Rules(Hardware Based Access Control) 來控制只有哪些用戶/群組可以登入哪些 Host 或是哪些 Host 群組，並且控制能管理的 service 有哪些，並且可以配合 FreeIPA 中的 Sudo Rules 來限定哪些用戶/群組可以在哪些 Host 或是哪些 Host 群組中使用 sudo 中的哪些指令

首先將 Client 的 DNS 設為 FreeIPA 的 Server，編輯`/etc/resolv.conf`

```conf
search example.com
nameserver 10.0.0.1
```

接著安裝 FreeIPA-Client，示範的 client 為 debian

```shell
# apt install freeipa-client
# ipa-client-install --mkhomedir
This program will set up IPA client.
Version 4.9.11

WARNING: conflicting time&date synchronization service 'ntp' will be disabled in favor of chronyd

Discovery was successful!
Do you want to configure chrony with NTP server or pool address? [no]:
Client hostname: debian.client.example.com
Realm: EXAMPLE.COM
DNS Domain: example.com
IPA Server: freeipa.example.com
BaseDN: dc=example,dc=com

Continue to configure the system with these values? [no]: yes
Synchronizing time
No SRV records of NTP servers found and no NTP server or pool address was provided.
Using default chrony configuration.
Attempting to sync time with chronyc.
Time synchronization was successful.
User authorized to enroll computers: admin
Password for admin@EXAMPLE.COM:
Successfully retrieved CA cert
    Subject:     CN=Certificate Authority,O=EXAMPLE.COM
    Issuer:      CN=Certificate Authority,O=EXAMPLE.COM
    Valid From:  XXXX-XX-XX XX:XX:XX
    Valid Until: XXXX-XX-XX XX:XX:XX

Enrolled in IPA realm EXAMPLE.COM
Created /etc/ipa/default.conf
Configured /etc/sssd/sssd.conf
Systemwide CA database updated.
Hostname (debian.client.example.com) does not have A/AAAA record.
Missing reverse record(s) for address(es): 10.1.0.100.
Adding SSH public key from /etc/ssh/ssh_host_ecdsa_key.pub
Adding SSH public key from /etc/ssh/ssh_host_ed25519_key.pub
Adding SSH public key from /etc/ssh/ssh_host_rsa_key.pub
SSSD enabled
/etc/ldap/ldap.conf does not exist.
Failed to configure /etc/openldap/ldap.conf
Configured /etc/ssh/ssh_config
Configured /etc/ssh/sshd_config.d/04-ipa.conf
Configuring example.com as NIS domain.
Configured /etc/krb5.conf for IPA realm EXAMPLE.COM
Client configuration complete.
The ipa-client-install command was successful
```

### 權限控制

- 在 Client 加入 FreeIPA 後可以在 Identity>Hosts 中看到，並且可以在 Identity>Group>Host Group 裡將 Hosts 集合成 Group，方便權限的管理

- 如果要開啟管理用戶能否登入特定主機，需要在 Policy>Host-Based Access Control>HBAC Rules 中將`allow_all`的規則 disable，不然所有用戶都可登入所有主機

- 可以在 Policy>Host-Based Access Control>HBAC Rules 新增規則來設定用戶/群組可以登入哪些主機，並管理哪些服務

- 在 Policy>Host-Based Access Control>HBAC Services 中可以設定服務的種類

- 最後可以在 Policy>Host-Based Access Control>HBAC Test 中測試設定是否正確

- 如果用戶要再 Client 中使用 sudo 需要在 Policy>Sudo>Sudo Rules 中新增規則

- 在設定完成後如果在 Client 沒有生效，可以等待5~10分鐘來讓快取失效重新存取伺服器的內容

## 參考

[^1]: [DNS — FreeIPA  documentation](https://www.freeipa.org/page/DNS)
[^2]: [Deployment_Recommendations — FreeIPA  documentation](https://www.freeipa.org/page/Deployment_Recommendations)
[^3]: [在 Oracle Linux 上安裝 FreeIPA 伺服器](https://docs.oracle.com/zh-tw/learn/ol-freeipa/index.html#disable-anonymous-binds)
[^4]: [LDAP — FreeIPA  documentation](https://www.freeipa.org/page/HowTo/LDAP)
[^5]: [noahbliss/freeipa-sam: System Account Manager for FreeIPA](https://github.com/noahbliss/freeipa-sam)
[^6]: [Quick_Start_Guide — FreeIPA  documentation](https://www.freeipa.org/page/Quick_Start_Guide)
[^7]: [FreeIPA workshop — FreeIPA 4.11-dev documentation](https://freeipa.readthedocs.io/en/latest/workshop.html)
[^8]: [鳥哥私房菜 - 第十一章、使用 LDAP 統一管理帳號](https://linux.vbird.org/linux_server/rocky9/0240ldap.php)
[^9]: [企業人員的統一管理-FreeIPA學習筆記](https://kentyeh.blogspot.com/2016/10/freeipa.html)
[^10]: [Using FreeIPA for User Authentication  · Annvix](https://annvix.com/using_freeipa_for_user_authentication)