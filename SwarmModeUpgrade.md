# Swarm mode 上線 6 OS升級前的準備

如果大家一直有跟進安全更新，基本上每個一至兩個月，都會有OS kernal和Docker Engine Update。也許大家習慣還是以不變應萬變，但有些時候還是不可避免地遇到嚴重漏動，需要強制更新。那麼當我們在這個情況時，我們該如何做呢？在開始做之前，我們先測試一下Swarm的容錯率有多高。

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