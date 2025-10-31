---
title: DNS zone transfer 的安全性
katex: false
mathjax: false
mermaid: false
categories:
  - 教程
tags:
  - DNS
excerpt: 如何在 BIND9 上進行 DNS zone transfer 的安全設定
date: 2025-10-31 18:01:09
updated: 2025-10-31 18:01:09
index_img:
banner_img:
---


# 前言

前幾天去面試交大資工所時，大部分的問題都答上了，但最後一個問題問有關 DNS zone transfer (master/slave) 如何預防 MITM 攻擊，這我就真的沒操作過了，畢竟之前的 master/slave 不是架在相對安全的網路(成大學網)，就是在安全的封閉網路內(內部網路)，還真沒考慮過這一點，只有做 ACL 控管而已，於是藉此機會加強 DNS 安全性，並順便整理成這篇文章。

# 設定

## 基礎

### ACL 限制

大部分的人應該都會做這部分最基礎的 ACL 限制

```named.conf
acl ALLOW-TRANSFER-Network {
	x.x.x.x;
	y.y.y.y;
	z.z.z.0/24;
};

options {
	...
	allow-transfer (
		a.a.a.a;
		ALLOW-TRANSFER-Network;
	);
};

zone example.com {
	...
	// 也可以設定在zone裡，會覆蓋options內的設定
	// 可在options內設定allow-transfer ( none;};，在zone內再設定權限
	allow-transfer (
		b.b.b.b;
	);
};
```

### 有關 notify 的二三事

`notify`可以在`options` block 和`zone` block 內做設定，可設為`yes`、`explicit`、`no`、`master-only`和`primary-only`五種值。前三種分別代表如下意義，後兩種為較多層環境下的設定

- `notify yes;`： 向網域內所有 NS 記錄的主機和`also-notify`設定的主機發送通知
- `notify explicit;`： 只向`also-notify`設定的主機發送通知
- `notify no;`： 不發送通知

## TSIG

### 生成金鑰

```shell
# tsig-keygen host1-host2 > host1-host2.key
```

在一些文章中可能會看到使用`dnssec-keygen`來生成`HMAC-MD5`的金鑰，類似底下這樣

```shell
# dnssec-keygen -a hmac-md5 -b 128 -n HOST host1-host2
```

但是可能會沒辦法執行，因為`dnssec-keygen`在編譯時沒有編譯這種加密算法進去，而且 MD5 也不太適合使用了，雖然`HMAC-MD5`還被視為安全；以我 debian 和 fedora 上的`dnssec-keygen`版本，都沒有支援 hmac 這類哈希的算法

生成的金鑰寫成設定大概長這樣，用`tsig-keygen`生成的會直接是可以 include 的格式

```named.conf
key "host1-host2" {
        algorithm hmac-sha256;
        secret "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=";
};
```

最好調整一下金鑰檔案的權限

```shell
# chown bind:bind host1-host2.key
# chmod 640 host1-host2.key
```

## 使用金鑰

將金鑰檔案複製到 master/slave 兩台 server 上，並 include 進`named.conf`的第一層，不要放在其他的 block 中。

增加 server block 到`named.conf`的第一層，兩邊都要設定

在 y.y.y.y 上設定到 x.x.x.x 都要用這把 key

```named.conf
server x.x.x.x {
  keys { host1-host2;};
};
```

在 x.x.x.x 上設定到 y.y.y.y 都要用這把 key

```named.conf
server y.y.y.y {
  keys { host1-host2;};
};
```

設定完成後在 x.x.x.x 和 y.y.y.y 中間的所有 DNS 通訊都會使用 TSIG 的金鑰進行簽章，防止有中間人進行竄改，但是 TSIG 不會加密通訊，如果不想要被中間人監聽到 DNS 記錄，最好單獨使用 IPSec 或是其他 VPN 建立安全通道。

## 其他使用方式

不止可以使用 TSIG 對 某台 server 進行驗證，也可使用 key 來做 ACL 限制

像是針對`allow-update`做限制，以下限制了只有 a.a.a.a 才能 update，並且 update 的請求要使用 TSIG 的金鑰進行簽章

```named.conf
allow-update { a.a.a.a; key host1-host2;};
```

## TKEY

TKEY 是一種可以在兩個 server 間自動協商金鑰的方法， Bind9 只實作了一種 TKEY 的模式： Diffie-Hellman key exchange[^1]，但是好像使用 TKEY 這種方式的設定比較少，網路上難找到配置的方式，可能是因為曾經出現過與 TKEY 相關的漏洞，導致攻擊者能夠輕易對 BIND9 伺服器進行 DoS 攻擊；因此在這裡就不提如何設定了。

## 參考

[^1]: [7. Security Configurations — BIND 9 9.18.14 documentation](https://bind9.readthedocs.io/en/v9.18.14/chapter7.html#tkey)
[^2]: [BIND, Master, Slaves and Notify - Server Fault](https://serverfault.com/questions/381920/bind-master-slaves-and-notify)
[^3]: [Chapter 4. Advanced DNS Features](https://nsrc.org/workshops/2008/cctld-ams/Documentation/bind-arm/Bv9ARM.ch04.html#tsig)
[^4]: [Enabling transfer security using TSIG](https://nsrc.org/workshops/2016/nsrc-nicsn-aftld-iroc/documents/lab/lab5-tsig-part1.htm)