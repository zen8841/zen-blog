---
title: 使用 LNMP 環境搭建 WordPress 網站
katex: false
mathjax: false
mermaid: false
excerpt: 記錄使用 nginx 架設 WordPress 的過程
date: 2024-08-05 23:41:48
updated: 2024-08-05 23:41:48
index_img:
categories:
- 教程
tags:
- server
- linux
- Nginx
- wordpress
---

# 前言

這算是之前搭建部落格時考慮過的方案，不過後來還是選擇了 hexo 這種靜態框架，所以就記錄一下查詢資料和架設的歷程。

# 架設

## 安裝

先下載安裝必要的組件

```shell
$ sudo apt install nginx nginx-extras
$ # 我使用的database是mariadb，也可使用mysql或postgresql
$ sudo apt install mariadb-server
$ # 安裝需要的php軟體包，可視情況減少或增加
$ sudo apt install php php-fpm php-cli php-mysql php-mbstring php-xml php-gd php-curl php-ssh2 php-imagick php-json php-tokenizer php-zip php-intl
$ # 下載wordpress
$ wget https://tw.wordpress.org/latest-zh_TW.tar.gz
$ # 解壓縮
$ tar -zxvf latest-zh_TW.tar.gz
$ # 新建網站的根目錄，sitename可以是網站的網址或任何你喜歡的名字，後面按需替換
$ sudo mkdir -p /var/www/sitename
$ sudo mv wordpress/* /var/www/sitename/
$ # 更改權限
$ chown -R www-data:www-data /var/www/sitename
$ chmod -R 755 /var/www/sitename
```

## Database 配置

使用 root 執行`mysql_secure_installation`

```shell
# mysql_secure_installation
```

跟著引導安裝即可，說明很完全

---

執行`mysql`，進入 Database

```sql
# 可以使用其他的名稱來命名Database，這裡以wordpress作為範例，如果改名後面也要更改
MariaDB [(none)]> CREATE DATABASE wordpress;
Query OK, 1 row affected (0.002 sec)

# 新建wpuser的帳戶，密碼為passwd，可以存取wordpress的所有權限，可以用其他用戶名和密碼，待會會使用到
MariaDB [(none)]> GRANT ALL PRIVILEGES ON wordpress.* TO "wpuser"@"localhost" IDENTIFIED BY "passwd";
Query OK, 0 rows affected (0.008 sec)

# 應用
MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.003 sec)

MariaDB [(none)]> exit
Bye
```

## 網頁設定

### php

編輯`/etc/php/8.2/fpm/php.ini`，搜尋這三個設定並更改，如果 php 版本較高，記得更改對應的位置

```ini
cgi.fix_pathinfo=0
upload_max_filesize = 256M
post_max_size = 256M
```

編輯`/etc/php/8.2/fpm/pool.d/www.conf`，這些應該是預設值，確認一樣即可

```conf
listen = /run/php/php8.2-fpm.sock
user = www-data
group = www-data
```

重啟`php-fpm`

```shell
# systemctl enable --now php8.2-fpm
# systemctl restart php8.2-fpm
```

### nginx

編輯`/etc/nginx/nginx.conf`，修改對應的部分

```conf
event{
    worker_connections 1024;
    use epoll;
    multi_accept on;
}
http{
    server_tokens off;
    client_max_body_size 256M;
}
```

編輯`/etc/nginx/fastcgi_params`，在最後加上設定

```conf
fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
```

新增`/etc/nginx/sites-available/wordpress`，編輯 virtual server 設定，這個設定為 http 站點，且設定為 default_server，可依照自己需求進行更改。

