# Swarm mode 上線 5 - load balancer | 負載平衡器
前面我們一直談 swarm 的設定，但對於真實的服務，我們還要考慮客戶端是如何連接我們的伺服器群集。通常網路服務，客戶端都會經過域名轉換成IP，然而通過IP連線服務。

## Ingress Network
假設我們 swarm 內有5個節點，那到底域名應該指向我們哪一個節點的 IP 呢？

如果我們不考慮節點死機的話，其實5個節點的IP都可以。因為 swarm 會自動把同一個公開的 port ，在每一個節點上都可以訪問到。

以下例子，即使只有一個 container 運行，佔用 port 8888，它還是會在5個節點上全開。 swarm 通過自己的 ingress network，它所有節點的 8888 串連起來。
```yaml
services:
  http:
    image: bretfisher/httpenv
    ports:
      - 8888:8888
    deploy:
      replicas: 1
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
```

我們可以在每個節點上，都會找到這個 ingress network，而且那個Network ID，應該是一樣的
```
> docker network ls | grep ingress
t7rmk6g9zybm   ingress           overlay   swarm
```

如果上述的 service 的 replicas 調成大於1的數量， ingress network 還會方便地自動 round robin (輪替) 地分派流量，達到最簡單的負載平衡。

## Virtual IP
前述的設定，我們有一最大的假設，就是節點不會死機。但實際情況下，各種原因，例如安全性更新、重啟中，都會讓節點暫時無法使用。即使所有 service 都是會自動 failover (故障轉移)，但客戶端還是用舊機 IP ，它還是無法訪問。因為該機 IP 已無法使用，除非我們連 IP 也懂 failover。這時， Virtual IP 就是我們的救命靈藥。

在 ubuntu 上，我們可以經過 keepalived 去設定 Virtual IP
```bash
apt-get update && apt-get install keepalived -y
```

然後設定 keepalived ， 假設 172.22.1.5 是我們的 Virtual IP 。 然後每個節點都要加入conf
```conf
# vim /etc/keepalived/keepalived.conf
# assume failover ip is 172.22.1.5
vrrp_instance VI_1 {
    # change interface according to machine status
    interface eth1
    state MASTER
    # 101 for node1, 102 for node2
    # you can start seq from other value, remind unqiue for each node is ok; 
    virtual_router_id 101
    # lower value will become master
    # ex, node1 priority 100, node2 priority 200, node3 priority 150.
    # if node 1, 2, 3 alive, node2 will become master.
    # if node 2 gone, node 3 will become master.
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass YOUR_RANDOM_PASSWORD
    }
    virtual_ipaddress {
        172.22.1.5
    }
}
```

上述需要特別注意的是
- virtual_router_id : 每個節點應該都要不一樣，以作唯一標識。
- priority : 每個節點應該都要不一樣，最大的那個節點，就會優先使用 Virtual IP 。
- auth_pass : 每個節點都相同，但大家在抄時，記得更改。

還有的是開通 iptables ，讓各個節點可以經網絡廣播的方式互相看到對方。
```bash
iptables -I INPUT -d 224.0.0.0/8 -j ACCEPT
iptables -I INPUT -p vrrp -j ACCEPT
systemctl restart keepalived
```

## proxy gateway load balance
前面的例子，我們已經成功設定 ingress Network，也加了 virtual ip 。如果大家的目標是單一 web 應用，應該就已經很足夠。但作為一個足夠節儉的老闆，怎會讓一個 Swarm 只跑一個 Web 應用？但問題來了，一個 docker swarm service 就已經佔用一個公開端口 (例如上述的8888，或是更常見的443)。怎麼可以做到多個 service 分享同一個端口？答案就是回到傳統的 Web Server 當中，使用它們的 virtual host 及 proxy 功能，以達到這一效果。我們就以 Nginx 為例，去建立一個守門口的網關 (gateway) 。

以下就是一個最簡單的例子，最前端的 http-gateway (nginx) 對外公開端口 8080 ，它根據 virtual host，去分派對應的請求去 dmzhttp (bretfisher/httpenv) 及 managerhttp (bretfisher/httpenv) 。構架圖就是以下這樣。
```
                              ┌───────────┐  
              ┌──────────────►│  dmzhttp  │  
              │               └───────────┘  
              │                              
           ┌───────────────┐                 
           │ http-gateway  │                 
  ────────►│ (nginx:8080)  │                 
           └──┬────────────┘                 
              │                              
              │              ┌─────────────┐ 
              └─────────────►│ managerhttp │ 
                             └─────────────┘ 
```

