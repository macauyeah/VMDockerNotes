當網安要求越來越高時，我們也要留心 docker image 的來源是不是有漏洞問題。 docker hub 本身就已經有一些安全掃瞄報告，以 nginx 的 1.27.3 版本為例， [docker hub nginx 1.27.3](https://hub.docker.com/layers/library/nginx/1.27.3/images/sha256-1c93c0fdf35c65d1a441ea049552faea6395ce4d5ad96258344d6e65d4c8c29e?context=repo) ， docker hub 已經列出相當多的CVE漏洞。 不過對於不公開的 docker image ，安全描瞄可是要收費的。作為小團隊，可能想先尋求一些簡單的免費方案。如果你想同樣的需求，可能Trivy會幫到你。

[Trivy](https://trivy.dev/latest/) 是一個用於描瞄軟件版本依賴或設定檔是否引用到一些有漏洞問題的軟件，它也能檢測 docker image 是否有漏洞或錯誤設定的問題。而且更好的是， Trivy 本身亦有 Docker Image 版本，我們就不用煩惱怎樣弄一個 Trivy 的執行環境，只要可以運行 docker ，有網路就可以了。但使用 Docker Image 版的 Trivy 有一個額外要求，就是它要有主機上的 docker.sock 權限



```bash
docker run -v /YOUR_PROJECT/:/opt/YOUR_PROJECT/ aquasec/trivy fs --scanners vuln,secret,misconfig /opt/YOUR_PROJECT/

docker run -v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.cache/:/root/.cache/ aquasec/trivy:0.58.0 image nginx:1.27.3
docker run -v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.cache/:/root/.cache/ aquasec/trivy:0.58.0 image nginx:1.27.3-alpine
```

