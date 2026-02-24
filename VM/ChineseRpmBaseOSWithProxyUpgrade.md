# Setup Squid proxy for Chinese RPM base Linux repo update

## ubuntu 24.04 squid settings

```bash
sudo apt-get update && sudo apt-get install squid
sudo vim /etc/squid/squid.conf
```

around line 1404, add

```bash
acl fixip src xxx.xxx.xxx.xxx
```

around line 1627, add 

```bash
http_access allow fixip
```

## rpm os settings
For example

Anolis OS 8, `/etc/yum.repos.d/AnolisOS-AppStream.repo`, `/etc/yum.repos.d/AnolisOS-BaseOS.repo`, etc, add `proxy=http://139.162.29.14:3128` at the end of each file
```
[AppStream]
name=AnolisOS-$releasever - AppStream
baseurl=http://mirrors.openanolis.cn/anolis/$releasever/AppStream/$basearch/os
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ANOLIS
gpgcheck=1
proxy=http://139.162.29.14:3128
```

```
[BaseOS]
name=AnolisOS-$releasever - BaseOS
baseurl=http://mirrors.openanolis.cn/anolis/$releasever/BaseOS/$basearch/os
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ANOLIS
gpgcheck=1
proxy=http://139.162.29.14:3128
```

OpenEuler 22, `/etc/yum.repos.d/openEuler.repo`, single file with all repo location, add `proxy=http://139.162.29.14:3128` at each end of repo config
```
[OS]
name=OS
baseurl=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/OS/$basearch/
metalink=https://mirrors.openeuler.org/metalink?repo=$releasever/OS&arch=$basearch
metadata_expire=1h
enabled=1
gpgcheck=1
gpgkey=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/OS/$basearch/RPM-GPG-KEY-openEuler
proxy=http://139.162.29.14:3128

[everything]
name=everything
baseurl=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/everything/$basearch/
metalink=https://mirrors.openeuler.org/metalink?repo=$releasever/everything&arch=$basearch
metadata_expire=1h
enabled=1
gpgcheck=1
gpgkey=http://repo.openeuler.org/openEuler-22.03-LTS-SP4/everything/$basearch/RPM-GPG-KEY-openEuler
proxy=http://139.162.29.14:3128
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
proxy=http://139.162.29.14:3128
...
...
```

## command
trouble shoot command

```bash
curl -v -x http://139.162.29.14:3128 http://mirrors.openanolis.cn/anolis/8.10/
```

partial upgrade command
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