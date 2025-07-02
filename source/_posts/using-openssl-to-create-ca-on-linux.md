---
title: 使用 openssl 在 Linux 上建立 CA
katex: false
mathjax: false
mermaid: false
excerpt: 如何在 Linux 上建立 Certificate Authority
date: 2024-08-26 00:38:38
updated: 2024-08-26 00:38:38
index_img:
categories:
- 教程
tags:
- linux
- server
- network
---

# 前言

最近打算幫內網的域名上個合格的 https 證書，之前一直是用自簽名的，每次都會有安全性警告，因此就有了這篇記錄。

# 設定

在 Linux 上架設 Certificate Authority 有兩種方式，比較簡單的方式是使用 easy-rsa，另一種就是使用 openssl 做設定，不過這兩種方式的底層都還是 openssl。

## easy-rsa

easy-rsa 建立 CA 的方式在[這裡](https://www.digitalocean.com/community/tutorial-collections/how-to-set-up-and-configure-a-certificate-authority-ca)[^1]已經說明的很詳細了，就不再介紹。

## openssl

### 建立 CA

首先要先創建用於存放 CA 的資料夾結構，在這裡用`/root/ca`作為存放 CA 的資料夾做示範，接下來的默認工作目錄都是`/root/ca`

```shell
# mkdir /root/ca
# cd /root/ca
# mkdir private/ certs/ newcerts/ crl/ requests/
# cp /usr/lib/ssl/openssl.cnf .
# chmod 600 openssl.cnf
# touch index.txt
# echo '01' > serial
```

這樣創建後的結構大概是這樣

```shell
# tree
.
├── certs
├── crl
├── index.txt
├── newcerts
├── openssl.cnf
├── private
├── requests
└── serial
```

- 目錄結構說明
  - certs： 存放證書
  - newcerts： 使用 openssl 以 CA 簽署完後會將證書放在這，可以手動移至 certs
  - private： 存放私鑰
  - crl： 存放 crl 的發布證書
  - index.txt： CA 的 datebase
  - serial： 目前證書發行的編號
  - openssl.cnf： 用於配置 CA 的設定

#### 產生 RootCA 私鑰

```shell
# openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -outform PEM -aes256 -out private/cakey.pem
```

- 參數說明
  - -algorithm RSA： 使用 RSA 算法
  - -pkeyopt rsa_keygen_bits:4096： 建立 4096bit 長度的 RSA 私鑰
  - -outform PEM： 使用 PEM 格式而非 DER
  - -aes256： 使用 aes256 加密私鑰
  - -out private/cakey.pem： 存放私鑰在`private/cakey.pem`，不要更改，這是`openssl.cnf`中預設的值

也可不必再此時產生私鑰，可以使用`openssl req`一次產生私鑰和公鑰/CSR，不過我個人習慣先產生私鑰，再按照私鑰產生公鑰。

在這裡我使用`openssl genpkey`生成私鑰，不過在網路上其他的教程有些會使用`openssl genrsa`生成 rsa 私鑰，根據 stackoverflow 的[這篇回答](https://stackoverflow.com/questions/65449771/difference-between-openssl-genrsa-and-openssl-genpkey-algorithm-rsa)[^2]，`genrsa`會生成 PKCS #1 格式的私鑰，`genpkey`則是生成 PKCS #8 格式的私鑰，不過我自己測試目前`genrsa`和`genpkey`都是生成 PKCS #8 格式的私鑰，所以結果是相同的。

#### 產生 RootCA 憑證(公鑰)

接下來就是產生 RootCA 的憑證，產生完成後要複製到客戶端上讓客戶端信任這個 RootCA，   Linux 可以參考[這篇 archwiki](https://wiki.archlinux.org/title/User:Grawity/Adding_a_trusted_CA_certificate)[^3]操作。

---

首先要修改複製過來的`openssl.cnf`，這份設定預設會影響`openssl ca`、`openssl req`的預設參數，想要了解 openssl 的設定可以參考[官網的文檔](https://docs.openssl.org/master/man5/x509v3_config/)[^4]

更正或增加內容到`openssl.cnf`

```conf
[ CA_default ]

dir             = /root/ca
copy_extensions = copy

[ policy_match ]
countryName             = match
stateOrProvinceName     = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ v3_ca ]
basicConstraints = critical,CA:true
keyUsage = critical, keyCertSign, cRLSign
```

- 說明
  - dir： 指向儲存 CA 的目錄，也就是`/root/ca`
  - copy_extensions： 這樣可以在生成 CSR 時就將 extention 包進去
  - [ policy_match ]： 這裡是設定申請證書的 csr 哪些部分要與 CA 的憑證相同才允許簽發憑證，這裡我改成只要求`countryName`相同就允許簽發
  - [ v3_ca ]： 這是一個設定 x509 憑證擴展選項的部分，在前面的`[ req ]`區段可以看到`x509_extensions = v3_ca`，這樣在待會使用`openssl req -x509`產生 RootCA 的憑證時會就會自動包含這些選項，不用手動加上`-extensions v3_ca`
  - keyUsage、basicConstraints： 讓憑證可以簽署其他憑證並且可以簽署 CRL 憑證

---

產生 RootCA 憑證

```shell
# openssl req -x509 -new -config openssl.cnf -key private/cakey.pem -out cacert.pem -set_serial 0 -days 3650
Enter pass phrase for private/cakey.pem:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:TW
State or Province Name (full name) [Some-State]:.Taiwan
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:example company
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:example.com
Email Address []:mail@example.com
```

- 參數說明
  - req -x509 -new： 直接產生憑證，而不是憑證申請要求(CSR)
  - -config openssl.cnf： 使用剛剛設定的的 openssl.cnf 做設定檔
  - -key private/cakey.pem： 剛剛產生的私鑰
  - -out cacert.pem：存放憑證在``cacert.pem`不要更改，這是`openssl.cnf`中預設的值
  - -set_serial 0： 序號0的憑證
  - -days 3650： 憑證有效期限3650天

#### 產生中間憑證

常規的 RootCA 不會直接簽發憑證，而是簽署中間 CA 的中間憑證，再由中間憑證簽發憑證，不過演示環境比較小，就沒做這邊，有興趣的可以參考[這篇文章](https://blog.davy.tw/posts/use-openssl-to-sign-intermediate-ca/)[^5]

#### CRL (可選)

**產生 CRL**

```shell
# # 記錄crl發行的版本
# echo 01 > crlnumber
# # 生成CRL，格式為PEM
# openssl ca -config openssl.cnf -gencrl -keyfile private/cakey.pem -cert cacert.pem -out crl/root.crl.pem
# # 轉換格式為DER，CRL大多為DER格式
# openssl crl -in crl/root.crl.pem -outform DER -out crl/root.crl
# # 建立一個給ca用的網站站點，並複製root.crl到那裡，使用nginx或apache把他呈現出來
# mkdir /var/www/ca
# cp crl/root.crl /var/www/ca/
```

**簽署**

更改 openssl.cnf，讓 CA 在簽證書時加上 CRL 的 entry

編輯`[ usr_cert ]`區段，  CA 的設定`x509_extensions = usr_cert`讓 openssl 在簽署證書時會自動帶上`[ usr_cert ]`裡面設定的擴展

```conf
[ usr_cert ]

basicConstraints=CA:FALSE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
# 上面的都是預設的

# 雖然是http，但是root.crl也是有被RootCA簽署過，可以不被竄改
crlDistributionPoints = URI:http://ca.example.com/root.crl
```

#### OCSP (可選)

生成給 ocsp responder 用的私鑰，不要加上密碼，不然 ocsp responder 每次啟動都要輸入密碼

```shell
# openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -outform PEM -out private/ocsp.key
```

產生 ocsp 證書的申請

```shell
# openssl req -new -key private/ocsp.key -addext 'extendedKeyUsage = critical, OCSPSigning' -out requests/ocsp.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:TW
State or Province Name (full name) [Some-State]:Taiwan
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:example company
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:ca.example.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

由 CA 簽署申請並發下證書，大概會長下面這樣

```shell
# openssl ca -in requests/ocsp.csr -config openssl.cnf -out certs/ocsp.pem
Using configuration from openssl.cnf
Enter pass phrase for /root/ca/private/cakey.pem:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 1 (0x1)
        Validity
            Not Before: xxxxx
            Not After : xxxxx
        Subject:
            countryName               = TW
            stateOrProvinceName       = Taiwan
            organizationName          = example company
            commonName                = ca.example.com
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Subject Key Identifier:
                xxxxx
            X509v3 Authority Key Identifier:
                xxxxx
Certificate is to be certified until xxxxx (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
```

證書會輸出到`certs/ocsp.pem`

啟動 ocsp responder

```shell
# openssl ocsp -index index.txt -CA cacert.pem -rsigner certs/ocsp.pem -rkey private/ocsp.key -port <the port to listen> -text
```

使用 systemd service 把 ocsp responder 變成 daemon

編輯`/etc/systemd/system/ocsp.service`

```service
[Unit]
Description=ocsp Server
StartLimitBurst=100

[Service]
Type=simple
ExecStart=/usr/bin/openssl ocsp -index /root/ca/index.txt -CA /root/ca/cacert.pem -rsigner /root/ca/certs/ocsp.pem -rkey /root/ca/private/ocsp.key -port <the port to listen> -text
Restart=always

[Install]
WantedBy=multi-user.target
```

啟動服務

```shell
systemd enable --now ocsp.service
```

更改 openssl.cnf，讓 CA 在簽證書時加上 ocsp 的 entry

```conf
[ usr_cert ]

basicConstraints=CA:FALSE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
# 上面的都是預設的

# 可以使用nginx代理
authorityInfoAccess = OCSP;URI:http://ca.example.com/ocsp/
```

#### 產生 HTTPS 憑證

產生私鑰和證書申請可以在其他電腦上完成，像是準備要用到證書的機器上，這樣不會暴露私鑰，只需要將 CSR 檔傳到 CA 的機器上就好

**產生私鑰**

```shell
# openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -outform PEM -out example.com.key
```

**產生證書申請**

申請 HTTPS 的證書， DNS 的部分要填正確的域名，可以包括通配符*，在 Common Name 的部分最好也填正確的域名

```shell
# openssl req -new -key example.com.key -addext 'subjectAltName=DNS:example.com,DNS:www.example.com' -out example.com.csr
```

一樣也是輸入對應的資訊，可以參考前面 ocsp 的部分

**CA 簽署**

將 CSR 上傳到 CA 的`requests`資料夾內

```shell
# openssl ca -in requests/example.com.csr -config openssl.cnf -out certs/example.com.pem
```

證書會輸出到`certs/example.com.pem`，將這個憑證複製回申請的機器上就完成了

# 同場加映

## 撤銷證書

```shell
# openssl ca -config openssl.cnf -revoke newcerts/02.pem
```

`-revoke`後面接憑證就 ok 了

## 驗證 CRL

由於 crl 是 DER 格式要加上`-crl_download`選項，如果是 PEM 格式的應該就不用

```shell
# openssl verify -crl_check -crl_download -CAfile cacert.pem verify.pem
```

`cacert.pem`是 CA 的憑證，`verify.pem`是待驗證的憑證

## 驗證 OCSP

```shell
# openssl ocsp -issuer cacert.pem -cert verify.pem -text -url <ocsp URL>
```

`cacert.pem`是 CA 的憑證，`verify.pem`是待驗證的憑證

## 參考

[^1]: [How To Set Up and Configure a Certificate Authority (CA) | DigitalOcean | DigitalOcean](https://www.digitalocean.com/community/tutorial-collections/how-to-set-up-and-configure-a-certificate-authority-ca)
[^2]: [ssl - Difference between `openssl genrsa` and `openssl genpkey -algorithm rsa`? - Stack Overflow](https://stackoverflow.com/questions/65449771/difference-between-openssl-genrsa-and-openssl-genpkey-algorithm-rsa)
[^3]: [User:Grawity/Adding a trusted CA certificate - ArchWiki](https://wiki.archlinux.org/title/User:Grawity/Adding_a_trusted_CA_certificate)
[^4]: [x509v3_config - OpenSSL Documentation](https://docs.openssl.org/master/man5/x509v3_config/)
[^5]: [如何使用 OpenSSL 簽發中介 CA](https://blog.davy.tw/posts/use-openssl-to-sign-intermediate-ca/)