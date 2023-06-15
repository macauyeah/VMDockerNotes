# 在Ubuntu Server 22.04上安裝Multipass並配置固定IP的注意事項

安裝Multipass很容易的，但配置固定IP就不是了。

另一個問題是官方文件都認為設定固定IP不是Multipass的範圍，它不想講太多。 但根據我的經驗，它的[官方例子](https://multipass.run/docs/configure-static-ips)在Ubuntu Server 22.04並不能用。 (它可能可以在Ubuntu桌面上運行吧，但我不確定。)

以下是我在不斷踩坑後找到的解決方案。

## 通過snap安裝Multipass
簡單，無腦
```
sudo snap install multipass
```

## 在主機上配置Virtual Bridge
很重要，很重要，很重要，在所有操作之前停止multipass。然後使用"network-manager"包安裝命令工具"nmcli"。
```
sudo snap stop multipass.multipassd
sudo apt-get update && sudo apt-get install network-manager
```

修改NetworkManager配置，以便它可以管理所有bridge interface。預設情況下，有很多類型的接口它都不管的。您需要在'unmanaged-devices'行的末尾添加 "***,except:type:bridge***"。 
```
sudo vim /usr/lib/NetworkManager/conf.d/10-globally-managed-devices.conf
```

NetworkManager Config Example
```conf
[keyfile]
unmanaged-devices=*,except:type:wifi,except:type:gsm,except:type:cdma,except:type:bridge
```

然後重新運行NetworkManager.service，並添加具有固定IP範圍的Bridge network interface。
```
sudo systemctl reload NetworkManager.service 
sudo nmcli connection add type bridge con-name localbr ifname localbr \
    ipv4.method manual ipv4.addresses 10.13.31.1/24
```

現在，如果您使用```ip a```檢查主機所有網卡，新network interface應該已經出現（但有機會是處於關閉狀態）。

## 將Multipass切換到lxd
Multipass網絡功能目前只在***lxd***後端上提供。
```
sudo snap start multipass.multipassd
multipass set local.driver=lxd
```

以下指令與官方教學相同。

## 在VM中創建額外的網卡
在創建新的VM實例時創建額外的網卡。
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
重啟VM實例網絡。
```
multipass exec -n test1 -- sudo netplan apply
```
然後，您應該在VM上看到固定IP列表。
```
multipass info test1
```