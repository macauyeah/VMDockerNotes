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


