# Steam Deck With Podman
眾所週知，Steam Deck預裝的是一台Linux主機。但它的系統比較特別，為了可以安全更新，所以系統最主要的部份都設定為唯讀(read only)。也就是，傳統你可以直接在Linux上經管理員權限安裝的軟件包，全部都會被擋，即使你把唯讀部份設為可讀寫(read / write)，在下次更新時，都會被一次過覆蓋掉。

筆者作為一個負責任的機迷+開發者，怎樣可以白白讓一台Linux機只可以玩遊戲呢? (怎樣跟老婆交代呢?)

所以筆者千辛萬苦，找到一個折衷方案，讓他可以當為開發機使用，那就是Podman。(當然，若果大家有條件有金錢，直接改裝Windows就可以了。)

# Podman是什麼?
Podman跟Docker一樣，都是一些管理和運行Container的主程式。跟Docker不一樣的是，它是Open source，而且是daemonless。

所謂的daemonless，就是不會有一個背景程式去長期管理Container。好處是不會因為背景程式死了，就全部Container一起掛掉，預設也不需要走管理員權限路線。但也因此跟Docker有一些使用上的差異，例如Podman沒有原生的docker-compose結構，即使坊間有python寫的podman-compose去硬對應docker-compose，但某些network是跟結構還是不能直接從Docker轉移過來。

就筆者早期的踩雷經驗而言，用Podman跑起一兩個獨立固定Port的Container來說，都很夠用，也不會遇到奇怪的Bug。所以這次，亦用來作為Steam Deck運行整合式開發的Container。

# Steam OS 3.5 後
在Steam OS 3.5之後，已經有預安裝的 podman，筆者建議，就直接使用預安裝版本就好。如果一定要自行安裝，請參閱後述的 【Steam OS 3.5前，不平凡的安裝之路】。

## build in podman
Steam OS 3.5，雖然已經有預安裝 podman ，但在實際環境下，多安裝一個 podman-compose 可以更方便地一體化操作。 我們可以經 python 安裝。

```bash
# install podman-compose by python package manager pip
python -m ensurepip --upgrade
python -m pip install podman-compose
```

剛安裝 podman-compose ，會出現在自己的 home 目標的隱藏目錄。最後一步就是要加到自己的 PATH 環境變數裏面。我們如果源用 bash 設定，就直接修改 `~/.bash_profile` 。
```bash
# edit .bash_profile, add following command
export PATH=$PATH:~/.local/bin/

# then reboot / source .bash_profile
```
修改保存後，就重啟。之後 podman-compose 的指令就可以任意存取了。

要補充一點，就是官方預安裝的 podman 還是缺少了一些 DNS 的元件，大家會看到 warning 提示。`WARN[0002] aardvark-dns binary not found, container dns will not be enabled`。該問題筆者認為主要是 podman rootless 的網路模式有很多限制，有很多功能都不是預設可以使用，要麽自行安裝 plugin 。內部網路是正常的，但唯獨DNS沒法解釋，所以如果大家只做 ip 測試是沒有問題。但若然想要互聯網服務，就必需要自己進行DNS解釋。例如你可能要這樣

```bash
curl https://github.com --resolve 'github.com:443:140.82.116.4'
```

這個解法最麻煩的是要為每個 service 內部重新安裝網絡DNS，就有點像每個 service 的內部環境都要很熟悉才能解，所以這並不是一個十分有效的做法。

我們還有另一種解法，就是 service 設定檔中將 network_mode 設為 host ，即是直接使用主機上的網絡。然後各個 service 之間都直接使用 localhost 溝通，它們之間的 port 也是互相看到的，而且連接互聯網時，DNS 也可以正常運作。但這個解法的缺點就同一個 port 不能有多個 service 使用，也就是不可能簡單地進入群集模式。不過我們現在主要用於開發環境，所以這並不會是很太大的問題。

## build in distrobox
在 Steam OS 3.5 中，除了 podman 外，還有預裝 distrobox 。 distrobox 其實是基於 container 技術的擴展應用，它目標是讓用經過 container 就可以輕鬆使用到不同 linux 的發佈版本。例如我想在 Steam OS 中使用 Ubuntu ，經過 distrobox 就可以用到。道理上， distrobox 基於 container (podman) 操作的，所以它能做到的，其實自己手動經 podman 也是可以做到。但若果大家想使用跨 Linux 版本的 GUI 程式，筆者還是建議優先使用 distrobox 。因為 distrobox 預設已為不同版本的 Linux 的 Image (來源影像檔) 加入部份調整，在運行時亦有x11等互通，指令也較為簡單。

以下做來例子，示範在 Steam OS 中就執行 Ubuntu 版本的 vscode。
```bash
xhost +si:localuser:$USER # enable x11 for desk user

distrobox create --image ubuntu:24.04 --name ubuntu
distrobox enter ubuntu

# install vscode inside distrobox ubuntu
sudo apt-get install wget gpg
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" |sudo tee /etc/apt/sources.list.d/vscode.list > /dev/null
rm -f packages.microsoft.gpg

sudo apt install apt-transport-https
sudo apt update
sudo apt install code # or code-insiders

# start vscode inside distrobox ubuntu
code

# delete distrobox ubuntu
distrobox stop ubuntu
distrobox rm ubuntu
```

註: Distrobox 也不是萬能的，例如它的 Ubuntu 版本內沒有 snap ，所以不能執行 Ubuntu 版本的 Firefox。
snap will not works (firefox not works)


# Steam OS 3.5前，不平凡的安裝之路
以下內容寫於2023年10月，當時版本為 Steam OS 3.4.x。 2023年11月之後 Steam OS 3.5.x , 3.6.x 推出後，筆者還有使用一段時間。直至2024年10月，才正式砍掉重練。以下安裝步聚，大家請留意是否有需要微調。

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
# chmod a+x pasta, if pasta's owned by root. let it share exec permission for everyone.

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


