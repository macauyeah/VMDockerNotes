# Swarm mode 上線 - 2

系統上線一段日後，總會遇到有需要更改的時候，我們一般要用的程式，可以經過Docker Image包裝，變得無痛。但Docker Swarm本身要有改動，就不是那麼的直白。

以筆者的經驗，若果Swarm mode的各位node都原地升級版本，其實都不難。但若果那些最基本的設定，例如node1的IP換了，那基本上等於要整個Swarm mode砍掉從來。

筆者最近就需要將整個Docker Swarm中多個node都一起轉IP。原本以為關掉改主機IP，Docker Swarm可以隨時重起。但問題是，在改IP過程中，其他Node2、Node3發現Node1的不見了，若然Node2、Node3經過cold start，它們的Docker daemon根本行不起來。若然要在這一刻硬來，就需要整個Docker daemon重設。這比當初以為的只刪除Docker Swarm要來得恐怖。

比較好一點的做法，應該是在改IP之前，先把Swarm mode中各個stack (service, network, volume)手動移除，然後把Swarm mode解除。

在Docker相關事項都變成可以獨立運作後，再做主機的更新

```bash
# delete stack and leave swarm in node 3, node 2, node 1
# don't shutdown docker in node 2,3 if node 1 is not available
docker stack ls
docker stack rm YOUR_STACK_NAME

# to be confirm if any dingling network, service is still there.
# not sure how stack handling re-deploy problem when something removed from yaml file
docker network ls
docker network rm YOUR_STACK_NETWORK

docker service ls
docker service rm YOUR_STACK_SERVICE

docker volume ls
docker volume rm YOUR_STACK_VOLUME

docker swarm leave --force

## begin host changes
## finish host changes, re-create cluster

# in node 1, init new swarm with new ip
docker swarm init --advertise-addr YOUR_NEW_NODE1_IP
# in node 1, get new manager token
docker swarm join-token manager
# in node 2 and node 3, join node 1 with new token
docker swarm join --token XXXXX YOUR_NEW_NODE1_IP:2377
```