## virtual ip for failover
in each vm node, install keepalived for ip failover
```bash
apt-get update && apt-get install keepalived -y
```


node 1 192.168.0.2
node 2 192.168.0.3
node 3 192.168.0.4

VIP 192.168.0.5
config of keepalived in node1
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


config of keepalived in node2
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

config of keepalived in node3
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

除了priority外，應該所有config都一樣。state指的是init state，在 priority 不為255的情況下，即使設定 MASTER 或 BACKUP 也可以，但因為priority不是隨機設定，為方便管理及人眼辨識，各node的config中，筆者認為priority的那個node應該預設為MASTER，日後可以減少誤會發生。