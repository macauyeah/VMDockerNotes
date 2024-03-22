# Docker Variable control
我們在Docker Image的打包時，最簡單當然就是每個步驟都使用最新版本。例如Docker Base Image，大家可能選用latest tag，安裝linux package （Linux包）,也可能就apt install / yum 安裝最新的穩定版本。但如果我們想要更好地做測試，就要使用指定版本，方便追蹤問題。而Docker在打包和運行時，都有不同的方式讓大家定義或覆寫指定參數。

## Docker build arg
我們先從打包Image開始。

例如我們需要使用一個Base image為 ubuntu，版本預設為22.04，但有需要時可以經build指令覆寫，可以這樣寫

```Dockerfile
ARG ubuntu_version=22.04
FROM ubuntu:${ubuntu_version}
```

```bash
# default ubuntu_version=22.04
docker image build -t test2204 ./
# or overwrite by --build-arg
docker image build -t test2404 --build-arg="ubuntu_version=24.04"
```

雖然Dockerfile的RUN指令都是使用linux shell，但在Dockerfile中想表達條件控制（if else statment）就不太易看。在外部加入script做控制，是另一個可行的後備選擇，它更可以連image名字也進行參數化。

```bash
# in bash script, you also can
if [ $beta == true ]
then
    ubuntu_version=24.04
else
    ubuntu_version=22.04
fi

docker image build -t test:${ubuntu_version} --build-arg ubuntu_version=${ubuntu_version}
```

## Docker Container Run and Docker Compose
一般來講，Linux Container 在執行時，就等於進入Linux Shell。也就是，我們可以使用Shell中的環境變數。

我們在打包Image前，已經可以在Dockerfile中定義自己的ENV數參（也就是環境變數）。與前面的Build Arg有所不同的是，ENV是定義在Dockerfile中，在Container運行時以環境變數的形式存在，它也可以在運行中被改變。而Arg，則只在打包Image時存在，運行期間就不存在了。（當然，你在打包時，用Arg傳入Env，以運到這個目的。）

另一個更特別的性質是，那怕ENV沒有定義在Dockerfile中，我們運行時也可以加入更多的環境變數，大家就當成是一般Linux操作，隨時在自己的shell中加入變數。

```bash
# -e, --env for inline variable
# --env-file for file
docker container run -e MYVAR1 --env MYVAR2=foo --env-file ./env.list ubuntu bash
```

同樣地Docker compose，也支援環境變數。筆者建議environment可以使用Array格式，日後可以更方便地直接改為env_file。
```yaml
# docker-compose.yaml
services:
  ubuntu:
    image: ubuntu:22.04
    environment:
      - RACK_ENV=development
      - SHOW=true
      - USER_INPUT
```

上述的寫法沒有任何問題，不過如果你的docker-compose.yaml是放在git等版本控制中，你更新環境變數就有可能會影響到其他人，這時你就會想轉成env_file。

docker-compose.yaml預設就會讀當前資料夾的.env，就算不存在，也可以正常運行。（當然，大家的Image/Container應該要有預設值）

```yaml
# docker-compose.yaml
services:
  ubuntu:
    image: ubuntu:22.04
    # if env_file is not defined, it will load .env.
    # or you can load the specific file.
    # env_file:
    #   - ./a.env 
```

env_file內，每一行就是一個變數
```
# .env or a.env
RACK_ENV=development
SHOW=true
USER_INPUT
```

使用預設的.env還有一個好處，就是我們可以把docker-compose.yaml也變成受環境變數控制。

```yaml
# docker-compose.yaml with variable control, only works in default .env
services:
  ubuntu:
    image: ubuntu:${ubuntu_version}
```

```
# .env
ubuntu_version=22.04
```