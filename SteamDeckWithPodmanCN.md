# Steam Deck With Podman

眾所週知，Steam Deck預裝的是一台Linux主機。但它的系統比較特別，為了可以安全更新，所以系統最主要的部份都設定為唯讀(read only)。也就是，傳統你可以直接在Linux上經管理員權限安裝的軟件包，全部都會被擋，即使你把唯讀部份設為可讀寫(read / write)，在下次更新時，都會被一次過覆蓋掉。

筆者作為一個負責任的機迷+開發者，怎樣可以白白讓一台Linux機只可以玩遊戲呢? (怎樣跟老婆交代呢?)

所以筆者千辛萬苦，找到一個折衷方案，讓他可以當為開發機使用，那就是Podman。(當然，若果大家有條件有金錢，直接改裝Windows就可以了。)

# Podman是什麼?
Podman跟Docker一樣，都是一些管理和運行Container的主程式。跟Docker不一樣的是，它是Open source，而且是daemonless。

所謂的daemonless，就是不會有一個背景程式去長期管理Container。好處是不會因為背景程式死了，就全部Container一起掛掉，預設也不需要走管理員權限路線。但也因此跟Docker有一些使用上的差異，例如Podman沒有原生的docker-compose結構，即使坊間有python寫的podman-compose去硬對應docker-compose，但某些network是跟結構還是不能直接從Docker轉移過來。

就筆者早期的踩雷經驗而言，用Podman跑起一兩個獨立固定Port的Container來說，都很夠用，也不會遇到奇怪的Bug。所以這次，亦用來作為Steam Deck運行整合式開發的Container。

# 不平凡的安裝之路
## install homebrew
Steam OS 3，雖然可以使用更改read / write，再使用pacman來安裝podman。但因為Steam OS更新後，全部要重來，工作量和網路流量都不少，所以筆者改為使用homebrew來安裝podman。homebrew只需要首次安裝時使用管理員權限，之後就會在/home資料夾下留下可執行的程式，所以它不會被Steam OS更新所破壞。

```bash
# brew install script, need sudo password
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"  # with sudo password

# brew post install script, no sudo required
(echo; echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"') >> /home/deck/.bash_profile
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```


## install podman
```bash
brew install podman

# running rootless mapping for podman container
# it might also affect how it download oci image
sudo touch /etc/subuid /etc/subgid
sudo usermod --add-subuid 100000-165535 --add-subgid 100000-165535 $USER
```

記得記得重新開機，之後應該就可以成功運行container
```bash
podman container run --name ubuntu2204 -it ubuntu:22.04 bash
```