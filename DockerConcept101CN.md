# Docker 101
雖然Docker面世很久了，但筆者也是最近才摸清Docker一些概念和實務上的用途。趁著還有一些記憶，分享一些筆記。好讓未正式用過的朋友少走一些冤枉路。

## Docker是輕量VM? 
首先，第一個最大的普遍認知錯誤，就是認為Docker是輕量VM。

但其實它不是VM(更不是輕量VM)，它其實只是Linux上幫忙起Process的背景程式，官方叫作Docker Engine。個人感覺就像是在Host OS Kernal上的一個Playground (Sandbox會不會pro一點?)。我們平時最常使用的，是請Docker Engine根據Docker Image生成Container。而這些Container，因為一些特性，我習慣對非核心用戶解釋為【分身】。

整個生態系統中，最特別的是Container和Container之間，本身是獨立的。它們之間沒有Network, Storge也不會共用。除非你刻意連結起，否則它們理論上不會互相污染。但因為Kernal本身也是直接用Host OS，所以還是有機會被Hack而互相污染。

## Docker 可以打包萬用Linux Applicaiton?
第二個認知錯誤，就是認為Docker Container有persistent storage，可以看成VM，做一些Statefull application。

最陽春的Persistent storage是存在的，但並不是預期你會用到Production上面。因為Production大家考量的是做Cluster(例如Docker Swarm, K8s, KIND)，隨時增加或刪除多個相同Image來源的Container(分身)，以應付大流量。而經過Cluster的網絡routing mesh，即使同一個IP的Network request，也並不會一定派到同一個Container上。

## Docker Run?
第三個，不是認知錯誤，而是入坑起點問題。大部份教學，都會教大家怎樣包裝自己的Docker image，然後經docker run，產生container。

但其實一個application，並不是單一container可以包辦的事。例如，一個Wordpress網站，就會分了為兩個不同的Service(服務)來運作 來運作。而同一個Service，在需要的情況下，可以由一個或多個分身(Container)組成。

回到Wordpress的例子，應該看由成以下各部份組成的系統
- 一個資料庫的Service，靠一個Container作為資料儲存。
- 一個Web Service，靠一個或多個Php (Apache) Container來應對使用者Http Request。
- 經Docker Network把上面兩個Service連在一起。而且Web Service會對外開放Port，但資料庫就只會對內開放Port。

而一般的docker run，需要自行把上述所有事情串連起來。需要學習的command多，也很難感受整體的架構。個人覺得，在testing / development 環境下，容易上手的入門方式應該是docker compose。它可以同時行起多個Service/Container，提供同一個內部Network，把串連的動作簡化。更重要的是，docker compose主要是靠yaml file來做設定。讓新用戶可以更容易地重現別人做的好設定。

## 為何要做成Docker (Container - 容器化)
1. 做到隔離效果

    傳統上，同一機器安裝不同的 lib / dependency ，可能出現衝突。在 docker 的環境下，不同 container 之間可以隔離開，除了是網路之間出現引用關係的衝突外，動態庫的衝突就沒有見過。一般處理好 Persistent Volume 的考量，單機下是沒有什麼問題的。

2. 遷移的過程比較簡單

    傳統上，要把程式從一台機器搬到另一台機器，要預先安裝好相關的 lib / dependency 。但使用 docker / container 後，只要 docker 版本相容就好。docker image 本身，就已包括所有的 lib / dependency 。另一個常見的傳統問題，就是 Linux 檔案的擁有權問題，特殊情況下，新機同一個 user 的 ID 編號也不一樣，可能要手動恢復權限。如果是 container 的 bind mount 檔案，只要使用 tar command （`tar --same-owner -xvf file.tar`）保留權限解壓就好。

3.  垂直水平擴容

    因為有隔離及遷移方便的優勢，原本的機器達到上限，可以隨時換到其他機器上，修改對應的用戶入口就可以了（或更改DNS，可以更無縫連接）。一台機器不夠，亦可以多台機器一起來。即使不使用 docker swarm / k8s 方案，有傳統的 proxy gateway 再加單機的 docker ，就可以做到分流的效果。
    
    當然使用 docker swarm / k8s 才是正解，可以更簡化 proxy gateway 的設定。而傳統的分佈式問題，例如 Share Storage 等，其實就沒有簡化到，但也沒有增加難度。所以大家若考慮擴容的問題，更適合考慮使用 Container 的方案。