換成 docker stack ，就大概如下
```yaml
services:
  http-gateway:
    image: http-gateway
    ports:
      - 8080:8080
    deploy:
      replicas: 1
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
  dmzhttp:
    image: bretfisher/httpenv
    deploy:
      replicas: 2
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
  managerhttp:
    image: bretfisher/httpenv
    deploy:
      replicas: 3
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
```

docker stack有一個很好的功能，就是 service 名會自動成為同一段網絡中的 hostname 。即是http-gateway中，它可以經DNS，找到 dmzhttp 、 managerhttp，也就是它的 nginx 可以設定成如下的樣子。
```conf
# default.conf
server {
	listen       8080;
	listen  [::]:8080;
	server_name  managerhttp;
	resolver 127.0.0.11 valid=30s;

	location ^~ / {
		set $upstream_manager managerhttp;
		proxy_cache off;
		proxy_pass http://$upstream_manager:8888$request_uri;
	}
}
server {
	listen       8080;
	listen  [::]:8080;
	server_name  dmzhttp;
	resolver 127.0.0.11 valid=30s;

	location ^~ / {
		set $upstream_dmz dmzhttp;
		proxy_cache off;
		proxy_pass http://$upstream_dmz:8888$request_uri;
	}
}
```
上面的例子中，就是一般的 virtual host + nginx proxy 設定。特別要說明的是 resolver 那一行，它指向 docker DNS (127.0.0.11)， 而且還可以讓nginx在找不到上游時，不要馬上死亡。這樣 docker swarm 中各個 service 隨時加加減減，有保命的作用。

最後我們的 http-gateway 就是 nginx image + default.conf 上述的 docker 就可以用以下方式打包。
```
# Dockerfile
# docker image build -t http-gateway ./
FROM nginx:latest
COPY default.conf /etc/nginx/conf.d/default.conf
```

上面的 docker stack 和 nginx config，只要同步增加 service 及對應的 proxy pass，就可以o讓同一個端口，根據不同hostname做分流。當然，如果大家可以共用端口及 hostname 也可以，分流就改用 nginx location 來設定，不過這是更加偏向 nginx 的內容，日後有機會再介紹。本篇就先集中於 docker 相關的議題。

在安全性的角度， docker 還有一些配置可以做，就是讓 dmzhttp 和 managerhttp 在不同的機器上發佈。假設我們的網絡分開兩段，一段是 manager 專用，一段是 dmz 專用。在建立 docker swarm 後，我們可以為不同的節點加入對應的標簽。

```bash
docker node update --label-add zone=manager YOUR_MANAGER_NODE
docker node update --label-add zone=dmz YOUR_DMZ_NODE
```

然後我們通過修改 docker stakc 中的 placement -> constraints ，限制不同的 service 在不同的節點上運行。
```diff
services:
  http-gateway:
    image: http-gateway
    ports:
      - 8080:8080
    deploy:
      replicas: 1
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
  dmzhttp:
    image: bretfisher/httpenv
    deploy:
      replicas: 2
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
+     placement:
+       constraints:
+         - node.labels.zone==dmz
  managerhttp:
    image: bretfisher/httpenv
    deploy:
      replicas: 3
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
+     placement:
+       constraints:
+         - node.labels.zone==manager
```
使用上面的例子，我們就可以達到簡單分離的效果。但大家緊記，這個分離效果始終是一個規則式功能，它與防火牆的隔離還是有本質上的區別。除了利用傳統的防火牆技術外，我們的docker swarm network，其實也可以做更多隔離，我們日後再慢慢加強這個例子。

# 基於安全性的各種考量
前面介紹了 ingress network ，亦介紹了 proxy gateway 。能做到的基本都做到了，再來就是考量安全性的問題。因為加了 proxy gateway ，前述的例子是所有 service ，都放在同一個 yaml 檔中。好處是，所有相關的東西存放在同一個檔中， gateway ，背後的 service 都一眼看到。但壞處就是有其中一個 service 更新，都要改那個 yaml 檔。更大的問題是， stack deploy 的指令，不單只更新其中一個 service ，就連其他 service 都會自動取得最新 image 而 redeploy 。

