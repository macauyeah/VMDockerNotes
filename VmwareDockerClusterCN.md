# Vmware下建立Docker Cluster

之前都使用Multipass作為Proof of Concept，自己做測試用。直正上Production，Network環境就多少有點差異。

假設大家為Application Admin，但無條件處理Vmware層面上的事項，只可以從VM內部install / setup application。

## 安裝Docker
script 都來自[Docker 官方網](https://docs.docker.com/engine/install/ubuntu/)，筆者微調了一些auto accept選項。

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -kfsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

假設三台VM已經安裝docker，ip分別為 10.13.31.21, 10.13.31.22, 10.13.31.23。

在其中一台VM上，例如:ip 10.13.31.21上，
```
sudo docker swarm init --advertise-addr 10.13.31.21 --data-path-port=7777
```

與前述Multipass不同的是，這裏的data-path-port要自定義，因為預設的***port 4789***在Vmware的有特殊同途。

之後部份就跟傳統做法一樣，先取得manager join token, 然後在其他VM上使用該token加入cluster
```
# at 10.13.31.21
sudo docker swarm join-token manager

# at 10.13.31.22, 10.13.31.23, 
sudo docker swarm join --token XXXXXXXXXXXXXXXXXXXXXXXXXX 10.13.31.21:2377
```

這樣，在swarm上的application，就會自動在10.13.31.21, 10.13.31.22, 10.13.31.23，上遊走。

即使你的app container目前是跑在10.13.31.23:8080上，但因為swarm mode routing mesh，你經過10.13.31.21:8080都可以連到該app。

如果你只是做stateless app load balance分流，這樣就足夠了，不用考慮ip fail over。但如果你要做到ip fail over，還要額外設定keepalive virtual ip，這個virtual ip會自動依付到某台活著的VM上，這樣外界才不會連到一個死ip上。又或者額外建一台load balancer，可以偵測到swarm node上那台機還活著，從而達到fail over效果。但這台load balancer也有一些穩定性要求，若然大家只是用一個普通的nginx做load balance，還是會有單點故障問題(single point of failure)。