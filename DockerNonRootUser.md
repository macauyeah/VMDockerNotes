# Docker 中的非管理員用户 Docker non-root user

## Container USER為何重要

在制作Docker Image的過程中，有時會接觸到 USER 這個設定。這事關到最後的 Docker Container內部運行的那個 user 到底會有什麼權限。大家也要知道，Docker Container 其實也只是一個 Linux 上的程序，也就是如果Container內權限過大，也有機會從 Container 內部存取到 Host上的資料。

一般情況下，Docker Image 預設的 USER 就是 root，最基礎的base image都是一樣。而我們想換，其實也相當簡單，就像Linux上起User一樣，只要經指令`RUN adduser xxx` 或`RUN useradd xxx` 也可以在 Docker Image 中創建帳號和 home 資料夾，之後就隨時經`USER xxx`來切換

## 實際上是不是這麼簡單?

如果你將要Container中執行的程序，是一個binary，平常你在Linux中也是以 non-root 方式執行，那麼是的，就是那麼簡單。例如你執行系統中的java, node, python，原本在Linux中就已經是誰都可以，那麼你的docker container 也應該沒有難度。

但如果原本的安裝包，預設是由system service來啟動，我們就要花點力氣，看看那個service是怎樣呼叫binary的，然後就一步一步模擬它的做法。例如筆者有打包的codeserver，預設是system service啟動，但它也有提共binary的執行方法，安定好home資料夾後，我們也可以手動啟動。

## 泛生之檔案權限問題

上述binary的情境之所以簡單，是因為大部份情況下，我們都只對於container 內部運行考慮即可，因為預設投產情況下的運作模式，都是隨時起、隨時刪、隨時砍掉重練，只要container內部運作可以自給自足，就可以了。Docker Swarm的運作也是如此，所以它不預期有的持久化資料權限的問題。

而持久化資料權限的問題，其實早在單個Linux伺服器就已經存在。同一個伺服器中，不同process就有不同的UID，當他們需要共同讀/寫某些檔案，就會設定多人權限。同理，當多個Container要共同檔案，也是同樣問題。在討論共享檔案之前，我們先看看預設 Docker Storage Mount 會給我們什麼權限。

- 如果是bind mount，bind mount的權限預設會是Host內的檔案或者資料夾的權限。
    - 如果Host是root，container內是non-root，container有機會無法讀寫bind mount內的檔案。
        - 留意權限設置就可以解決問題
    - 如果Host是non-root，但container 內是root，從container內生成的檔案，Host的non-root user就無法使用。
        - Host是non-root的話就一定無解，Host至少有sudo權限，臨時變成管理員，去修正問題。
    - 如果host和container也是non-root，但UID不夾，其實也不能交換使用。
        - 跟上述一樣，最後要靠sudo來解決問題。
    - 如果host和container也是root，就沒有權限問題，但就有安全性的風險。
- 如果是volume mount，就還是看看 mount path 是docker image layer中現有的 path還是新起的path
    - 大部份手動建立的named volume都是root
    - 經docker compose起的named volume滿足以下條件的話，將會是non-root。
        - docker image 中的已有該path存在。
        - named volume未存在，docker compose會把對應path的內容在初次建立時抄到named volume 中。
        - 例如ubuntu:24.04中的/home/ubuntu，存在於docker image中，它的擁有者就是UID 1000，我們經docker compose `HOME_VOLUME:/home/ubuntu`，在HOME_VOLUME建立時，就會是UID 1000。但如果是 `NOT_EXISTS:/home/ubuntu/somethingNotExists`，那麼NOT_EXISTS建立時，也會是root

上述討論的Storage mount是集中在單機情況下，使用HOST OS的本地儲存。若現在的場境是多機共享的share storage，就會更麻煩，還要看看那個share storage本身的屬性。例如常見的Linux NFS，其實有指定的權限，跟NFS的Login權限有關，如果你的process本身對檔案權限很敏感，就請先不要挑戰NFS(例如postgresql)。

## Rootless mode - Rootless 模式

Rootless 模式指的是在Host中，執行Container的使用者，不需要是管理員，筆者就常用於開發環境中。投產環境中反而沒有聽過這樣的討論，因為投產環境很少可以讓非管理員去執行這麼重要的環境管理。

雖然只是開發環境，但這像前述的bind mount討論中，如果Host是non-root，但container 內是root，又或是兩者non-root，但UID不夾，也會出現權限問題。無腦的將host user加入docker group，只可以讓非管理員可以運行docker，但解決不了權限問題。

真正有條件解決的，可能就會向linux subgroup的方式發展。暫時筆者用得比較順的rootless mode，可以無腦用的，不是docker，是podman。有興趣的朋友可以經podman官網看看教學，它給筆者的感覺就像是自動轉換UID。

[podman rootless mode](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md)