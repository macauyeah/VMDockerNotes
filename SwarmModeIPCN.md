# Swarm mode 上線 4 | IP 設定

## 單機模式 IP設定
平常我們自己做測試，網絡功能通常用預設的就好。但當我們的Docker Container需要存取在區域網內的其他資源，避晚IP網段相衝是必需要的事。

大部份情況下，單機Docker使用的預設IP段會是

- 172.17.0.0/16
- 172.18.0.0/16
- ...

若然現在區域網中，有一段172.18.0.0/24，大家不想Docker踩到其中，可以修改設定檔，加入預設的default-address-pools，以後它就只會從指定的區段使用。

```json
# vim /etc/docker/daemon.json
{
  "default-address-pools": [
    {
      "base": "172.17.0.0/16",
      "size": 24
    },
    {
      "base": "172.19.0.0/16",
      "size": 24
    },
    {
      "base": "172.20.0.0/16",
      "size": 24
    }
  ]
}
```

其中base，是docker可以操作的總區域，size指的是Docker要自行分段的話，每段的大小是多少，上述的例子，就代表未來可能有以下Docker 網段。

- 172.17.0.0/24
- 172.17.1.0/24
- ...
- 172.17.255.0/24
- 172.19.0.0/24
- 172.19.1.0/24
- ...
- 172.19.255.0/24
- 172.20.0.0/24
- 172.20.1.0/24
- ...
- 172.20.255.0/24

修改完設定後，重啟Docker就會生效。當然，重啟前，先刪除所有不在預設範圍的所有Container。

## Swarm模式 IP設定
Swarm模式，與單機差不多，它需要在初始化Swarm就要定義，而且它不能與單機的網段有重疊。單機會預設使用Private IPv4 Class B，Swarm則是預設使用Private IPv4 Class A段，所以我們若就更改，就使用10.x.x.x吧。

```sh
docker swarm init --default-addr-pool 10.1.0.0/16 --default-addr-pool-mask-length 24
```

經上述例子初始化的 ingress 網段，將會是 10.1.0.0/24，隨後每個stack 則會是
- 10.1.1.0/24
- 10.1.2.0/24
- 10.1.3.0/24


## 重置Swarm
跟單機的情況類似，如果已建立Swarm後才修改網段，還是要整個刪掉重來。

每個節點都要執行以下指令。
```bash
docker swarm leave --force
```

實測swarm leave這個指令也會把所有運行中的stack刪掉。

各節點重新建立swarm
```bash
# in node 1, init new swarm with new ip
docker swarm init --default-addr-pool 10.1.0.0/16 --default-addr-pool-mask-length 24
# in node 1, get new manager token
docker swarm join-token manager
# in node 2 and node 3, join node 1 with new token
docker swarm join --token XXXXX YOUR_NEW_NODE1_IP:2377
```

## 雙管齊下
如果大家同想要修定單機及Swarm的網段，還要留意有一個特別的網段docker_gwbridge。它雖然是Swarm的附帶產物，但它則是受單機的網段控制。也就是，如果大家有需要同時修改單機及Swarm的網段，則需要手動刪除Swarm及docker_gwbridge

在每個節點先刪掉舊有的Swarm及docker_gwbridge，並關掉docker
```bash
docker swarm leave --force
docker network rm docker_gwbridge
```

在每個節點為docker_gwbridge修改設定，然後重起docker
```json
# vim /etc/docker/daemon.json
{
  "default-address-pools": [
    {
      "base": "172.17.0.0/16",
      "size": 24
    }
  ]
}
```

然後像前述一樣，重起Swarm。