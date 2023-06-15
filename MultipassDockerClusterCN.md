# 使用 Multipass 建立Docker Cluster

以下流程，假設各位已經
- 在Ubuntu Server中開設了virtual bridge 供Multipass設定[Static IP](MultipassStaticIpCN.md)，並且network interface定為 ***localbr***
- 使用Packer template制成[docker.img](https://github.com/macauyeah/ubuntuPackerImage/blob/main/template.json) , 並存放於當前資料夾內


使用docker.img 起三個node，並使用network interface ***localbr***，各有一個指定的mac address
```bash
multipass launch file://$PWD/docker.img --name node21 --network name=localbr,mode=manual,mac="52:54:00:4b:ab:21"
multipass launch file://$PWD/docker.img --name node22 --network name=localbr,mode=manual,mac="52:54:00:4b:ab:22"
multipass launch file://$PWD/docker.img --name node23 --network name=localbr,mode=manual,mac="52:54:00:4b:ab:23"
```

對運行中的三個node，為它們設定static ip
```bash
multipass exec -n node21 -- sudo bash -c 'cat << EOF > /etc/netplan/10-custom.yaml
network:
    version: 2
    ethernets:
        extra0:
            dhcp4: no
            match:
                macaddress: "52:54:00:4b:ab:21"
            addresses: [10.13.31.21/24]
EOF'

multipass exec -n node22 -- sudo bash -c 'cat << EOF > /etc/netplan/10-custom.yaml
network:
    version: 2
    ethernets:
        extra0:
            dhcp4: no
            match:
                macaddress: "52:54:00:4b:ab:22"
            addresses: [10.13.31.22/24]
EOF'

multipass exec -n node23 -- sudo bash -c 'cat << EOF > /etc/netplan/10-custom.yaml
network:
    version: 2
    ethernets:
        extra0:
            dhcp4: no
            match:
                macaddress: "52:54:00:4b:ab:23"
            addresses: [10.13.31.23/24]
EOF'

multipass exec -n node21 -- sudo netplan apply
multipass exec -n node22 -- sudo netplan apply
multipass exec -n node23 -- sudo netplan apply
```

使用node21作為Leader (Manager)，與其他兩個node一起組成Cluster
```bash
multipass exec -n node21 -- sudo docker swarm init --advertise-addr 10.13.31.21
multipass exec -n node21 -- sudo docker swarm join-token manager

managerToken=$(multipass exec -n node21 -- sudo docker swarm join-token manager -q)
multipass exec -n node22 -- sudo docker swarm join --token $managerToken 10.13.31.21:2377
multipass exec -n node23 -- sudo docker swarm join --token $managerToken 10.13.31.21:2377
```

Cluster就建立完成。

若想刪掉重來
```bash
multipass delete node21
multipass delete node22
multipass delete node23
multipass purge
```

## 備註
在直正使用時，大部份時間還需要做port forwarding。multipass沒有自己的port forward，可以用ssh tunnel來模擬。

例如把Ubuntu Server的8080指向node21的8080，可以這樣
```bash
sudo ssh -i /var/snap/multipass/common/data/multipassd/ssh-keys/id_rsa -L 0.0.0.0:8080:10.13.31.21:8080 ubuntu@10.13.31.21
```

完整的script可以參考[initDockerCluster.sh](https://github.com/macauyeah/ubuntuPackerImage/blob/main/initDockerCluster.sh)。

沒有Bare Metal Ubuntu或者沒有static ip也可以參考[initDockerClusterWithoutStaticIp.sh](https://github.com/macauyeah/ubuntuPackerImage/blob/main/initDockerClusterWithoutStaticIp.sh)。只是因為network brandwidth問題，我就不會在每次更新時都測試。