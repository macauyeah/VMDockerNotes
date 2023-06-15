# 把command line 的程式打包成Docker Image

平時我們用別人的Docker image，都以網絡服務為主，也就是，他們的image通常以常駐，打開某個網絡端口(UDP/TCP Port)提供服務。

但像nodejs, java等，其實是一些command line程式。它們其實可以一次性地執行後，回傳結果。

這次我們就以一個mdbook程式為例子，介紹打包的流程。

第一件事，在Docker image中，安裝或編譯binary
```
# Dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install cargo -y && cargo install mdbook

ENV PATH="${PATH}:/root/.cargo/bin"

VOLUME /opt/mdbookdata
WORKDIR /opt/mdbookdata
ENTRYPOINT ["mdbook"]
```

打包成image，給他一個名字，例如mdbook:beta
```bash
sudo docker build -t mdbook:beta ./
```

在Dockerfile中，我選擇使用 ***ENTRYPOINT*** 的 ***exec*** 格式，因為我需要後期在 ***docker run*** 動態加入參數。

例如，要把參數 ***--version*** 傳入container，讓mdbook就可以看到該參數。
```bash
sudo docker run --rm -it -v $(pwd):/opt/mdbookdata mdbook:beta --version
```

但在docker image中，它是使用root來執行。如果你直接執行build，那麼它的輸出資料夾 ***book/*** 的擁有人就會變成了root。你要用指定 ***--user $(id -u):$(id -g)***來指定執行者。
```bash
sudo docker run --user $(id -u):$(id -g) --rm -it -v $(pwd):/opt/mdbookdata mdbook:beta
```

還有，如果不想每次都打這麼長的指令，你可以在 ***~/.bashrc*** 中做別名(alias)。

```bash
# edit ~/.bashrc
alias mdbook='sudo docker run --user $(id -u):$(id -g) --rm -it -v $(pwd):/opt/mdbookdata mdbook:beta $1'
```

這樣就可以簡化執行指令。
```bash
# let .bashrc take effect without logout login
source ~/.bashrc
# run command through alias
mdbook --version
# it support multiple arguments
mdbook build --dest-dir ./dest
```