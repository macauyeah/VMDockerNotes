# Swarm mode 上線 7 - load balancer | 反向代理

前述我們介紹了負載平衡器的概念，也使用了nginx作為反向代理，管理網絡訪問，分流到對應的服務上( docker service )。 nginx是穩定的，大家初次使用 reverse proxy （反向代理），請選擇它，因為相對簡單，也易於在單機上做對比測試。

而nginx有個麻煩的地方，就是每次加 docker service，都需要更改 nginx 的設定。我們service 越多，config檔就越長。一個不少心，某些設定有衝突，就會讓 nginx 無法重起。

所以，我們在一定規模後，就需要改用自動化的反向代理。 [traefik](https://doc.traefik.io/traefik/) 就是其中之一。官方有提供一個 swarm 的範例，需要設定的部份相對比較多。幸好筆者找到一個Github網路資源，[bluepuma77 traefik-best-practice](https://github.com/bluepuma77/traefik-best-practice/blob/main/docker-swarm-traefik/docker-compose.yml) ，內有一個traefik在docker swarm上的基本設定，足以解開筆者的某些謎思，可以讓筆者進行使用驗證。

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

# SSL
做完上述的使用驗證後，我們可以正式開始看官方的例子，該例子加入了SSL，這就更充份地體現反向代理的用途。

[官方教學連結](https://doc.traefik.io/traefik/setup/swarm/)

官方的yaml也很長，筆者實測了一個簡化版本。

```yaml
# traefik-ssl.yaml
services:
  traefik:
    image: traefik:v3.4
    ports:
      - target: 443
        published: 443
        protocol: tcp
    networks:
      - traefik_proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    configs:
      - source: dynamic-tls.yaml
        target: /dynamic/tls.yaml
    secrets:
      - source: certs-local.key
        target: /certs/local.key
      - source: certs-local.crt
        target: /certs/local.crt
    command:
      - --api.dashboard=true
      - --log.level=INFO
      - --accesslog=true
      - "--providers.file.filename=/dynamic/tls.yaml"
      - --providers.swarm.exposedByDefault=false
      - --providers.swarm.network=traefik_proxy
      - --entrypoints.websecure.address=:443
      - --entrypoints.websecure.http.tls=true
      #- --entrypoints.websecure.asDefault=true
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role==manager

  # Deploy the Whoami application
  whoami:
    image: traefik/whoami
    networks:
      - traefik_proxy
    deploy:
      labels:
        # Enable Service discovery for Traefik
        - "traefik.enable=true"
        # Define the WHoami router rule
        - "traefik.http.routers.whoami.rule=Host(`whoami.swarm.localhost`)"
        # Expose Whoami on the HTTPS entrypoint
        #- "traefik.http.routers.whoami.entrypoints=websecure"
        # Enable TLS
        - "traefik.http.routers.whoami.tls=true"
        # Expose the whoami port number to Traefik
        - traefik.http.services.whoami.loadbalancer.server.port=80

networks:
  traefik_proxy:
    name: traefik_proxy
    driver: overlay
    attachable: true

configs:
  dynamic-tls.yaml:
    file: ./dynamic/tls.yaml

secrets:
  certs-local.key:
    file: ./certs/local.key
  certs-local.crt:
    file: ./certs/local.crt
```

餘下的就照跟官方設定

生成cert file。（或大家有正式的證書，就可以免去這一步。）
```bash
mkdir -p certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/local.key -out certs/local.crt \
  -subj "/CN=*.swarm.localhost"
```

指向cert的動態設定檔。
```yaml
# dynamic/tls.yaml
tls:
  certificates:
    - certFile: /certs/local.crt
      keyFile: /certs/local.key
```

然後我們就可以這樣測試
```bash
curl -v -k -H 'host:whoami.swarm.localhost' https://127.0.0.1/
```

筆者在一開始時，始終無法設定 `dyanmic/tls.yaml` ，其實是筆者誤會了 traefik 的讀取方式。本個例子中，traefik 其實會動態讀取 swarm 及 file provider 的設置，而`dyanmic/tls.yaml`是經過file provider的方式生效。也就是 `traefik-ssl.yaml` 中的`"--providers.file.filename=/dynamic/tls.yaml"`。

本個例子與官方例子最大的不同，是官方的cert, tls, 是直接使用bind mount的方式存取，如果你有多過一個manager，這個方式不太有效。本文就用了swarm config及swarm secret，方便多個manager自動配置。不過swarm config及swarm secret都有個缺點，若要更新它們的內容，就必需要重命名（例如dynamic-tls.yaml=> dynamic-tls.yaml2） ，否則swarm不允許發佈。

完整 yaml 請見 [github](https://github.com/macauyeah/ubuntuPackerImage/tree/main/traefik)

# Reference:
- https://github.com/bluepuma77/traefik-best-practice/blob/main/docker-swarm-traefik/docker-compose.yml
- https://doc.traefik.io/traefik/routing/routers/#path-pathprefix-and-pathregexp
- https://doc.traefik.io/traefik/setup/swarm/
- https://github.com/macauyeah/ubuntuPackerImage/tree/main/traefik