# Docker 上戰場 - Docker Swarm

大部份在大公司工作的朋友，在公開的應用情景中，應該都會直接選擇公有雲 (Public Cloud) 的 Kubernetes 作為 投產環境 (Production)。但對於澳門的中、小微企，要找一個附合個人資料保護法的公有雲，可真難。想要簡單地靠自己架設本地 Kubernetes ，也不是一件易事。退而求其次，想要快速擁有一個群集 (Cluster) ，Docker Swarm 是一個最直觀的選擇。

在介紹 Swarm mode 的之前，我們先聊一聊可以有哪些選擇，也就是怎樣管理你的App還是有不同的選擇。雖然筆者對於 Docker Swarm Mode 的資歷尚淺，但由於後期更動的難點越來越多，筆者很想早一點討論其中不同操作的差異。

## Docker Swarm
Docker Swarm Mode其實是 Docker 提供的一個群集 (Cluster)環境。在其中運行的 Docker Image，都可以比較方便地隨時分身到不同的node(節點)上，對於提高負載或可用性，都是一個不錯的解決。

只要該 Image 跑起的Container是 無狀態的 - Stateless(前後兩次執行的結果互不相干涉)，或者是把狀能儲存 - Stateful的部份(有干涉的部份)外包到第三方(例如儲存空間使用NFS，或記憶體暫存改為Key-Value Database)，就可以方便地運作在Docker Swarm mode上。

## 部署Docker Swarm的選項： 把App轉成Swarm mode 還是把底層程式變成Swarm mode?
Docker Swarm可以把Image變成分身Container，但並沒有硬性改變傳統App操作方式。大部份App在執行時，都需要另一個底層程式的支緩。例如
- Php Web App，需要底層php fpm + nginx或apache
- Java Web App，就需要java + Tomcat

所以在發佈App時，可以選擇把
- App直接打包成Image
- 只把底層程式打包在Image中(例如Tomcat)，再用Docker Volume的方式讓Container可以起動App。

### 兩者有何差別?
就信心層面上，一定是把App直接打包成Image實際一點。因為這樣可以極大地減少測試環境和正式環境的差異而出現的問題。筆者一開始也不完全讚成，但也越來越傾向這種做法。

在解釋筆者為何有這個結論前，先條列式地對比一下兩種差別。

| 事項              | 打包App成為Image | 打包底層程式成為Image |
| :---------------- | :------  | :------ |
| 打包複雜度 | 需要把App用到的一些環境變數引入設定Image的 `entrypoint` 中，方便配合不同的環境可以改變App的行為。<br>打包次數根據App數量有關。<br>比較靈活，但比較需要學習和試錯。</li> | 底層程式統一設定環境變數，其中所有App都會使用類似或相同的設定，設定方式跟傳統方式無異。<br>打包次數根據底層程式數量有關。<br>一般有官方發佈的版本。<br>比較死版，但要試錯的成本較低 |
| 發佈流程 | 打包App成Image。再靠Docker Swarm設定Image有多少分身。多個Image的協調，靠著Swarm完成。 | 原來的底層程式已存在於Docker Swarm中，只需把新建或更新了的App放入不同分身的儲存空間，讓底層程式動態跑起App。但若底層程式也要需要因應App數量而改變，有機會底層程式也需要重新打包。|
| 管理複雜度 | 每個App都是獨立的，代表有任何更新也是獨立更新。<br>在微服務的協作環境中，需要管理員從Image層面為每個App設定網絡(network)或開放端口(Port)。但每個App可以設定不同的分身數量，靈活性一定比只打包底層程式要高。 | 使用同一個底層程式Image的App都會使用同一個網路和端口設定。<br>在微服務的協作環境中，管理員要應付的設定數量一定比打包App要少。<br>但由於是Conatiner分身是針對底層程式，所以若然某個App有不同需求，就要重新設定另一套底層程式。 |
| 資源消耗 | 每個App獨立，代表資料消耗也是獨立的。總消耗就是倍數增長。但可以經Docker限定每個Image的使用量 | 因為底層程式共用，多少有可以省略的地方。資源限制沿用底層程式的傳統邏輯。 |

上述幾個點，最後其實都是複雜度和靈活度的取捨。雖然打包App的工序更多，但提供的靈活性也更多。如果考慮要從傳統模式中過渡，方便與完全不懂Docker的同事協作，就首選打包底層程式。如果考慮可重複性和信心保證，還是打包App比較直接，要複制一個環境到另一個環境，也比較易測試。

