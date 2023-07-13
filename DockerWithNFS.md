# 自己架設Docker的共享儲存空間

Docker很好用，在單機環境下真的很好用。Docker原本的設計，是為了快速迭代而設計成Image的。在一般設定下，每次新建或重建container，都會根據Image重設一下各方面的環境，包括儲存空間。重設CPU，Memory，大家都很易理解，但重設儲存空間，真的不是每一個使用情況都可以這樣。

又或者說，未必所有使用情況都會有一個第三方的儲存空間可以用。所以良心的Docker在單機環境下也有提供bind mount或是docker named volume，作為可以長期保存，不受container生死的影響，以達到長期存在Data的存在。

## 單機-儲存空間
單機情況下很簡單，就用一個docker compose做例子
```
services:
  nginx:
    image: nginx:1.23
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./html/:/usr/share/nginx/html/
      - nginxlogs:/var/log/nginx/

volumes:
  nginxlogs:
```

其中html就是一個bind mount，而nginxlogs就是一個docker named volume，兩者都可以長期保存data，除非各位自己手動刪除，否則不會因為container的興亡而不見了。

但有兩個很重要的分別
- bind mount，直接跟host os連接，實際上是每次folder有更新，docker都要同步host和container之間的資料。
	- bind mount在linux下很暢順，因為大部份docker image/container原本就是linux engine，所以folder mount真的可以互通。
	- bind mount在windows / mac下，就會不斷抄資料。面對大量檔案，例如node_module，就會有速度上的問題
- docker named volume，就是docker 分離一些獨立空間，然後再綁到container上
	- 相對bind mount，即使在windows / mac下，都沒有那個速度上的問題。筆者猜測，即使是獨立空間，其實本身都已經限定在linux enginx下，所以沒有需要抄資料。
	- 但在windows / mac下，因應docker 底層建立Linux VM的技術不同，你可能沒法在windows / mac預設環境下直接讀取docker named volume。
	- 若要讀取docker named volume，最好的做法，還是連上docker container，然後用docker cp 來抄回資料。一但抄資料，其實都會有速度上問題，不過docker cp是手動決定何時做的，不做docker cp，其實container也是可以用。

## Cluster (Docker Swarm) - 儲存空間
雖然良心的bind mount和named volume解決了單機上的儲存問題，但到了cluster環境，就沒有可以跨機同步儲存空間的做法，要做就自己建立。

筆者也稍為研究了一下同步的問題，不過對技術真的很有要求。所以退而求其次，筆者還是選擇簡單的第三方儲存空間。就是做一個可以分享存取的NAS。

### 建立nfs
linux下要安裝nfs其實很簡單，不過要注意資料夾和防火牆權限。

以下安裝教學以ubunut 22.04為例，記得把下面的YOUR_DOCKER_NODE_ADDRESS_RANGE轉為你的真實IP段落
```
apt update && apt install nfs-kernel-server ufw -y

mkdir -p /mnt/nfs_share
chown -R nobody:nogroup /mnt/nfs_share/
chmod 777 /mnt/nfs_share/

echo "/mnt/nfs_share YOUR_DOCKER_NODE_ADDRESS_RANGE/24(rw,sync,no_subtree_check)" >> /etc/exports

exportfs -a

systemctl restart nfs-kernel-server

ufw allow from YOUR_DOCKER_NODE_ADDRESS_RANGE/24 to any port nfs
ufw allow ssh
ufw enable
```

### 修改docker compose
最後，你在原來的docker-compose的docker volume上加driver_opts就大功告成。

記得把下面的YOUR_NFS_IP轉為你的真實IP
```
volumes:
  nginxlogs:
    driver_opts:
      type: "nfs"
      o: "nfsvers=4,addr=YOUR_NFS_IP,nolock,soft,rw"
      device: ":/mnt/nfs_share/nginxlogs"
```