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

## install pasta
似乎在 podman 升級到 5.0.x 之後，需要一個名為 "pasta" 的依賴套件。這個套件要麼透過 pacman 安裝或從原始碼進行編譯。但為了避免 Steam OS 在更新時會破壞套件，我選擇了直接下載了它的二進位檔案。（我嘗試從原始碼編譯，但許多C 函式庫在 Steam OS 上無法取得的，最後只能直接下載官方的二進位檔。）

It seems after upgrading podman to 5.0.x, podman need a dependency "pasta". Either install by pacman or build from source. In-case steam os will break the package later, I download the binary directly. (I tried to build from source but a lot of c library missing from steam os. That's why I download the binary direclty form official website)

```bash
curl https://passt.top/builds/latest/x86_64/pasta -o pasta
chmod u+x pasta
# update PATH variable in current session or add in ~/.bashrc
export PATH=$PATH:$(pwd)
```

## change oci runtime
在添加了 pasta 之後，運行 podman 5.0.x 仍然失敗。podman 會抱怨 'crun' 的版本不正確。這是由於機器上存在兩個版本的 'crun'，一個來自 Steam OS 的 /usr/bin/crun，一個來自 brew。

After adding pasta, running podman 5.0.x still fail. podman complaint 'crun' version is wrong. Becuase two verions of 'crun' in the machine, one from steam os /usr/bin/crun, one from brew.
```bash
podman run --name basic_httpd -dt -p 8080:80/tcp docker.io/nginx
# Error: OCI runtime error: crun: unknown version specified
```

為解決這個問題，我們需要修改 podman 的設定，讓它的 oci runtime指向 brew 資料夾。（或者任何一個你行找回來的版本，版本最底要求為1.14.x以上，crun於brew 的版本為1.15)

To solove the problem, need to config podman oci runtime to brew folder. (or any crun path with version higher than 1.14.x, version of crun in brew is 1.15)

```bash
mkdir -p ~/.config/containers/
cp /usr/share/containers/containers.conf ~/.config/containers/
```

如下，在```[engine.runtimes]```底下加入 brew crun path

add brew crun path to section ```[engine.runtimes]``` like below
```conf
# vim ~/.config/containers/containers.conf
[engine.runtimes]
crun = [
  "/home/linuxbrew/.linuxbrew/bin/crun"
]
```
