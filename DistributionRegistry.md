# Distribution Registry

# tag 命名
一般來講，同一個docker image會提供多個不同的版本，每個版本會附予不同的tag，以作標識。但以docker image的維護者來講，它的tag通常代表的是自己程式的版本號


官方有時都會為舊image的tag更新base image，這樣可以確保底層的漏洞有修補到。頂層的用戶不用擔心如何更新，只要取得同一個tag的最新版本就可以了。

所以同一個tag，

[Offical Doc](https://distribution.github.io/distribution)