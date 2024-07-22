# Swarm mode 上線
之前一直都討論Image 的打包形式，現在聊聊部署上線時的一些指令。

## Docker Service
swarm mode 主要通過"docker service" 指令去產生一堆可以在不同節點上運行的container。為了更加形象地講，我把container稱為Image的分身。

```docker service create```跟```docker container run```的感覺很像，兩者都可以指定image 

```bash
# swarm mode
$ docker swarm init
$ docker service create --name nginx_s nginx

# container mode
$ docker container run -d --name nginx_c nginx
```

兩者的差別在於docker service 可以指定多少個分身，可以隨時加減數目，而且如果你有多過一台機器，分身就會在不同的機器上遊走。而docker container就是只對本機有操作，也不會散播到其他機器。
```bash
# swarm mode
$ docker service create --replicas=2 --name nginx_s nginx
$ docker service ls

ID             NAME      MODE         REPLICAS   IMAGE          PORTS
uro4rwy6nelh   nginx_s   replicated   2/2        nginx:latest

$ docker service update --replicas=5 nginx_s
$ docker service ls

ID             NAME      MODE         REPLICAS   IMAGE          PORTS
uro4rwy6nelh   nginx_s   replicated   5/5        nginx:latest

# container mode
$ docker container run -d --name nginx_c1 nginx
$ docker container run -d --name nginx_c2 nginx
$ docker container run -d --name nginx_c3 nginx
$ docker container run -d --name nginx_c4 nginx
$ docker container run -d --name nginx_c5 nginx
$ docker container ls
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS         PORTS     NAMES
c45771f06612   nginx     "/docker-entrypoint.…"   7 seconds ago    Up 6 seconds   80/tcp    nginx_c5
a587a718da3a   nginx     "/docker-entrypoint.…"   9 seconds ago    Up 9 seconds   80/tcp    nginx_c4
079f206f8645   nginx     "/docker-entrypoint.…"   9 seconds ago    Up 9 seconds   80/tcp    nginx_c3
e10dc525fd22   nginx     "/docker-entrypoint.…"   10 seconds ago   Up 9 seconds   80/tcp    nginx_c2
dcaa2b4bb3de   nginx     "/docker-entrypoint.…"   10 seconds ago   Up 9 seconds   80/tcp    nginx_c1
```

在建立網段時也差不多，service需要的是overlay network，而container用一般network就可以。
```
# swarm mode
$ docker network create --driver overlay nginx_s_gateway
$ docker service update --network-add name=nginx_s_gateway,alias=gateway nginx_s
$ docker service ps nginx_s
ID             NAME            IMAGE          NODE         DESIRED STATE   CURRENT STATE             ERROR     PORTS
fxqtheyvr914   nginx_s.1       nginx:latest   dockertest   Running         Running 33 seconds ago
u0pvj1leoizw    \_ nginx_s.1   nginx:latest   dockertest   Shutdown        Shutdown 33 seconds ago
q7arumjlxduv   nginx_s.2       nginx:latest   dockertest   Running         Running 36 seconds ago
kurlwqfmopbg    \_ nginx_s.2   nginx:latest   dockertest   Shutdown        Shutdown 37 seconds ago
zd0zlkhxafv0   nginx_s.3       nginx:latest   dockertest   Running         Running 40 seconds ago
3kapr00fs6pt    \_ nginx_s.3   nginx:latest   dockertest   Shutdown        Shutdown 40 seconds ago
5o4afd3whygo   nginx_s.4       nginx:latest   dockertest   Running         Running 35 seconds ago
oxocropolbo8    \_ nginx_s.4   nginx:latest   dockertest   Shutdown        Shutdown 35 seconds ago
x5y94jf3ok51   nginx_s.5       nginx:latest   dockertest   Running         Running 38 seconds ago
cgld3au0w1i9    \_ nginx_s.5   nginx:latest   dockertest   Shutdown        Shutdown 39 seconds ago

# container mode
$ docker network create nginx_c_gateway
$ docker network connect --alias gateway nginx_c_gateway nginx_c1
$ docker network connect --alias gateway nginx_c_gateway nginx_c2
$ docker network connect --alias gateway nginx_c_gateway nginx_c3
$ docker network connect --alias gateway nginx_c_gateway nginx_c4
$ docker network connect --alias gateway nginx_c_gateway nginx_c5
```

不過比較大的差異是service會停了原有的分身，重開新的分身去加入網段。所以上面的docker service ps nginx_s執行結果，就有一半是停掉的。

類似地，docker service也不能單獨地停掉分身，頂多只能調整```--replicas=NUMBER```，來控制分身數量。而單機則可以經過```docker container stop```來暫停分身。

## Docker Stack
同樣地，在單機管理container時，可以通過內建的docker compose指令配搭docker-compose.yaml檔案，管理多個有協作關係的container。Swarm mode也可以通過docker stack去管理yaml檔案中的多個有協作關係的service。

因為docker compose已支援v3及Compose Specification標準；但docker stack暫時只能支援到v3格式，所以說yaml檔案不能完全照搬。但指令上行為是差不多的。

```
docker stack deploy -c setting.yaml stack-name
docker stack rm stack-name

docker compose -f setting.yaml up -d
docker compose down
```

docker stack也跟docker service類似，沒有隨時叫停的功能，而docker compose 就可以暫時叫停分身```docker compose -f setting.yaml stop```。

yaml例子待補完

## Docker Service Rollback
詳見[SwarmModeRollback.md](SwarmModeRollbackCN.md)