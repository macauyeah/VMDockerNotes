# Docker 來源掃瞄 - Docker Image Scan
當網安要求越來越高時，我們也要留心 docker image 的來源是不是有漏洞問題。 docker hub 本身就已經有一些安全掃瞄報告，以 nginx 的 1.27.3 版本為例， [docker hub nginx 1.27.3](https://hub.docker.com/layers/library/nginx/1.27.3/images/sha256-1c93c0fdf35c65d1a441ea049552faea6395ce4d5ad96258344d6e65d4c8c29e?context=repo) ， docker hub 已經列出相當多的CVE漏洞。 不過對於不公開的 docker image ，安全描瞄可是要收費的。作為小團隊，可能想先尋求一些簡單的免費方案。如果你想同樣的需求，可能Trivy會幫到你。

## Trivy
[Trivy](https://trivy.dev/latest/) 是一個用於描瞄軟件版本依賴或設定檔是否引用到一些有漏洞問題的軟件，它也能檢測 docker image 是否有漏洞或錯誤設定的問題。而且更好的是， Trivy 本身亦有 Docker Image 版本，我們就不用煩惱怎樣弄一個 Trivy 的執行環境，只要可以運行 docker ，有網路就可以了。但使用 Docker Image 版的 Trivy 有一個額外要求，就是它要有主機上的 docker.sock 權限。

描瞄的指令如下，其中 docker.sock 就是為了讓 containers 內部的程式可以存取主機的 docker daemon , .cache 則是為了方便暫在下載資源。

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.cache/:/root/.cache/ aquasec/trivy:0.58.0 image nginx:1.27.3
docker run -v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.cache/:/root/.cache/ aquasec/trivy:0.58.0 image nginx:1.27.3-alpine
```

上面故意用 nginx 的兩個同版本號不同平台的 docker image，其實就是為了引出一些潛在問題。nginx 預設是使用的debain OS的，在筆者寫文章的當下，已經更新到最近的 image ，但始終有一大部份可能的漏洞。反觀 alpine OS 版本，就找不到這麼多問題。

```
nginx:1.27.3 (debian 12.8)
==========================
Total: 141 (UNKNOWN: 0, LOW: 93, MEDIUM: 30, HIGH: 16, CRITICAL: 2)

nginx:1.27.3-alpine (alpine 3.20.3)
===================================
Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```

這是因為 alpine 預設安裝的依賴較少，所以找到的漏洞也少。正所謂，做多錯多，唔做唔錯（大誤）。這其實有好有不好，因為在發生問題時，在 alpine 下可能連基本的除錯工具都沒有。除非大家有完整測試，或者對 alpine 有相當的認識，你才會選擇一個非官方預設的版本。但就以事論事，引用較少的依賴，長久之下的確是不會有那麼多隱患。大家如果有條件，也可以試試 alpine 或其他版本。

## podman 怎麼辦？
前一節我們可以看到，Trivy需要經過 socket 的方式才能存取主機上的 container daemon 操作權。但 podman 作為一個不主張 daemon (daemon less)，亦主張不需要 root (rootless)，那麼它該怎樣執行？

其實podman也有user層面上的 socket，而且 trivy 也有對應的方式去轉用第三方 socket (有點像使用遠端主機 socket，但官方並未宣佈正式支援遠端的方式。)

具體使用方式，筆者亦已在 steam deck 上測試，使用方式如下。不過因為 steam deck 預設沒有 root，筆者就省略 cache 指令，免得之後要有權限問題要手動清理。


```bash
$ systemctl --user start podman.socket
$ ls $XDG_RUNTIME_DIR/podman/podman.sock
/run/user/1000/podman/podman.sock
$ podman run -v $XDG_RUNTIME_DIR/podman/podman.sock:$XDG_RUNTIME_DIR/podman/podman.sock  \
--rm docker.io/aquasec/trivy:0.58.0 image --docker-host=unix://$XDG_RUNTIME_DIR/podman/podman.sock  \
nginx:1.27.3
```

# Ref
- [Podman socket activation](https://github.com/containers/podman/blob/main/docs/tutorials/socket_activation.md)
- [Trivy: Support for rootless podman](https://github.com/aquasecurity/trivy/issues/3098)




```bash
docker run -v /YOUR_PROJECT/:/opt/YOUR_PROJECT/ aquasec/trivy fs --scanners vuln,secret,misconfig /opt/YOUR_PROJECT/
```
