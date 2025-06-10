# 不用Multipass，自動化還有什麼選擇？
因為multipass 升級同時轉換driver的關係，很久之前筆者介紹的multipass static ip 慢慢開始失效。如果大家只是為了做lab，雖然multipass預設的不是fix ip，但它的dhcp ip並不常更換，在multipass上起VM還是有一定優勢。

但若大家在更大的環境下，不可能有類似multipass exec 的型式去下指令，又或者，我們本地也沒有足資源做VM，必需使用公有雲，我們還有其他可以自動化的方法嗎?

有的。那就最初的ssh。

假設在公有雲，開了三台Linux VM，要作為聯機實驗用。我們只需要再一台Linux跳板機(可以是cloud VM或是local Mac / Linux，就可以順序以ssh為三台VM下指令。我們不需要開三台terminal，在不同VM之間切換，我們是直接在跳板機下指令，也就在跳板機上，實現自動化為三台機進行一系列的設定。

即是如果之前可以經multipass exec 完成的自動化，只要不涉及重置網絡操作，道理上也可以經ssh 實現。例如筆者之前的docker init可以這樣改寫

```bash
# local
multipass exec -n NODE_NAME -- sudo docker swarm init
# remote
ssh USERNAME@NODE_NAME -- sudo docker swarm init
```

抄檔案也可以改寫

```bash
# local
multipass transfer SOME_SCRIPT_FILE NODE_NAME:.
# remote
scp SOME_SCRIPT_FILE NODE_NAME:.
```

也因為公有雲或某些公司網絡，我們什少可以改變它的網絡設定，我們基本只可以使用預留的IP進行設定。不過也因為這樣，我們什少再作出重置網絡的操作。

但大家還是要留意，如果要真順暢ssh或scp，需要預先綁定ssh key。這些預先綁定ssh key的功能，一般在各大的public cloud都會有。如果沒有，我們也可以自動化開始之前，先使用ssh-copy-id為所有VM加入ssh key，這邊筆者就不再重複敍述。

## 參考資料

[https://www.cyberciti.biz/faq/what-does-double-dash-mean-in-ssh-command/](https://www.cyberciti.biz/faq/what-does-double-dash-mean-in-ssh-command/)