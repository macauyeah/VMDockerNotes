# Notes about installing Multipass on Ubuntu Server 22.04 and config static IP

Install mulitpass on Ubuntu is easy, but config static IP is not. 

Another problem is official document stating that config static IP is beyond the scope of Multipass. As my experience, the [official example](https://multipass.run/docs/configure-static-ips) is not work on Ubuntu 22.04 server. (It might work on Ubuntu Desktop, but I am not sure).

Following is my solution after trial and errors.

## Install Multipass by snap
Easy step.
```
sudo snap install multipass
```

## Config a new Virtual Bridge on Host
Important, stop multipass before everything. Then install command tools "nmcli" with package "network-manager"
```
sudo snap stop multipass.multipassd
sudo apt-get update && sudo apt-get install network-manager
```

Modify NetworkManager config, so that it could manage all bridge interface.
By default, it excludes many types of interface. You need to add "***,except:type:bridge***" at the end of line 'unmanaged-devices' 
```
sudo vim /usr/lib/NetworkManager/conf.d/10-globally-managed-devices.conf
```

NetworkManager Config Example
```conf
[keyfile]
unmanaged-devices=*,except:type:wifi,except:type:gsm,except:type:cdma,except:type:bridge
```

Then reload NetworkManager.service, and add the virtual bridge with fix IP range.
```
sudo systemctl reload NetworkManager.service 
sudo nmcli connection add type bridge con-name localbr ifname localbr \
    ipv4.method manual ipv4.addresses 10.13.31.1/24
```
The new interface should appear now (but down) if you check by ```ip a```. 

## Switch Multipass to lxd
Multipass networks feature currently are only available on ***lxd*** backend.
```
sudo snap start multipass.multipassd
multipass set local.driver=lxd
```

Following commands are all same as official document.

## Create Extra interface in VM
Create extra interface when creating new new VM instance.
```
multipass exec -n test1 -- sudo bash -c 'cat << EOF > /etc/netplan/10-custom.yaml
network:
    version: 2
    ethernets:
        extra0:
            dhcp4: no
            match:
                macaddress: "52:54:00:4b:ab:cd"
            addresses: [10.13.31.13/24]
EOF'
```
Restart VM instance networks.
```
multipass exec -n test1 -- sudo netplan apply
```
And you should see the fix ip list on VM 
```
multipass info test1
```