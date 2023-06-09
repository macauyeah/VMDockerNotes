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

In the Dockerfile, I choose to use ***ENTRYPOINT*** and in ***exec*** form, because I want to append mdbook argument through ***docker run***

For example, passing ***--version*** to container, so that mdbook could get the argument.
```bash
sudo docker run --rm -it -v $(pwd):/opt/mdbookdata mdbook:beta --version
```

But in the base docker image, it run as root. If you build a book by image, the ***book/*** output folder's ownership will be set to root. You could change the default ownership by add ```--user $(id -u):$(id -g)```

```bash
sudo docker run --user $(id -u):$(id -g) --rm -it -v $(pwd):/opt/mdbookdata mdbook:beta
```

And also, if you want to make an cammand alias in your bash shell, so that you no need to run a long command very time, you could add a line in ```~/.bashrc```.

```bash
# edit ~/.bashrc
alias mdbook='sudo docker run --user $(id -u):$(id -g) --rm -it -v $(pwd):/opt/mdbookdata mdbook:beta $1'
```

Then you can call it from your host by
```bash
# let .bashrc take effect without logout login
source ~/.bashrc
# run command through alias
mdbook --version
# it support multiple arguments
mdbook build --dest-dir ./dest
```