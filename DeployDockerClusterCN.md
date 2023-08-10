# Docker In Production

大部份在大公司工作的朋友，在公開的應用情景中，應該都會直接選擇Public Cloud(公有雲)的Kubernetes作為Production(投產環境)。但對於澳門的中、小微企，要找一個附合個人資料保護法的Public Cloud，可真難。想要簡單地靠自己架設本地Kubernetes，也不是一件易事。退而求其次，想要快速擁有一個Cluster(群集)，Docker是一個最直觀的選擇。

之前筆者都介紹過Swarm mode的一些指令，但指令只是執行的手法，怎樣管理你的App還是有不同的選擇。

## 把App轉成Swarm mode 還是把底層程式變成Swarm mode?

雖然筆者對於Docker Swarm Mode的資歷尚淺，但由於後期更動的難點越來越多，筆者很想早一點討論其中不同操作的差異

## Docker Swarm
Docker Swarm Mode其實是Docker提供的一個Cluster(群集)環境。在其中運行的Image，都可以比較方便地隨時分身到不同的node(節點)上，對於提高負載或可用性，都是一個不錯的解決。

只要該Image跑起的Container是Stateless(前後兩次執行的結果互不相干涉)，或者是把Stateful的部份(有干涉的部份)外包到第三方(例如儲存空間使用NFS，或記憶體暫存改為Key-Value Database)，就可以方便地運作在Docker Swarm mode上。

## 部署Docker Swarm的選項
Docker Swarm可以把Image變成分身Container，但並沒有硬性改變傳統App操作方式。大部份App在執行時，都需要另一個底層程式的支緩。例如
- Php Web App，需要底層php fpm + nginx或apache
- Java Web App，就需要java + Tomcat

所以在發佈App時，可以選擇把
- App直接打包成Image
- 只把底層程式打包在Image中(例如Tomcat)，再在跑起Container時再動態接起App。

### 兩者有何差別?
就信心層面上，一定是把App直接打包成Image實際一點。因為這樣可以極大地減少測試環境和正式環境的差異而出現的問題。筆者一開始也不完全讚成，但也越來越傾向這種做法。

在解釋筆者為何有這個結論前，先條列式地對比一下兩種差別。

| 事項              | 打包App成為Image | 打包底層程式成為Image |
| :---------------- | :------  | :------ |
| 打包複雜度 | 需要把App用到的一些環境變數引入設定Image的entrypoint中，方便配合不同的環境可以改變App的行為。<br>打包次數根據App數量有關。<br>比較靈活，但比較需要學習和試錯。</li> | 底層程式統一設定環境變數，其中所有App都會使用類似或相同的設定，設定方式跟傳統方式無異。<br>打包次數根據底層程式數量有關。<br>比較死版，但要試錯的成本較低 |
| 發佈流程 | 打包App成Image。再靠Docker Swarm設定Image有多少分身，每個分身不需要特別再設定。 | 原來的底層程式已存在於Docker Swarm中，只需把新建或更新了的App放入不同分身的儲存空間，讓底層程式動態跑起App。 |
| 管理複雜度 | 每個App都是獨立的，代表有任何更新也是獨立更新。<br>在微服務的協作環境中，需要管理員從Image層面為每個App設定網絡(network)或開放端口(Port)。但每個App可以設定不同的分身數量，靈活性一定比只打包底層程式要高。 | 使用同一個底層程式Image的App都會使用同一個網路和端口設定。<br>在微服務的協作環境中，管理員要應付的設定數量一定比打包App要少。<br>但由於是Conatiner分身是針對底層程式，所以若然某個App有不同需求，就要重新設定另一套底層程式。 |

上述幾個點，最後其實都是複雜度和靈活度的取捨。雖然打包App的工序更多，但提供的靈活性也更多。如果考慮要從傳統模式中過渡，方便與完成不懂Docker的同事協作，就首選打包底層程式。如果考慮可重複性和信心保證，還是打抱App比較直接，要複制一個環境到另一個環境，也比較易測試。

# CI/CD with gitlab