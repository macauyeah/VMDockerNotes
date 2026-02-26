# Qemu 助你快速測試國産 OS
不知道大家最近有沒有在考慮使用國産OS，如果有，大家又是怎樣做初步測試的呢?

筆者之前一直都笨笨的從ISO開始安裝，所以每試不同的版本，都要從零開始。不但重複，OS安裝階段的複制過程也是很耗時的。但其實國産OS，大部份都有qcow2的格式，我們若只是本地測試的話，其實可以利用qcow2來做快速VM生成。

## 阿里aliyun

之前筆者有介紹過ubuntu的multipass，若你想試用的國産OS就是阿里aliyun，只要你有ubuntu，就可以快速跑起來。

但如果沒有ubuntu，只要有qemu（多平台），也是可以的。

```bash
qemu-system-x86_64  \
  -cpu host -machine type=q35,accel=kvm -m 2048 \
  -nographic \
  -snapshot \
  -netdev id=net00,type=user,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net00 \
  -drive if=virtio,format=qcow2,file=aliyun_3_x64_20G_nocloud_alibase_20251215.qcow2 \
  -drive if=virtio,format=raw,file=seed.img
```

上述指令中的qcow2和seed.img，比可以在官方網站可以下載的。預設帳號 `alinux` 密碼 `aliyun`。

seed.img是用於cloud-init的，就是初始化VM所用。第二次開機時，就不需要再使用seed.img

```bash
# remove the last line file=seed.img
qemu-system-x86_64  \
  -cpu host -machine type=q35,accel=kvm -m 2048 \
  -nographic \
  -snapshot \
  -netdev id=net00,type=user,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net00 \
  -drive if=virtio,format=qcow2,file=aliyun_3_x64_20G_nocloud_alibase_20251215.qcow2
```

## 華為OpenEuler

同樣地，我們也是用qemu可以跑起OpenEuler，只是它沒有seed.img（不支持cloud init），所以直接跑起就好。

```bash
qemu-system-x86_64  \
  -cpu host -machine type=q35,accel=kvm -m 2048 \
  -nographic \
  -netdev id=net00,type=user,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net00 \
  -drive if=virtio,format=qcow2,file=openEuler-24.03-LTS-SP3-x86_64.qcow2
```

預設帳號 `root` 密碼 `openEuler12#$`

第二次開機，指令也是一樣。

## 龙蜥AnolisOS

方式也一樣。

```bash
qemu-system-x86_64  \
  -cpu host -machine type=q35,accel=kvm -m 2048 \
  -nographic \
  -netdev id=net00,type=user,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net00 \
  -drive if=virtio,format=qcow2,file=AnolisOS-8.10-x86_64-ANCK.qcow2
```

預設帳號 `anuser` 密碼 `anolisos`

## 總結

如果只是要體驗一下國産OS，Qemu快速起VM就足夠。但Qemu本身只有NAT網絡，想要做VM之間的通訊，需要大量的學習成本。

# 番外篇 Steam Deck 運行qcow2 image

上期我們介紹了如何使用qemu快速跑起三個不同的qcow2，如果你像筆者一樣，相用Steam Deck使測試，它沒有自帶的qemu，那又該如何呢？

## Distrobox 幫到你

因為qemu是一個CPU的程序，理論上只要有container，就可以跑起來。因為Steam Deck自帶Distrobox和podman，用它來起個ubuntu的container，就可以下載qemu。

```bash
distrobox create --name ubuntu --image ubuntu:24.04
distrobox enter ubuntu
sudo apt-get update && sudo apt-get install qemu-system-x86
```

筆者親測，可行，只是速度有點慢，應該是kvm加速沒有生效。

## Boxes 幫到你

除了Distrobox，Steam Deck上還有一個神器Boxes，不過它要經flatpak（Discovery）安裝。

```bash
flatpak install org.gnome.Boxes
```

Boxes 可以直接運行 qcow2 格式的VM。OpenEuler ，AnolisOS 都順利運行。而阿里aliyun，筆者就未有詳細測試，道理上只要經過第一次cloud-init改寫的qcow2檔，搬來Boxes就可以用。

## 為何 Distrobox + Qemu 可以運行，還是用Boxes?

因為Qemu還是有一些學習成本，而Boxes就有齊snapshot等功能，也可以隨時加大hdd，學習成本較低。當然，steam deck 上的網絡限制還是一樣多，Steam OS有很多定制的環境，想不破壞那些規則而安裝插件，是不可行的。