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


# draft

[Offical Doc](https://distribution.github.io/distribution)


# Garbage collection on registry

config
```yml
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
```

command
```sh
bin/registry garbage-collect --delete-untagged=true /etc/docker/registry/config.yml
```
some image keep using same tag to keep check latest version; but older blob is not able to be deleted according to normal manifest garbage collection. Must use --delete-untagged flag to delete unused blob
