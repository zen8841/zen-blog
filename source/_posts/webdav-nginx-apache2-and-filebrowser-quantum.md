---
title: WebDav, Nginx, Apache2 與 Filebrowser Quantum
katex: false
mathjax: false
mermaid: false
categories:
  - Tutorial
tags:
  - Linux
  - server
  - docker
excerpt: 搭建 WebDav Server 方案的選擇，與使用 Apache2 搭建 WebDav + Nginx 反向代理的教學
date: 2026-07-08 04:09:41
updated: 2026-07-08 04:09:41
index_img:
banner_img:
---


# 前言

這算是接續上一篇的內容，換到 NiceTab 後還是被 Onetab 丟資料的事嚇怕了，所以打算盡快搭一個 WebDav 來備份資料；其實以前就有搭 WebDav 的需求，不過一直因為沒有急迫感，就拖了很久，這次搭好服務順便把像 Tampormonkey 之類的擴充功能一起備份還有以前的需求一起完成。

# 選型

最一開始是先考慮 Nginx，因為已經有一個正在運行的 Reverse Proxy 了，先說能力方面， Nginx 只提供了`PUT`、`DELETE`、`COPY`、`MOVE`等基礎的 HTTP Method，雖然可以使用 dav-ext-module 來補充 Nginx 本身缺少的 HTTP Method(`PROPFIND`、`OPTIONS`等)，但是這個項目已經8年沒有更新，有一些已知的問題只能去用特定的 fork 解決[^1]，並且也有聽說有時候會遇到不穩定的問題，兼容性不如 httpd(Apache2) 來得好。

第二個問題就比較麻煩了， Nginx 在本機以`www-data`的用戶/組運行，不管你是用什麼帳戶通過 HTTP Basic Authentication，最後寫入 FS 的都會用`www-data`作為 owner:group，這我就比較不能接受了，我的 NAS 上存的檔案都是1000:1000，並且像是 SMB、其他的 Docker 服務都依賴這套權限設定，不太方便更改，我也沒有意願再多開一個 Nginx 的 Service，因此放棄 Nginx。

後來看到了 Filebrowser Quantum，剛好以前也有用 Filebrowser，不過現在 Filebrowser 已經完成了，進入維護模式，主要只處理問題和解決安全問題，恰巧 Filebrowser Quantum 有提供 WebDav 的功能，就打算嘗試看看，雖然體驗還不錯，我打算替換掉我現有的 Filebrowser 服務，不過他的 WebDav 功能不太好， WebDav 功能預設是開啟，只要申請一把 API Key 就可以直接開始使用 WebDav，權限就是這個 User 能看到的所有權限，但是如果想要分派不同權限就只能創建多個 User，然後登入這些 User 在申請 API Key，我覺得這個實在是有點麻煩，不太符合我的哲學，故放棄。

對於為什麼選擇 Filebrowser Quantum 而不是 NextCloud 有兩個原因，其一就是他的數據儲存資料夾不喜歡有其他人去動，他會 Index 檔案儲存到 SQL 才能使用搜尋或縮圖等功能，其二就是他是 php 寫的。

最後我還是選擇了 httpd(Apache2)，雖然他會遇到和 Nginx 相同的權限問題，但是可以透過使用 Docker 並透過限制 user 來解決，且 httpd 對於 WebDav 的支援也是最好的，也是因為這個原因才沒有使用 Dave 或是 rclone 等方式架設。

# 架設

## httpd

先建立 docker compose 和設定檔用的目錄，

```shell
$ mkdir WebDav
$ cd WebDav
$ mnkdir -p {conf/extra,logs,var}
```

從 docker image 裡 dump 出設定檔

```shell
$ docker run --rm httpd cat /usr/local/apache2/conf/httpd.conf > ./conf/httpd.conf
$ docker run --rm httpd cat /usr/local/apache2/conf/extra/httpd-dav.conf > ./conf/extra/httpd-dav.conf
```

開啟 httpd 的 WebDav 功能相當簡單，只要 Enable 模組，並對目錄啟用`Dav On`即可，大部分的設定其實是做權限管理的

編輯`conf/httpd.conf`

