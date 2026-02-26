# 架設 Squid proxy，作為國産 Linux RPM 安裝包更新的 RPM proxy
前編我們介紹了 [Qemu 運行國産OS做快速測試](ChineseRpmBaseOSWithProxyUpgrade.md)。應該基本使用大家都可以實驗到。

在投産環境上，我們通常還要控制它的kernel或lib版本更新，但這麼多的不同OS版本，想一次過做rpm mirror，並不太實際。

若以監管為目標，那些不同種類的OS，限制互聯網存取，統一經過某個http proxy的取得RPM更新，應該是一個最低成本的做法。

本文就來介紹一下，使用Ubuntu 24建設 Squid http proxy，達到rpm proxy的結果。

## ubuntu 24.04 squid settings

```bash
sudo apt-get update && sudo apt-get install squid
sudo vim /etc/squid/squid.conf
```

約在1404行，指定一個新的acl(access control list)名字，`fixip`, 它的允許來源IP是你的rpm base OS

```bash
# around line 1404, add
acl fixip src xxx.xxx.xxx.xxx
```

約在1627行, 放行新的acl
```bash
# around line 1627, add 
http_access allow fixip
```

## rpm os settings
在rpm base的OS上，通常在 `/etc/yum.repos.d/` 低下就找到它們的 rpm 包來源為置，在每個來源上加上 proxy 設定，就可以了。

### Anolis OS 8
因為rpm來源眾多，我們只想讓其中兩個經proxy更新，例如
`/etc/yum.repos.d/AnolisOS-AppStream.repo`, `/etc/yum.repos.d/AnolisOS-BaseOS.repo`, 最在後加入 `proxy=http://yyy.yyy.yyy.yyy:3128`。 yyy.yyy.yyy.yyy 就是設了 Squid的機器
```
[AppStream]
name=AnolisOS-$releasever - AppStream
baseurl=http://mirrors.openanolis.cn/anolis/$releasever/AppStream/$basearch/os
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ANOLIS
gpgcheck=1
proxy=http://yyy.yyy.yyy.yyy:3128
```

```
[BaseOS]
name=AnolisOS-$releasever - BaseOS
baseurl=http://mirrors.openanolis.cn/anolis/$releasever/BaseOS/$basearch/os
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ANOLIS
gpgcheck=1
proxy=http://yyy.yyy.yyy.yyy:3128
```

### OpenEuler 22
來源檔只有一個，`/etc/yum.repos.d/openEuler.repo`, 但內存多個section, 需要在每個section的尾段，加入 `proxy=http://yyy.yyy.yyy.yyy:3128`
```
[OS]
name=OS
baseurl=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/OS/$basearch/
metalink=https://mirrors.openeuler.org/metalink?repo=$releasever/OS&arch=$basearch
metadata_expire=1h
enabled=1
gpgcheck=1
gpgkey=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/OS/$basearch/RPM-GPG-KEY-openEuler
proxy=http://yyy.yyy.yyy.yyy:3128

[everything]
name=everything
baseurl=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/everything/$basearch/
metalink=https://mirrors.openeuler.org/metalink?repo=$releasever/everything&arch=$basearch
metadata_expire=1h
enabled=1
gpgcheck=1
gpgkey=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/everything/$basearch/RPM-GPG-KEY-openEuler
proxy=http://yyy.yyy.yyy.yyy:3128
...
...

[update]
name=update
baseurl=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/update/$basearch/
metalink=https://mirrors.openeuler.org/metalink?repo=$releasever/update&arch=$basearch
metadata_expire=1h
enabled=1
gpgcheck=1
gpgkey=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/OS/$basearch/RPM-GPG-KEY-openEuler
proxy=http://yyy.yyy.yyy.yyy:3128
...
...
```

## 指令
我們可以先用curl，來測試一下最基本的連線。請確保指令是在最初定義的`xxx.xxx.xxx.xxx`範圍內。

```bash
curl -v -x http://yyy.yyy.yyy.yyy:3128 http://mirrors.openanolis.cn/anolis/8.10/
```

### 部份更新指令
由於我們前述 rpm 包並不是所有都加了proxy，我們只限定某些進行更新，所以我們使用`--disablerepo --enablerepo`來限制指定的更新來源。
```bash
# anolis
sudo dnf install --disablerepo='*' --enablerepo='BaseOS' 'tmux'
sudo dnf upgrade --disablerepo='*' --enablerepo='kernel-5.10' 'kernel*'

# openeuler
dnf install --disablerepo='*' --enablerepo='everything' tmux
dnf upgrade --disablerepo='*' --enablerepo='update' 'kernel*'
```

## external ref
- squid tutorial [https://www.digitalocean.com/community/tutorials/how-to-set-up-squid-proxy-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-squid-proxy-on-ubuntu-20-04)
- yum proxy tutorial [https://www.baeldung.com/linux/yum-dnf-repositories-set-proxy](https://www.baeldung.com/linux/yum-dnf-repositories-set-proxy)