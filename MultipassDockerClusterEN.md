# Create Docker Cluster with Multipass

The following instruction, assume you already prepared
- Setup Virtual Bridge in Ubuntu Server for Multipass to config [Static IP](MultipassStaticIpEN.md). And the network interface name is ***localbr***.
- A [docker.img](https://github.com/macauyeah/ubuntuPackerImage/blob/main/template.json) by Packer template, and put it in current working folder.


Create 3 nodes with docker.img, and attach to bridge network ***localbr*** with specific mac address
```bash
multipass launch file://$PWD/docker.img --name node21 --network name=localbr,mode=manual,mac="52:54:00:4b:ab:21"
multipass launch file://$PWD/docker.img --name node22 --network name=localbr,mode=manual,mac="52:54:00:4b:ab:22"
multipass launch file://$PWD/docker.img --name node23 --network name=localbr,mode=manual,mac="52:54:00:4b:ab:23"
```

Create static ip in 3 running nodes.
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

Let node21 as Leader (Manager) and create a cluster with other two nodes.
```bash
multipass exec -n node21 -- sudo docker swarm init --advertise-addr 10.13.31.21
multipass exec -n node21 -- sudo docker swarm join-token manager

managerToken=$(multipass exec -n node21 -- sudo docker swarm join-token manager -q)
multipass exec -n node22 -- sudo docker swarm join --token $managerToken 10.13.31.21:2377
multipass exec -n node23 -- sudo docker swarm join --token $managerToken 10.13.31.21:2377
```

That's all.

If you want to redo everything, delete them.
```bash
multipass delete node21
multipass delete node22
multipass delete node23
multipass purge
```

## Remark
In practical, most of time, we also need port forwarding. But Multipass no build-in port forward function. We might use ssh tunnel to simulate that.

Ex. create tunnel for Ubuntu Server port 8080 point to node21's 8080
```bash
sudo ssh -i /var/snap/multipass/common/data/multipassd/ssh-keys/id_rsa -L 0.0.0.0:8080:10.13.31.21:8080 ubuntu@10.13.31.21
```

You can find the complete script here [initDockerCluster.sh](https://github.com/macauyeah/ubuntuPackerImage/blob/main/initDockerCluster.sh)