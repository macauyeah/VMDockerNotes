# Swarm mode 上線
之前一直都討論Image 的打包形式，現在聊聊部署上線時的一些指令。

## Docker Service
swarm mode 主要通過"docker service" 指令去產生一堆可以在不同節點上運行的container。為了更加形象地講，我把container稱為Image的分身。

```docker service create```跟```docker container run```的感覺很像，兩者都可以指定image 

```
# swarm mode
$ docker swarm init
$ docker service create --name nginx_s nginx

# single mode
$ docker container run -d --name nginx_c nginx
```

兩者的差別在於docker service 可以指定多少個分身，可以隨時加減數目，而且如果你有多過一台機器，分身就會在不同的機器上遊走。而docker container就是只對本機有操作，也不會散播到其他機器。
```
# swarm mode
$ docker service create --replicas=2 --name nginx_s nginx
$ docker service ls

ID             NAME      MODE         REPLICAS   IMAGE          PORTS
oyf0k1prmwdi   nginx_s   replicated   2/2        nginx:latest

$ docker service update --replicas=5 nginx_s
$ docker service ls

ID             NAME      MODE         REPLICAS   IMAGE          PORTS
oyf0k1prmwdi   nginx_s   replicated   5/5        nginx:latest


# single mode
docker container run -d --name nginx_c1 nginx
docker container run -d --name nginx_c2 nginx
docker container run -d --name nginx_c3 nginx
docker container run -d --name nginx_c4 nginx
docker container run -d --name nginx_c5 nginx
```

