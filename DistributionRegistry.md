# Distribution Registry



# Docker Tag 命名
一般來講，同一個docker image會提供多個不同的版本，每個版本會附予不同的tag，以作標識。但以docker image的維護者來講，它的tag通常代表的是自己程式的版本號。不過這個版本號卻存在很多變數，就讓筆者好好地逐一說明。

## 程式的版本號
在沒有Docker的年代，其實所有軟件在發佈時，都會標示版本號，方便使用方明確追蹤問題，自行選擇升級、降級以解決相容性問題。大家要重現問題，也能清清楚地重現。所以docker image的tag，在某程度，都是代表發佈自己的程式版本號。但以前的年代，軟件底層的依賴，例如OS層面的共享程式庫，則不在發佈的管控中，所以過去的程式，在跨電腦安裝時，都會出現缺少某些共享庫的問題。而使用了Docker後，image以內的共享庫的都會在打包的那一刻固定和發佈，就不會有漏的問題。

## 庫更新，怎麼辦
上面說到image可以打包共享庫，但問題是共享庫也會有安全性更新問題，那麼對docker image的維護者來講，它自己的tag又該如何命名？

因為庫的量可大可少，所以一般來說，都不可能完全把各個庫的版本號寫在自己的tag上。退而求其次，就是用"版本號+日期"，庫的細版本號，就存在原始碼當中。[Ubuntu](https://hub.docker.com/_/ubuntu/tags) 就是這樣的例子。

不過"版本號+日期"的命名方式真的方便嗎？每次下遊用戶想更新去最近版本，都要自己找一次最近的日期。這樣對很多用戶來講都不夠方便。所以docker又提供了一個重tag的功能。例如ubuntu:noble，在早些時候指著noble-20240904.1，然後過幾天，又指向更新的noble-20241009。更常見的是latest，每次image都預設會存在，docker也希望大家會定期更新這個tag，讓大家可以更易地找到最新版本。

註: 這跟git tag有所不同，git tag並不預期會變的。當協作者收到tag後，那怕上遊刻意更新tag指針，協作者沒有刪除原tag之前，都不會知道tag更新去了哪裏。

## 我們該如何選
在發佈方和引用方來講，引用時可以明確使用唯一的"版本號+日期"，對穩定性來講是有意義的。不過多多少少，會產生額外的時間成本。發佈方來說，就是多用了一些儲存空間，方便引用方可以隨時找到舊(庫)版本。而引用方，就要手動修改引用號，作為驗收依據，自動更新的難度比較大。

但對於自動更新要求比較大的情況下，可能就是使用latest或者會隨時更新的share tag(共用tag)比較實際。但我們也依然要定一些方式去版本更新記錄，例如：同時使用
- beta
- latest
- archive

每日自動更新beta，只有所有測試都通過時，才把archive指向現在的latest，再把latest指向現在的beta。這樣做的好處是，核心的docker stack檔案改變的機會較少，也可以免除docker swarm做太細緻的權限管理。

# Private Registry 私有影像倉庫
[Offical Doc](https://distribution.github.io/distribution)
如果 server 群沒有互聯網，又或對私隱很有要求，需要自建一個最簡單的 registry ，可以用這個。當然，那台機第一次必需經互聯網。架起後就可以斷網，並由其他 client 提送新的 registry image更新。

## Registry Server 起動方式
最簡單的起動方式，但什麼都不設定。
```bash
docker run -d -p 5000:5000 --name registry registry:3
```

若想要加入 SSL，讓你的 client 不會認為它是不安全的 registry ，最簡易可以寫成 docker compose, 由 `docker compose up -d` 執行。
```yml
# docker-compose.yml
registry:
  restart: always
  image: registry:3
  ports:
    - 5000:5000
  environment:
    REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
    REGISTRY_HTTP_TLS_KEY: /certs/domain.key
  volumes:
    - /path/data:/var/lib/registry
    - /path/certs:/certs
```

上述的 environment 中，有條件的話，還請設定需要登入才能訪問限制。最簡單，可以使用 apache http header 驗證方式。
```diff
# docker-compose.yml
registry:
  restart: always
  image: registry:3
  ports:
    - 5000:5000
  environment:
    REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
    REGISTRY_HTTP_TLS_KEY: /certs/domain.key
+   REGISTRY_AUTH: htpasswd
+   REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
+   REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
  volumes:
    - /path/data:/var/lib/registry
    - /path/certs:/certs
+   - /path/auth:/auth
```

`REGISTRY_AUTH`, `REGISTRY_AUTH_HTPASSWD_PATH`, `REGISTRY_AUTH_HTPASSWD_REALM` 的值照抄就好，然後`/path/auth/htpasswd` 就需要以 htpasswd 的格式提供內容 [apache password_encryptions](https://httpd.apache.org/docs/2.4/misc/password_encryptions.html)。即是以下那個樣子
```
USERNAME_1:BCRYPT_HASH_1
USERNAME_2:BCRYPT_HASH_2
USERNAME_3:BCRYPT_HASH_3
```

## Client 連線方式
一切都設定好後，在 client 端，就可以登入並推送你的 image，(題外話，cli登入的都是以明文的方式存在電腦中，所以不要隨便在公開的地方存入自己的帳號)
```bash
# login
docker login YOUR_DOMAIN:5000
# try re-upload image
docker image tag registry:3 YOUR_DOMAIN:5000/registry:3
docker image push YOUR_DOMAIN:5000/registry:3
```

如果 server 端沒有提供SSL，那麼 client 就只能設定 http 的不安全連線。 [https://distribution.github.io/distribution/about/insecure/](https://distribution.github.io/distribution/about/insecure/)

修改 **client** 端的 /etc/docker/daemon.json (Windows Docker Desktop請經 Gui修改)，然後重啟 **client** 端的 docker
```json
{
  "insecure-registries" : ["YOUR_DOMAIN:5000"]
}
```

## Registry Server 維護 - Garbage collection 垃圾回收
當我們設立了自己的 Registry 倉庫之後，少不免就是要維護硬碟的用量。很多過期的 Image ，沒有需要，那就手動刪除，然後進行 Garbage collection (垃圾回收)。另一種情況，就如前述教學中，大家使用統一版本號，例如 latest ，表面上看似只有一個 tag ，但其實底下可能已經藏有多個不同的版本，也需要經過Garbage collection來清理空間。

因為回收過程比較危險，所以官方並不建議自動做，以下就簡單講講為了做刪除和回收，設定檔要怎樣改。為方便改設定，我們更新 docker compose yaml 檔，把 server config 都帶到 container 外面。
```diff
registry:
  restart: always
  image: registry:3
  ports:
    - 5000:5000
  environment:
    REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
    REGISTRY_HTTP_TLS_KEY: /certs/domain.key
    REGISTRY_AUTH: htpasswd
    REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
  volumes:
    - /path/data:/var/lib/registry
    - /path/certs:/certs
    - /path/auth:/auth
+   - /path/config.yml:/etc/distribution/config.yml
```

config.yml 就如下所示，為了提供 API 刪除 image 的可能，`storage.delete.enbled` 要為 `true`，又為著之後進行回收時，可以避免有人於回收中途上載，所以預先加入 `storage.maintenance.readonly.enabled` 的控制項。回收之前要把readonly改為true，回收後再調為false。 每次修改完，記得重啟一下 docker service 。
```yml
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
  maintenance:
    readonly:
      enabled: false
```

Garbage collection 指令
```sh
# inside container
# bin/registry garbage-collect [--dry-run] [--delete-untagged] [--quiet] /path/to/config.yml
bin/registry garbage-collect --delete-untagged=true /etc/docker/registry/config.yml

# outside container, at host level
docker exec -it YOUR_CONATINER_NAME bin/registry garbage-collect --delete-untagged=true /etc/docker/registry/config.yml
```

# draft
some image keep using same tag to keep check latest version; but older blob is not able to be deleted according to normal manifest garbage collection. Must use --delete-untagged flag to delete unused blob

quote from offical web site [https://distribution.github.io/distribution/about/garbage-collection/](https://distribution.github.io/distribution/about/garbage-collection/)
The --delete-untagged option can be used to delete manifests that are not currently referenced by a tag.

# Reference
- [https://distribution.github.io/distribution/about/insecure/](https://distribution.github.io/distribution/about/insecure/)
- [https://distribution.github.io/distribution/about/garbage-collection/](https://distribution.github.io/distribution/about/garbage-collection/)