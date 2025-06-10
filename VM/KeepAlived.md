# Virtual IP

雖然我們Docker Swarm、Galera等服務可以很容易地提供到Cluster的功能。但以用戶來講，怎樣知道該連線去那台伺服器，又是另一個問題。用戶不可能逐台伺服器逐台IP去訪問。通常，大家會以為在Cluster服務外部，加個 Load Balancer(負載均衡器)就已經可以解決問題。但其實Load Balancer本身也需要做Cluster，其中一個掛了，別的也需要頂上。那麼用戶到底是怎樣訪問伺服器的？

我們簡單地，可以經過 Virtual IP (簡稱VIP) 來解決這件事。即是把我們網絡服務的域名，綁到VIP上，然後這個VIP可以在不同伺服器上游走，只要有一台伺服器活著，都可以回應這個VIP的請求。而這個VIP的功能，可以經keepalived簡單地做到。

## 配置
假設我們的配置如下
- node 1 IP: 192.168.0.2, network interface: eth1
- node 2 IP: 192.168.0.3, network interface: eth1
- node 3 IP: 192.168.0.4, network interface: eth1
- virtual IP: 192.168.0.5

每個node，都有自己的IP，而virtual IP只會附在其中一台機上。

如果在 GaleraCluster 的情況下，可以看成只有virtual IP剛好附在其上的那台機工作，即是以 active passive 的方式運作。

如果在 Docker Swarm 的情況下，在預設模式下就已經有的mesh IP的機制，即使用virtual IP只在其中一台機上運作，但ingress networks都會擴散到所有機器上，所以是active active的方式運作。

## 設定 Keepalived
在三個node上，都各自安裝 keepalived。以下以 ubuntu 24.04 為例
```bash
# ubuntu 24.04
apt-get update && apt-get install keepalived -y
```

node 1 的 keepalived 設定
```conf
# vim /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    # change interface according to machine status
    interface eth1
    # one node is MASTER, other nodes are BACKUP
    state MASTER
    # all nodes in same group must be same value
    virtual_router_id 101
    # higher value will become master
    # ex, node1 priority 100, node2 priority 200, node3 priority 150.
    # if node 1, 2, 3 alive, node2 will become master.
    # if node 2 gone, node 3 will become master.
    priority 100
    # VRRP Advert interval in seconds (e.g. 0.92) (use default)
    advert_int 1
    virtual_ipaddress {
        192.168.0.5
    }
}
```

node 2 的 keepalived 設定
```conf
# vim /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    # change interface according to machine status
    interface eth1
    # one node is MASTER, other nodes are BACKUP
    state BACKUP
    # all nodes in same group must be same value
    virtual_router_id 101
    # higher value will become master
    priority 99
    # VRRP Advert interval in seconds (e.g. 0.92) (use default)
    advert_int 1
    virtual_ipaddress {
        192.168.0.5
    }
}
```

node 3 的 keepalived 設定
```conf
# vim /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    # change interface according to machine status
    interface eth1
    # one node is MASTER, other nodes are BACKUP
    state BACKUP
    # all nodes in same group must be same value
    virtual_router_id 101
    # higher value will become master
    priority 98
    # VRRP Advert interval in seconds (e.g. 0.92) (use default)
    advert_int 1
    virtual_ipaddress {
        192.168.0.5
    }
}
```

然後在各node上執行。
```bash
systemctl restart keepalived
```

上述設定中，除了 priority 外，應該所有 config 都一樣。state指的是初始化狀態，在 priority 不為255的情況下，即使設定 MASTER 或 BACKUP ，也會動態改變。又因為 priority 不是動態改變的，為方便管理及人眼辨識，筆者認為priority最高的那個node應該預設為MASTER，可以減少日後發生誤會。

如果一切正常的話，192.168.0.5只會出現在node1上。當node1掛了，192.168.0.5才會出現在node2。當node1、node2同時掛了，192.168.0.5才會出現在node3上。這個VIP，同一時間只會出現當時活著的機器中，priority最高的那一台。priority 最高的那一台，它的狀態為MASTER。這些狀態，我們可以經以下指令確認

```bash
# confirm state
systemctl status keepalived
# confirm ip
ip a | grep 192.168.0.5
```

## Keepalived 可能的異常
如果 Keepalived 之間無法溝通，每個node都自認為MASTER，192.168.0.5會同時出現在所有node上。這個情況下，網絡請求還是可能的，但當真正出現 failover (故障轉移)時，因為 ARP (Address Resolution Protocol) 等問題，路徑可能無法那上跳到活著的機器上，通常要等個十幾秒才會恢復。在前述的設定中， advert_int 就是各node溝通的時間間隔，以秒為單位。正常若果只有一個MASTER的話，failover可以在一至兩秒內完成。

造成 keepalived 無法溝通的原因很多，其中一個就為設定上的失誤，筆者初期就試過誤設定 virtual_router_id 。在有需要溝通的機器中，應該設定為相同的值。另一個原因則是防火牆，所幸的是 ubuntu 24.04 中， iptables 預設就接受它們之間的連線。如果是其他 Linux 版本，遇到無法溝通的情況，所以先關掉 iptables 服務，或者把 iptables 上的所有 rule 刪掉再試試。