```conf
# 把模組取消註解啟用
LoadModule dav_module modules/mod_dav.so
LoadModule dav_fs_module modules/mod_dav_fs.so
LoadModule dav_lock_module modules/mod_dav_lock.so

...
# 設定DocumentRoot到要共享的目錄，待會要用docker map volume進去的位置
# DocumentRoot "/usr/local/apache2/htdocs"
DocumentRoot "/srv"

# 用不到的路徑或功能通通註解
# <Directory "/usr/local/apache2/htdocs">
#     #
#     # Possible values for the Options directive are "None", "All",
#     # or any combination of:
#     #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
#     #
#     # Note that "MultiViews" must be named *explicitly* --- "Options All"
#     # doesn't give it to you.
#     #
#     # The Options directive is both complicated and important.  Please see
#     # http://httpd.apache.org/docs/2.4/mod/core.html#options
#     # for more information.
#     #
#     Options Indexes FollowSymLinks
#
#     #
#     # AllowOverride controls what directives may be placed in .htaccess files.
#     # It can be "All", "None", or any combination of the keywords:
#     #   AllowOverride FileInfo AuthConfig Limit
#     #
#     AllowOverride None
#
#     #
#     # Controls who can get stuff from this server.
#     #
#     Require all granted
# </Directory>

...

# 用不到的路徑或功能通通註解
# <IfModule dir_module>
#     DirectoryIndex index.html
# </IfModule>

...
    # 用不到的路徑或功能通通註解
    # ScriptAlias /cgi-bin/ "/usr/local/apache2/cgi-bin/"

...

# 用不到的路徑或功能通通註解
# <Directory "/usr/local/apache2/cgi-bin">
#     AllowOverride None
#     Options None
#     Require all granted
# </Directory>

...
# 要在<IfModule headers_module>中添加RequestHeader edit Destination ^https http early，關於原因後文會說明
<IfModule headers_module>
    RequestHeader edit Destination ^https http early
</IfModule>

...

# 取消註解
Include conf/extra/httpd-dav.conf
```

接著更改`conf/extra/httpd-dav.conf`，這是 httpd 提供的一套預設設定檔

可以把附帶的`Alias`刪掉，因為已經設定了`DocumentRoot`，不用映射目錄過來了，這份設定可供參考，按需修改即可