打包底層程式的做法不是大問題，不過一定要考慮的是多個App之間是怎樣建構、多點佈署該如何實現，設定檔的備份又該如何做。因為Image內沒有太多關於佈署的設定，想偷賴也不行。若設定檔、多個App都底層程式都一起打包到同一個Image中，這其實是一個多Service的小系統。除非多個App很緊密連接，不然不推薦這種做法。因為每個App有更新，都會讓所有App重啟，就會對實際使用帶來很大的不便。

### 到底如何選？
如果大家上正式的Docker課程，Docker導師通常會推薦為每個App打包成獨立Image，因為底層程式的Overhead(釋譯成上頭成本？)通常不大，例如底層程式是Tomcat、Apache、Nignx這類網頁伺服器，重量級的開銷並不是因為多幾個Web Engine的分身造成，通常都是因為業務本身。但如果你講的底層程式是資料庫等的大型程式，才可能會有明顯的差異。

實務上的建議，就是必需考慮自身的經驗，到底那個方案自己比較有把握。獨立打包App，在正式環境也需要考慮跟蹤問題的情況，多個不同App要溝通，需要了解Container網絡。如果打包底層程式，所有App都可以當成是本機下運行，會讓你更有信心追蹤問題，也是一個很好的出發點。到了有需要彈性改變不同App的分身數量，才轉向獨立打包的做法。

筆者最初也是走打包底層程式的方向，到了自己有信心試用Docker Swarm，才走向獨立打包的做法。筆者親身經驗，因為到了Docker Swarm，網段會變得暴增，這跟公司現有的內部網絡相衝的機會就會變多。在Swarm建立初時，筆者並沒有意識到這件事，所以當初排查問題，也花了一些時間才知道要向網段衝突上著手。

另一個出自Docker導師實務上的建議，就是正式環境中不要做用Docker compose，應該使用Docker Swarm。那怕Swarm只有一個節點，也應該用Swarm，導師的主要理據是Swarm有Rolling Update （滾動更新）的機制。同一個node也可以有多個分身，每個分身輪流更新，就不會出現大中斷的情況。筆者就自身經驗，Tomcat可以同時容納一個App的多個版本，Nginx也有Failover（故障轉移）等，如果你很熟悉這些功能，就不一定要需要靠Swarm去提供。也就是可以按自己步調去慢慢改變。

# CI/CD系統的參與
CI/CD 全稱是continuous integration (CI) 和 continuous delivery (CD)，字面上代表的持續地集成和發佈，實體上就是某台伺服器自動發佈APP。因為使用到Docker Cluster，不論前述什麼選擇，都會有多個node(節點)的出現。要發佈App，總不能一個個node逐個登入設定。所以我們需要一些CI/CD工具，把這個過程都自動化。

在筆者的認知上，CI/CD系統，由兩個部份組成，一個是取得Source Code(程式原始碼)的過程，一個是編譯或發佈Source Code的過程。Gitlab，Github，BitBucket等大型的代碼庫供應商，它們天生為了保存Source Code而提供服務的。不少CI/CD系統都可以跟它們整合，它們提供了存取Source Code的部份，剩下你只要能提供編譯或發佈的伺服器就好。

如果作為小型開發團隊，很少會有意願去自己花錢養一個編譯或發佈的伺服器。(極端地，如果我就是一人團隊，我用自己電腦編譯和發佈就好，伺服器能做的，我自己也能做。)好消息的是，Github提供了一個叫Github Action的CI/CD系統，即使你沒有自己的編譯專用的伺服器，Github Action也可以用Docker Image，提供一個臨時的編譯程序，用完就刪掉。詳細功能還請各位先查看官方教學，筆者也暫時只能零星使用經驗，無法給出有意思的架構。

如果對智慧財產權有高度重視，Source Code不能存放在公開的伺服器，那麼Gitlab Enterprise Edtion則是一個好選擇。運用Gitlab ee，你可以用自己的機器，造一個純本地的庫存伺服器。更強的是，它內建也有CI/CD系統，只要你有間置的伺服器，就可以作為編譯使用。筆者也是從這個方向著手，架設了自己的Gitlab Runner(Gitlab CI/CD系統)。在這裏，就分享一下與Docker Swarm整合的概念。

對於前述兩種選擇，GitLab Runner都可以做得到
- 底層程式打包成Image並運行在Swarm mode上，每次發佈的是App Binary(執行檔或核心檔案)。
- 把App直接打包成Image，並運行在Swarm mode上，每次發佈的是App Image。

## CI/CD - 打包底層程式成為Image
在這個選擇下，其實就跟傳統自動化發佈的做法類似，只是發佈時，要多個node報行更新指令。如果你使用的底層程式原本就有支援多版本並行，這樣更新時就不用太操心rollback(回滾?)等操作。若系統不支援多版本並行，為求簡化，若遇到要rollback的情況，重跑過去舊的CI/CD操作也是一個做法。當然，我們也可以經過一些備份的操作，來保存被代替的程式，若在發佈過程中出問題，也可以手動重來，不過整件事就越來越複雜。

