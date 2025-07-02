---
title: '如何打開 jnlp 格式的 kvm console'
katex: false
mathjax: false
mermaid: false
excerpt: 分享如何使用 jnlp 格式的 kvm/IPMI
date: 2024-11-13 01:56:50
updated: 2024-11-13 01:56:50
index_img:
categories:
- 教程
tags:
- server
---

# 前言

有些老舊的伺服器內的 IPMI 給的 kvm console 是一個 jnlp 檔，而不是直接給 html5 來用，本文將介紹如何使用 jnlp 檔來連上 IPMI 開啟 kvm 。

# 設定

1. 安裝`icedtea-web`和`jre8`

{% note info %}

也可以使用`jdk8`( `java8`的環境就可 )，之前我使用`jre8-adoptopenjdk`，後來變成`jdk8-temurin`，不過前陣子更新後出[問題](https://github.com/AdoptOpenJDK/IcedTea-Web/issues/915)了[^1]，所以改回`jre8`

{% endnote %}

2. 修改`java.security`( linux 上全域的設定在`/etc/java-jre8/security/java.security` )，找到這三段內容，全部註解掉

```
jdk.certpath.disabledAlgorithms=MD2, MD5, SHA1 jdkCA & usage TLSServer, \
    RSA keySize < 1024, DSA keySize < 1024, EC keySize < 224, \
    SHA1 usage SignedJAR & denyAfter 2019-01-01
```

```
jdk.jar.disabledAlgorithms=MD2, MD5, RSA keySize < 1024, \
      DSA keySize < 1024, SHA1 denyAfter 2019-01-01
```

```
jdk.tls.disabledAlgorithms=SSLv3, TLSv1, TLSv1.1, RC4, DES, MD5withRSA, \
    DH keySize < 1024, EC keySize < 224, 3DES_EDE_CBC, anon, NULL
```

3. 打開 Java Control Panel，修改 security 分頁的設定，將安全等級下降為 high，並把 IPMI 的域名/IP 加進白名單來允許 Self-signed certificate

![](java_security.png)

4. 把`icedtea-web`設成用來開啟 jnlp 檔，並把使用的 java 切成 jre8 即可( 例如在 Arch 上是這樣設定`sudo archlinux-java set java-8-jre/jre` )

# 同場加映

1. 如果使用的是 DELL 的伺服器，而且 IPMI 為 iDRAC6 ，那麼可以直接使用 docker 來開啟 kvm( [DomiStyle/docker-idrac6](https://github.com/DomiStyle/docker-idrac6) )
2. 如果使用的是很舊的 IPMI，有可能連設定的網頁部分都需要 flash ( 例如 Cisco UCS C240 )，不想配置 flash 環境的話也可以直接用 docker 來開( [jchprj/Play-Adobe-Flash-After-EOL](https://github.com/jchprj/Play-Adobe-Flash-After-EOL) )

## 參考

[^1]: [Fatal: Initialization Error (unsigned jars?) · Issue #915 · AdoptOpenJDK/IcedTea-Web](https://github.com/AdoptOpenJDK/IcedTea-Web/issues/915)