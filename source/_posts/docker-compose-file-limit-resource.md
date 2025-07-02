---
title: 如何在 docker compose file 中限制系統資源的使用
katex: false
mathjax: false
mermaid: false
excerpt: 使用 docker-compose.yml 來限制資源使用
date: 2024-06-07 03:34:01
updated: 2024-06-07 03:34:01
index_img:
categories:
- 教程
tags:
- server
- docker
---

#  前言

這也算是在舉辦 TSCCTF 中整理的一些資訊，因為題目都是放在 docker container 中跑，為了避免題目被打穿佔用資源影響到其他題目的正常運作，所以當時幫出題人員整理了一下在 docker compose  file 中限制系統資源使用的方式。

不過現在 docker 中已經包含了 compose 組件，而 compose 組件吃的格式和`docker-compose`指令吃的格式有點不一樣，所以重新整理了這一篇文章。

# 格式

{% fold @docker-compose version 2、3中限制的方法 %}

這兩種格式已經被棄用[^1]

## version 2

```yaml
version: "2"
services:
  service_name:
    image: image_name
    
    ## 若沒有寫入的需求請一律使用read_only，volume也請使用ro掛載
    read_only: true
    
    ## 若有寫入需求請註解掉read_only，改用以下兩行限制rootfs size
    ## 此選項只支援部分storage driver，在overlay2中只支援xfs
    ## https://docs.docker.com/reference/cli/docker/container/run/#storage-opt
    # storage_opt:
      # size: 1G
      
    ## 可使用的cpu數
    cpus: 1
    
    ## 優先級，可不設定
    ## 可以理解為在資源不足時，容器搶資源的能力，按照cpu_share的比例來分資源，預設為1024
    # cpu_shares: 1024
    
    ## 限制memory大小，視個別情況決定限制的大小
    mem_limit: 1G
    
    ## 資源不足時啟用的限制，可不設定
    # mem_reservation: 128M
```



## version3

```yaml
## 請使用3.7以上
version: "3.7"
services:
  service_name:
    image: image_name
    
    ## 若沒有寫入的需求請一律使用read_only，volume也請使用ro掛載
    read_only: true
    
    ## 若有寫入需求請註解掉read_only，改用以下兩行限制rootfs size
    ## 此選項只支援部分storage driver，在overlay2中只支援xfs
    ## https://docs.docker.com/reference/cli/docker/container/run/#storage-opt
    # storage_opt:
      # size: 1G
      
    deploy:
      resources:
        limits:
          ## 可使用的cpu數，可以設定為小數
          cpus: '1'
          ## 限制memory大小，視個別情況決定限制的大小
          memory: 1G

        ## 資源不足時啟用的限制，可不設定
        # reservations:
          # cpus: '0.5'
          # memory: 128M
```

docker-compose v3下必須在`docker-compose`後加上參數啟動

`docker-compose --compatibility up -d`

其實也可以用`docker stack deploy --compose-file docker-compose.yml stack_name`來 deploy，但是必須先初始化 docker stack

{% endfold %}

## compose 組件

這個版本的 compose 格式應該會是現行的版本，這部份參考自 docker 官網的文檔[^2]

```yaml
services:
  service_name:
    image: image_name
    
    ## 若沒有寫入的需求請一律使用read_only，volume也請使用ro掛載
    read_only: true
    
    ## 若有寫入需求請註解掉read_only，改用以下兩行限制rootfs size
    #storage_opt:
      #size: 1G
      
    ## 可使用的cpu數(還有很多其他的限制種類，可以在參考中查看)
    cpus: 1
    
    ## 優先級，可不設定
    #cpu_shares: 1024
    
    ## 限制memory大小，視個別情況決定限制的大小
    mem_limit: 1G
    
    ## 資源不足時啟用的限制，可不設定
    #mem_reservation: 128M
```

這個版本的格式不再需要指定`version`，使用`docker compose -f compose.yml up -d`啟動(如果在`compose.yml`的目錄可以不加`-f`)

## 參考

[^1]: [Legacy versions | Docker Docs](https://docs.docker.com/compose/compose-file/legacy-versions)
[^2]: [Services top-level elements | Docker Docs](https://docs.docker.com/compose/compose-file/05-services)
[^3]: [Setting Memory And CPU Limits In Docker | Baeldung on Ops](https://www.baeldung.com/ops/docker-memory-limit)
[^4]: [在 Docker Compose file 3 下限制 CPU 與 Memory - Yowko's Notes](https://blog.yowko.com/docker-compose-3-cpu-memory-limit/)
[^5]: [How to specify Memory & CPU limit in docker compose version 3 - Stack Overflow](https://stackoverflow.com/questions/42345235/how-to-specify-memory-cpu-limit-in-docker-compose-version-3)
[^6]: [How to limit IO speed in docker and share file with system in the same time? - Stack Overflow](https://stackoverflow.com/questions/36145817/how-to-limit-io-speed-in-docker-and-share-file-with-system-in-the-same-time)
[^7]: [Docker, mount volumes as readonly - Stack Overflow](https://stackoverflow.com/questions/19158810/docker-mount-volumes-as-readonly)