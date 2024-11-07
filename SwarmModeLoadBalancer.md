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

# draft
## virtual host load balance
前面的例子，我們已經成功設定 ingress Network，也加了 virtual ip 。如果大家的目標是單一 web 應用，應該就已經很足夠。但作為一個足夠節儉的老闆，怎會讓一個 Swarm 只跑一個 Web 應用？但問題來了，一個 docker swarm service 就已經佔用一個公開端口 (例如上述的8888，或是更常見的443)。怎麼可以做到多個 service 分享同一個端口？答案就是回到傳統的 Web Server 當中，使用它們的 virtual host 及 proxy 功能，以達到這一效果。我們就以 Nginx 為例，去建立一個守門口的網關。

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

然後我們通過修改 docker stakc 中的 placement -> constraints ，限制不同的 service 在不同的節點上運行。這樣，我們就可以達到簡單分離的效果（但 nginx networks 還是會互通被 hack ? 所以最好還是不要使用同一network作為媒介？ 經過public port會好一些嗎？ 雲原生的 proxy gateway 可能就做得到？）
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

seperate
```yaml
# nginx-stack.yaml
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
      # placement:
      #   constraints:
      #     - node.role==manager
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
      # placement:
      #   constraints:
      #     - node.labels.zone==dmz
networks:
  nginx_default:
    external: true
```