對於一個緊密的系統來講，同步更新可能不是大問題。但對於一些預定排程發佈的系統可不能這樣因為副作用而更新了。如果你也有這樣的分開管理需求，可以參考下面做法，把 gateway service 及 upstream service 放在不同的檔案中，然後經過 external network把所有 service 串連起來。
```yaml
# nginx-stack.yaml, docker stack deploy -c nginx-stack.yaml nginx
services:
  http-gateway:
    image: http-gateway
    ports:
      - 8080:8080
    deploy:
      replicas: 1
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure

# manager-stack.yaml
services:
  managerhttp:
    image: bretfisher/httpenv
    networks:
      - nginx_default
      - default
    deploy:
      replicas: 3
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.labels.zone==manager
networks:
  nginx_default:
    external: true

# dmz-stack.yaml
services:
  dmzhttp:
    image: bretfisher/httpenv
    networks:
      - nginx_default
      - default
    deploy:
      replicas: 2
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.labels.zone==dmz
networks:
  nginx_default:
    external: true
```

這樣，不同 service 的維護人員，就可以獨自控制自己的檔案。在第一次發佈時，確認 nginx-stack.yaml 先行發佈就可以了。對應的發佈指令是`docker stack deploy -c nginx-stack.yaml nginx`，它會自動産生一個 nginx_default （即 stack名字_default ）的網絡。之後其他service，就可以經networks的設定找到它了。

```yaml
services:
  YOUR_SERVICE:
    networks:
      - nginx_default
      - default

networks:
  nginx_default:
    external: true
```

上述即使分離檔案，在安全性考量時還是有一個問題，就是 ingress network 的問題。試想一下，dmzhttp （Demilitarized Zone）原本被設定的原因，就是想限制某些訪問只能一些可以公開的服務。但因為經過 ingress network 之後，它們會在所有機器上開放這些 port。那就是，以下面的例子來講，若 dmzhttp 是公開的服務， intrahttp 是內部服務，即使用 intrahttp 使用不同的port 8889。但一經 swarm mode 預設的 ingress network ，在`node.labels.zone==dmz`的那些節點，還是可以訪問到 intrahttp 。

```yaml
services:
  dmzhttp:
    image: bretfisher/httpenv
    ports:
      - 8888:8888
    deploy:
      replicas: 2
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.labels.zone==dmz
  intrahttp:
    image: bretfisher/httpenv
    ports:
      - 8889:8888
    deploy:
      replicas: 3
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.labels.zone==intra
```

我們前述介紹的 proxy gateway ，其實已經有一定程度可以解決這個問題。因為 proxy gateway 是根據 http 協定中的 host header 去做分流。在邊界網絡進來的「合法」訪問，道理上會好好地經引導到我們的 dmzhttp 。不過網路的邪惡可容小看， proxy gateway 也會有被騙的一日。有特定能力的攻擊者，只需找到目標域名，還是可以接觸到 intrahttp 。

若要做進一步隔離，在這種情況下，我們可以在 dmz , intra 機器中各設定一套 swarm ，完全獨立，這是最安全的做法。但這樣做的管理成本就會變高，因為兩個網段都會有自己的 manager 節點，而且在 dmz 網段的 manager 節點也有被攻擊的可能。

若我們回到單一 swarm 的方向，可以修改各個 service 中的 port 和 deploy 。利用 post mode 中的「host」，配合 deploy mode 中的「global」，完全跳開 ingress network。
```yaml
services:
  dmzhttp:
    image: nginx
    ports:
      - target: 80
        published: 8888
        mode: host
    deploy:
      mode: global
      update_config:
        delay: 1s
      restart_policy:
        condition: any
      placement:
        constraints:
          - node.labels.zone==dmz
  intrahttp:
    image: bretfisher/httpenv
    ports:
      - target: 8888
        published: 8888
        mode: host
    deploy:
      mode: global
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.labels.zone==intra
```

上面的例子中， dmzhttp 會在所有 dmz 的機器中，每個節點只運行一份服務，而且直接使用該機的 8888 port ，外面不會再有 ingress network 的 存在。同樣地，intrahttp 會在 intra 的所有節點，運行一份服務，佔用它們的8888 。這兩個服務，即使使用一個 port ，swarm 也不會說有任何問題。因為它們不會經 ingress network 搶佔其他人的 8888。
