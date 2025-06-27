# Swarm mode 上線 7 - load balancer | 反向代理

前述我們介紹了負載平衡器的概念，也使用了nginx作為反向代理，管理網絡訪問，分流到對應的服務上( docker service )。 nginx是穩定的，大家初次使用 reverse proxy （反向代理），請選擇它，因為相對簡單，也易於在單機上做對比測試。

而nginx有個麻煩的地方，就是每次加 docker service，都需要更改 nginx 的設定。我們service 越多，config檔就越長。一個不少心，某些設定有衝突，就會讓 nginx 無法重起。

所以，我們在一定規模後，就需要改用自動化的反向代理。 [traefik](https://doc.traefik.io/traefik/) 就是其中之一。所恨的是，官方沒有提供 swarm 的範例，需要自行摸索。幸好筆者找到一個Github網路資源，[bluepuma77 traefik-best-practice](https://github.com/bluepuma77/traefik-best-practice/blob/main/docker-swarm-traefik/docker-compose.yml) ，內有一個traefik在docker swarm上的基本設定，足以解開筆者的某些謎思，至少可以讓筆者進行使用驗證。

bluepuma77 提供的範例可能還有些複雜，筆者就再簡化一下，讓大家可以從最基本的環境中開始。

下述 docker service 中
  - traefik: 自動偵測 swarm 中，有那些其他 service 需要經過traefik 代理。
  - whoami: 一個官方提供的簡單版http 回應，它正常可以回應 http 80的請求。
```yml
# traefik-stack.yaml
services:
  traefik:
    image: traefik:v3.4
    ports:
      - target: 80
        published: 80
        protocol: tcp
        # mode: host
    networks:
      - proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      #- /var/log:/var/log
    command:
      - --api.dashboard=true
      - --log.level=INFO
      #- --log.filepath=/var/log/traefik.log
      - --accesslog=true
      #- --accesslog.filepath=/var/log/traefik-access.log
      - --providers.swarm.exposedByDefault=false
      - --providers.swarm.network=proxy
      - --entrypoints.web.address=:80
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role==manager
  whoami:
    image: traefik/whoami
    networks:
      - proxy
    deploy:
      replicas: 3
      labels:
        - traefik.enable=true
        - traefik.http.routers.whoami.rule=Host(`whoami.localhost`)
        # other rules reference to https://doc.traefik.io/traefik/routing/routers/#path-pathprefix-and-pathregexp
        - traefik.http.services.whoami.loadbalancer.server.port=80
        # test command with curl, ingress seems only work on ipv4
        # curl -v -H 'host:whoami.localhost' http://127.0.0.1/

networks:
  proxy:
    name: proxy
    driver: overlay
    attachable: true
```

有一些重要的地方需要特別說明：
- 需要設定 `--providers.swarm.exposedByDefault=false`，不然traefik自己也需要定義反向代理的port。設定了這個，也可以讓 swarm 中某些 service 得以被忽略。有需要經 traefik 對外的，就在 `label` 下設定 `traefik.enable=true`
- 需要設定`--providers.swarm.network=proxy`，swarm中也需要有該網絡的存在。不然traefik 沒有預設的網絡可以走。
- 現時 docker service 使用是的 ingress mode，方便 traefik service 可以在不同的 manager 上遊走。測試時需要注意使用 ipv4 ，例如 curl 需要指定 ipv4 的 ip 即`curl -v -H 'host:whoami.localhost' http://127.0.0.1/` ，若直接使用 whoami.localhost ，有機會會指向 ivp6 ， ingress mode 就接不到。

# Reference:
- https://github.com/bluepuma77/traefik-best-practice/blob/main/docker-swarm-traefik/docker-compose.yml
- https://doc.traefik.io/traefik/routing/routers/#path-pathprefix-and-pathregexp
