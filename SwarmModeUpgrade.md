# Swarm mode 上線 6 - Host OS 升級流程

如果大家一直有跟進安全更新，基本上每個一至兩個月，都會有OS kernel和Docker Engine Update。也許大家習慣還是以不變應萬變，但有些時候還是不可避免地遇到嚴重漏動，需要強制更新。那麼當我們在這個情況時，我們該如何做呢？在開始做之前，我們先測試一下Swarm的容錯率有多高。

官方就宣稱，只要swarm中，manager例下的數目沒有超過半數，就依然可以運行。這個部份，筆者相信大家一早就感受過。但筆者認為，在直正出意外的情況是，少於半數的manager倒下了，但其餘的manager又不幸要重啟，到底又些活著的manager，又可否成功重新啟動？所以下面就來做些測試。

## 測試1
- [測試REPO](https://github.com/macauyeah/ubuntuPackerImage)
- [初始化script initDockerCluster.sh](https://github.com/macauyeah/ubuntuPackerImage/blob/main/initDockerCluster.sh)

筆者在原本的教學中，就有一個3個manager node + 2 worker node的範例，我們只要安裝ubuntu OS, packer, 並使用

```
# run setupMultipassWithFixIP.sh will install multipass and config fix ip
./setupMultipassWithFixIP.sh


# install packer, please refer to https://github.com/macauyeah/ubuntuPackerImage/blob/main/README.md
packer init template.pkr.hcl
packer build template.pkr.hcl

# initialize docker cluster in multipass
./initDockerCluster.sh
```

在上述的環境中，node21, node22, node23是manager, node24, node25則是worker。在全關的情況下，只要正常啟動兩台manager，worker就可以成功復活。

## 測試2
- [測試REPO](https://github.com/macauyeah/ubuntuPackerImage)
- [初始化script initDockerClusterLoopJoin.sh](https://github.com/macauyeah/ubuntuPackerImage/blob/main/initDockerClusterLoopJoin.sh)

筆者寫了另一個起始方案，會有5個manager node，而且依賴順序如下

- node22 => node21
- node23 => node22
- node24 => node23
- node25 => node24
- node21 => node22


```
# initialize docker cluster in multipass
./initDockerClusterLoopJoin.sh
```
在上述的環境中，5台機也是manager。 initDockerClusterLoopJoin 的前半部份，它會建立順序依賴，即每一台機器，都經前一台機器進行加入的動作。而後半部份會把node21刪掉，並經multipass用一個全新的guest os重起，重新加入到。

在全關的情況下，只要正常啟動三台manager，它們都可以繼續運行。

這個測試的例子表明，即使原本作為依賴的機器死了，只要群齊中其餘多數manager仍然存在，它們也是可以復活的。更重要的是，即使最初引領一切的node21死掉了，什至是被刪掉重來，也是相安無事的。

## 結論
更新時，最保守的做法，是先加入新的manager，再除去舊有的manager。但這個做法下，manager的IP就不可避免地被改變。若然DNS或者防火牆沒有相應的自動化幫忙，先加入再替換就變得很痛苦了。

而上述的測試，其實代表了我們可以先暫停或移除舊有的manager，更新完後再接回去，這樣部份IP就可以重用。我們只要維持多於原本半數的manager活著，然後逐一替換或升級原有的機器，也不會有問題。即使在升級途中，其他manager不幸地斷線，重啟後它們還是有條件自行修復。我們也不需要顧及更新順序，只需想好Virtual IP的分配策略就足夠，其餘就像是單機升級一樣。


# 實戰升級流程
假設我們有5個 node，都為manager，各個 docker 版本都為28.0.4 ，我們將要關掉node 5 (ubuntu 22)，並加入node 6 (ubuntu24)，更新流程如下
1. 如果node5有vvip，login node 5，關掉vvip
    - `systemctl stop keepalived && systemctl disable keepalived`
1. login node1, 把node5降為drain模式，變為worker，並從群集中刪除
    - `docker node update --availability drain node5`
    - `docker node demote node5`
        - 若然node5不是直接關機、刪除，只想好好地離開群集，可以 login node5, 在node5上預先執行 `docker swarm leave`
    - `docker node rm --force node5`
        - 如果之前node5有好好地離開群集，而且狀態已經轉為down，那麼就不用"force"了，用最保守的刪除指令就可以 `docker node rm node5`
1. login node1, 取得manager token
    - `docker swarm join-token manager`
1. node5關機，新增node6，使用相容的ip段，或者使用node5的ip
1. login node6, 加入群集，設定vvip
    - `docker swarm join --token xxxx XX_IP:XX_PORT`
    - `apt-get update && apt-get install keepalived -y`
    - `systemctl start keepalived`

上述的操作，有一些可能的陷阱，筆者就剛好踩過，未來不知道會不會有官方保證

- docker的版本需要相同，不同版本可能不能加入群集，例如
    - docker 28.0.4 不能加到 docker 27.5.1。
    - docker 27.2.x 不能加到 docker 27.5.1。
- docker swarm，官方雖然宣稱支援不同版本共存，但這指的是已加入的node，在不解綁的情況下原機升級。
- 在swarm已有多版本共存的情況下，有一個node選擇完全脫離，它想再加入，也是會失敗的。可能這不是docker自身的限制，而是底層library的相容性問題。筆者在實測不同版本時，就得到這樣的error。`docker credentials: cannot check peer: missing selected ALPN property`