筆者發佈的基本思路是
- 使用docker image，編譯和打包App Binary。
	- 使docker image做編譯的好處是，你可以比較放心地假設每次編譯時，你的編譯環境都是乾淨的。
- 傳送上述的結果至生產環境可以取用的地方。
- 跳入生產環境執行更新指令

```yml
# file .gitlab-ci.yml
# need Docker executor, the gitlab runner agent need docker permission
stages:
  - deploy

maven-deploy:
  image: maven:3-eclipse-temurin-17
  stage: deploy
  script:
    - "mvn clean compile package -Pprod -Dmaven.test.skip=true"
    # ${ID_RSA} is a rsa private key file location, it can skip password input during ssh or scp operation. 
    # you need to pre-config production serevr auth before exec this script
    - "chmod og= ${ID_RSA}"
    - "scp -i ${ID_RSA} -o StrictHostKeyChecking=no app.war YOUR_ACCOUNT@PRODUCTION_SERVER_1:./"
    - "ssh -i ${ID_RSA} -o StrictHostKeyChecking=no YOUR_ACCOUNT@PRODUCTION_SERVER_1 UPDATE_COMMAND"
    - "scp -i ${ID_RSA} -o StrictHostKeyChecking=no app.war YOUR_ACCOUNT@PRODUCTION_SERVER_2:./"
    - "ssh -i ${ID_RSA} -o StrictHostKeyChecking=no YOUR_ACCOUNT@PRODUCTION_SERVER_2 UPDATE_COMMAND"
```

這裏有些隱藏的管理成本，如果你生產環境中有多個node，最後那幾行指令就要多抄幾次。

## CI/CD - 打包App成為Image
在這個選擇下，對比傳統自動化發佈的做法，現在要多做一步，就是要包裝自己的Image。不過好處是docker swarm有提供監測工具，在發佈過程每個分身會逐個更新，前一個分身更新成功後才會到下一個分身更新。而且
rollback等的操作，你可以靠docker做到。即是要手動rollback，也可以透過更正docker tags來達到，所以整體上來說沒有比傳統的麻煩。

筆者發佈的基本思路是
- 編譯App Binary。
- 打包成docker image。
- 經docker上傳image。
- 跳入生產環境執行更新指令。

```yml
# file .gitlab-ci.yml
# need Shell executor, the gitlab runner agent also need docker permission
stages:
  - deploy

docker-deploy:
  stage: deploy
  script:
    - "mvn clean compile package -Pprod -Dmaven.test.skip=true"
    - "docker image build -t YOUR_IMAGE_PRIVATE_REPO ./"
    - "docker image push YOUR_IMAGE_PRIVATE_REPO"
    # ${ID_RSA} is a rsa private key file location, it can skip password input during ssh or scp operation. 
    # you need to pre-config production serevr auth before exec this script
    - "chmod og= ${ID_RSA}"
    - "ssh -i ${ID_RSA} -o StrictHostKeyChecking=no YOUR_ACCOUNT@PRODUCTION_SERVER SWARM_UPDATE_COMMAND"
```

對比傳統自動化發佈的做法，最後的更新指令，只要執行一次就可以。當然，原本在Docker Swarm中要管理的事還是要好好管理。

## CI/CD - 備註事項
雖然CI/CD可以幫忙簡化更新的過程，但實際操作會比上述的例子複雜一些。因為通常對非技術型的外界用戶來說，一個Web App會包含很多不同的功能。上述的例仔，在實際情況下可能需要拆解成很多微服務來進行。所以對管理上還是有相當的挑戰。


# DRAFT
## CI/CD 其實要分開
CI是編譯 binary/docker image 的運程，CD是發佈的過程。編譯與發佈，並不一定對等實現。多次編譯，可以最後才會發佈一次。發佈也有分不同環境，發佈到測試場，還是投産？測試場與投産的也會存在版本差異，通常新一個版本還在測試，投産則用前一個版本。

如果有清楚的版本號，CI會産生一個帶版本號的binary或image，CD則選擇不同的版本號配不同的config。如果要更準確地rollback ，CD還要有自己的版本號。為了靈活CD，減少CI的次數。

目前script base可以 default branch event - build image (CI)， tag event deploy (（)CD)。 build image 時留一個有 commit hash 的docker tag ，之後 git tag 就根據當時的 commit hash 找回原本的image / binary，進行 docker stack deploy 的指令。這樣可以減少tag時需要執行的build 指令。不過CD如何以，就要以另一個repo儲起不同env的config。同一個repo儲config，也會出現改config後要重build的情況。