我的設定參考了部分 [docker-webdav](https://github.com/BytemarkHosting/docker-webdav) 和[他的改進](https://github.com/mgutt/docker-apachewebdav)的設定

```conf
DavLockDB "/usr/local/apache2/var/DavLock"

# 把Directory改成要共享的目錄
<Directory "/srv">
    # 啟用WebDav
    Dav On

    # 我只開了FollowSymLinks這個Options
    # 關掉了Index，如果想在網頁上看檔案我可以用其他服務
    # 如果要開Index可以看看fancyindex的設定
    # Options Indexes FollowSymLinks
    # IndexOptions Charset=UTF-8
    Options FollowSymLinks
    # 不要檢查有沒有.htaccess，提高性能
    AllowOverride None
    # 不用管有沒有index.html之類的檔案
    DirectoryIndex disabled
    # 不要對client的IP反查DNS，提高性能
    HostnameLookups Off

    # 設定驗證，建議用Basic就好，反正會有https
    # digest只能使用md5儲存，Basic可以用Bcrypt
    # https://httpd.apache.org/docs/current/misc/password_encryptions.html
    AuthType Basic
    AuthName WebDav
    AuthUserFile "/usr/local/apache2/user.passwd"

    # 限定使用者存取
    <RequireAll>
        Require user aaa
    </RequireAll>
</Directory>

# 設定只能有部分目錄的帳號
<Directory "/srv/bbb">
    # 扣掉FollowSymLinks，減少風險
    Options -FollowSymLinks
    # 限定這兩個user都可以存取
    <RequireAny>
        Require user aaa
        Require user bbb
    </RequireAny>
    # 給限定user恢復FollowSymLinks
    <If "%{REMOTE_USER} == 'aaa'">
        Options +FollowSymLinks
    </If>
</Directory>

# 給某些帳號read only權限
<Directory "/srv/ccc">
    # 扣掉FollowSymLinks，減少風險
    Options -FollowSymLinks
    <RequireAny>
        # 限定這兩個user都可以存取
        <RequireAll>
            Require user ccc
            # 限定只有唯獨的操作，只能使用在httpd 2.4+後的語法
            Require method GET POST OPTIONS PROPFIND
        </RequireAll>
        Require user aaa
    </RequireAny>
    # 給限定user恢復FollowSymLinks
    <If "%{REMOTE_USER} == 'aaa'">
        Options +FollowSymLinks
    </If>
</Directory>

# 一些客戶端的兼容性設定
#
# The following directives disable redirects on non-GET requests for
# a directory that does not include the trailing slash.  This fixes a
# problem with several clients that do not appropriately handle
# redirects for folders with DAV methods.
#
BrowserMatch "Microsoft Data Access Internet Publishing Provider" redirect-carefully
BrowserMatch "MS FrontPage" redirect-carefully
BrowserMatch "^WebDrive" redirect-carefully
BrowserMatch "^WebDAVFS/1\.[01234]" redirect-carefully
BrowserMatch "^gnome-vfs/1\.0" redirect-carefully
BrowserMatch "^gvfs/1" redirect-carefully
BrowserMatch "^XML Spy" redirect-carefully
BrowserMatch "^Dreamweaver-WebDAV-SCM1" redirect-carefully
BrowserMatch " Konqueror/4" redirect-carefully
BrowserMatch " Konqueror/5" redirect-carefully
BrowserMatch " dolphin/" redirect-carefully
BrowserMatch "^Microsoft-WebDAV-MiniRedir" redirect-carefully
BrowserMatch "^Jakarta-Commons-VFS" redirect-carefully
```

## 設定密碼

可以使用`htpasswd`設定密碼，如果本機沒有可以直接使用 docker 內的 binary

```shell
$ docker run -it --rm httpd /bin/bash
```

`htpasswd`會用到的參數大概有幾個

- -n: 不使用檔案，直接顯示到 stdout
- -b: 使用 cli 的參數作為密碼，反之則是直接輸入
- -B: 使用 bcrypt
- -c: 建立一個新的密碼檔案

大概可以組合出這麼幾種用法

```shell
$ htpasswd -nB myName >> user.passwd
$ htpasswd -nBb myName mypasswd >> user.passwd
$ htpasswd -Bb -c /path/to/passwd/file myName mypasswd
$ htpasswd -Bb /path/to/passwd/file myName mypasswd
$ htpasswd -B -c /path/to/passwd/file myName
$ htpasswd -B /path/to/passwd/file myName
```

這樣就可以建立帳戶並設定密碼了

## docker compose

基底大概是這樣，如果要設定 IP 可以在此基礎上往上加

```yaml
services:
  webdav:
    image: httpd:latest
    container_name: WebDav
    volumes:
      - ./conf/httpd.conf:/usr/local/apache2/conf/httpd.conf:ro
      - ./conf/extra/httpd-dav.conf:/usr/local/apache2/conf/extra/httpd-dav.conf:ro
      - ./user.passwd:/usr/local/apache2/user.passwd:ro
      - ./logs:/usr/local/apache2/logs
      - ./var:/usr/local/apache2/var
      - /path/to/your/data:/srv
    user: 1000:1000
    restart: unless-stopped
```

## Nginx 反向代理 WebDav

如果有設定 http 到 https 的301轉址需要把301改成308，308就是301的升級版，會保留原有請求的 HTTP Method，而不會改成`GET`，如果不使用308可能會有一些客戶端會有問題

```conf
server {
    .....
    return 308 https://$host$request_uri;
}
```

### 關於 Destination

關於前文提到的`RequestHeader edit Destination ^https http early`，如果沒有做，就要在 Nginx 這邊設定，這項設定是將 Destination Header 內的 https 改成 http 在最早接到請求時，因為 TLS 的部分在外部，內部的 WebDav 只有 http，如果收到`MOVE`的 Destination 是 https 的就會出錯，因此要先改

如果要在 Nginx 這邊做可以參考這個，

```conf
server {
    ......
    location / {
        proxy_pass ....;
        set $dest $http_destination;
        if ($http_destination ~ "^https://(?<myvar>(.+))") {
            set $dest http://$myvar;
        }
        proxy_set_header Destination $dest;
    }
}
```

不過 Nginx 的 if 性能不好，有一句話 "If is evil" 就是形容這個的，建議改成 map

```conf
http{
    map $http_destination $fixed_destination {
      ~^https://(?<remainder>.+)$  http://$remainder;
      default                      $http_destination;
    }
    ......
    server {
        ......
        location / {
            proxy_pass ....;
            proxy_set_header Destination $fixed_destination;
        }
    }
}
```

不過如果後端 Apache2 設定了， Nginx 就不用再設定，我建議是在 Apache2 設定就好

### 其他設定

其他的設定關於 TLS 相關的，就按個人設定習慣設定即可

```conf
upstream webdav {
    server 10.60.11.40:80;
    keepalive 32;
}

server {
    listen ...;
    server_name ...;
    
    tls...
    
    # 關閉限制client上傳大小
    client_max_body_size 0;

    # 關閉buffer，直接傳向後端，減少延遲
    # Disable Buffering
    proxy_buffering off;
    proxy_request_buffering off;
    
    location / {
        proxy_pass http://webdav;
        proxy_http_version 1.1;
        # 和後端複用tcp連線，和upstream的keepalive配合
        proxy_set_header   Connection         "";

        proxy_set_header   Host               $host;
        proxy_set_header   X-Forwarded-Host   $host;
        proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto  $scheme;
        proxy_set_header   X-Forwarded-Port   $server_port;
        proxy_cookie_path  /                  "/; Secure";

        # 限定連接時間
        # Proxy timeouts between successive read/write operations, not the whole request.
        proxy_connect_timeout              300s;
        proxy_send_timeout                 300s;
        proxy_read_timeout                 300s;
    }
}
```

## 參考

[^1]: [Nginx搭建WebDAV服务 | Alliot's blog](https://www.iots.vip/post/nginx-webdav-server)

