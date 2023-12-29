# Swarm mode 上線 - 3 | Rollback
Docker Swarm提供了一個很方便的rollback功能。針對swarm service的config都可以用。官方提供了一些[rollback config](https://docs.docker.com/engine/reference/commandline/service_rollback/)的例子。

今天筆者今天就來個自己更常見的例子，rollback image

例設我們使用docker stack和yaml檔來操作docker service。
```yaml
# nginx-rollback.yaml
services:
  nginx:
    image: nginx:1.25.2
    ports:
      - 8080:80
    deploy:
      replicas: 2
      update_config:
        delay: 1s
      restart_policy:
        condition: any
```

首次建立nginx service, 當前版本會是nginx:1.25.2
```bash
> sudo docker stack deploy -c nginx-rollback.yaml nginx-test
### system output
Creating network nginx-test_default
Creating service nginx-test_nginx

> sudo docker service ps nginx-test_nginx
### system output
ID             NAME                 IMAGE          NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
kn6o6p45rxtq   nginx-test_nginx.1   nginx:1.25.2   node2     Running         Running 8 minutes ago
x1qa0tmhh8w6   nginx-test_nginx.2   nginx:1.25.2   node1     Running         Running 8 minutes ago
```

然後嘗試更新nginx:1.25.3
```yaml
# nginx-rollback.yaml
services:
  nginx:
    image: nginx:1.25.3
    ports:
      - 8080:80
    deploy:
      replicas: 2
      update_config:
        delay: 1s
      restart_policy:
        condition: any
```
```bash
> sudo docker stack deploy -c nginx-rollback.yaml nginx-test
### system output
Updating service nginx-test_nginx (id: nlure4s6ipnzuy4qki66dz8o9)

> sudo docker service ps nginx-test_nginx
### system output
ID             NAME                     IMAGE          NODE      DESIRED STATE   CURRENT STATE             ERROR     PORTS
83m9iyc2xg5q   nginx-test_nginx.1       nginx:1.25.3   node2     Running         Running 3 seconds ago
kn6o6p45rxtq    \_ nginx-test_nginx.1   nginx:1.25.2   node2     Shutdown        Shutdown 7 seconds ago
8a9vidondmyo   nginx-test_nginx.2       nginx:1.25.3   node1     Ready           Preparing 2 seconds ago
x1qa0tmhh8w6    \_ nginx-test_nginx.2   nginx:1.25.2   node1     Shutdown        Running 2 seconds ago
```

假設我們發現nginx:1.25.3有些副作用不是我們想要的，可以經docker service rollback回到上次的版本，即是nginx:1.25.2
```bash
> sudo docker service rollback nginx-test_nginx
###
nginx-test_nginx
rollback: manually requested rollback
overall progress: rolling back update: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged

> sudo docker service ps nginx-test_nginx
ID             NAME                     IMAGE          NODE      DESIRED STATE   CURRENT STATE                 ERROR     PORTS     
0mm3hzp0aaok   nginx-test_nginx.1       nginx:1.25.2   node2     Running         Running about a minute ago
83m9iyc2xg5q    \_ nginx-test_nginx.1   nginx:1.25.3   node2     Shutdown        Shutdown about a minute ago
kn6o6p45rxtq    \_ nginx-test_nginx.1   nginx:1.25.2   node2     Shutdown        Shutdown 10 minutes ago
cb1pdqw0bmyx   nginx-test_nginx.2       nginx:1.25.2   node1     Running         Running about a minute ago
8a9vidondmyo    \_ nginx-test_nginx.2   nginx:1.25.3   node1     Shutdown        Shutdown about a minute ago
x1qa0tmhh8w6    \_ nginx-test_nginx.2   nginx:1.25.2   node1     Shutdown        Shutdown 10 minutes ago
```