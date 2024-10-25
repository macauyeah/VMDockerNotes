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
以下就是一個最簡單的例子，最前端的 http-gateway 對外公開端口 8080 ，它根據 virtual host，去分派對應的請求去 dmzhttp 及 managerhttp。

all in one
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
      # placement:
      #   constraints:
      #     - node.labels.zone==dmz
  managerhttp:
    image: bretfisher/httpenv
    deploy:
      replicas: 3
      update_config:
        delay: 10s
      restart_policy:
        condition: on-failure
      # placement:
      #   constraints:
      #     - node.role==manager
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