```conf
server {
        listen 80 default_server;
        ## Your website name goes here.
        # server_name example.com;
        ## Your only path reference.
        root /var/www/sitename;
        ## This should be in your http block and if it is, it's not needed here.
        index index.php;

        client_max_body_size 512M;

        # log
        access_log /var/log/nginx/wordpress/access.log;
        error_log /var/log/nginx/wordpress/error.log;

        location = /favicon.ico {
                log_not_found off;
                access_log off;
        }

        location = /robots.txt {
                allow all;
                log_not_found off;
                access_log off;
        }

        location / {
                # This is cool because no php is touched for static content.
                # include the "?$args" part so non-default permalinks doesn't break when using query string
                try_files $uri $uri/ /index.php?$args;
        }

        location ~ \.php$ {
                #NOTE: You should have "cgi.fix_pathinfo = 0;" in php.ini
                include fastcgi_params;
                fastcgi_intercept_errors on;
                fastcgi_pass unix:/run/php/php-fpm.sock;
                #The following parameter can be also included in fastcgi_params file
                fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }

        location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
                expires max;
                log_not_found off;
        }
}
```

最後將站點啟用並重啟 nginx

```shell
$ # 創建設定檔中log的放置資料夾
$ sudo mkdir -p /var/log/nginx/wordpress/
$ # 啟用站點
$ sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/wordpress
$ # 刪除預設站點
$ sudo rm /etc/nginx/sites-enabled/default
$ # 測試設定檔是否正確，應該是不會報錯
$ sudo nginx -t
$ # 啟用服務
$ sudo systemctl enable --now nginx.service
$ # 重啟nginx
$ sudo nginx -s reload

```

## WordPress 安裝

在完成前面的設定後使用瀏覽器打開你架設 WordPress 服務器的 IP，應該會看到這個頁面

![](wordpress1.png)

接下來就跟著導引安裝即可

資料庫名稱使用剛剛 Database 設定部分資料庫的名字，帳號和密碼同樣也是按照設定的填

![](wordpress2.png)

如果都沒填錯，那麼應該就會看到這個畫面，執行安裝即可

![](wordpress3.png)

接下來就是 WordPress 站點的設定，同樣也是自己按需填寫即可，這邊填的都是範例資料

![](wordpress4.png)

這樣就完成 WordPress 的安裝了，登入即可開始 Wordpress 網站的設定

![](wordpress5.png)

![](wordpress6.png)

![](wordpress7.png)

## 參考

[^1]: [Install WordPress with Nginx on Ubuntu 18.04 | DigitalOcean](https://www.digitalocean.com/community/tutorials/install-wordpress-nginx-ubuntu)
[^2]: [Creating Database for WordPress – Advanced Administration Handbook | Developer.WordPress.org](https://developer.wordpress.org/advanced-administration/before-install/creating-database/)
[^3]: [Editing wp-config.php – Advanced Administration Handbook | Developer.WordPress.org](https://developer.wordpress.org/advanced-administration/wordpress/wp-config/)
[^4]: [NGINX 搭配 PHP-FPM 配置 WordPress 多站點網站 for CentOS 8 - 腳印網頁資訊設計](https://footmark.com.tw/news/web-design/wordpress/nginx-php-fpm-wordpress-centos8/)
[^5]: [用 Nginx 取代 Apache 吧 – WordPress 實務操作 Round1 – Mr. 沙先生](https://shazi.info/%E7%94%A8-nginx-%E4%BE%86%E5%AE%89%E8%A3%9D-wordpress-%E6%94%B9%E7%94%A8-lnmp-%E5%8F%96%E4%BB%A3-apache-%E5%90%A7/)
[^6]: [[講解] nginx 與 php-fpm 運作介紹與設定 | 辛比記](https://tec.xenby.com/20-nginx-%E8%88%87-php-fpm-%E9%81%8B%E4%BD%9C%E4%BB%8B%E7%B4%B9%E8%88%87%E8%A8%AD%E5%AE%9A%E8%AC%9B%E8%A7%A3)
[^7]: [調整 php-fpm & nginx 參數，讓 Server 可以承受更大的流量 | by 林鼎淵 | Dean Lin | Medium](https://medium.com/dean-lin/%E8%AA%BF%E6%95%B4-php-fpm-nginx-%E5%8F%83%E6%95%B8%E4%BE%86%E5%B0%8D%E6%8A%97%E5%A4%A7%E6%B5%81%E9%87%8F%E6%83%85%E5%A2%83-b465a913ee07)