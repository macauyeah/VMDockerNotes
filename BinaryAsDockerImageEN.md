# Encapsulate Command Line Binary as Docker Image

Usually in docker images, they mainly focus on providing network service. Their image will keep an alive process in some specific port to provide service.

But some other images like nodejs, java, they are an time only binary. They can run and return result form command line output.

In this article, we will build an image to provide mdbook binary.

First thing first, in docker image to install or compile the binary.
```
# Dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install cargo -y && cargo install mdbook

ENV PATH="${PATH}:/root/.cargo/bin"

VOLUME /opt/mdbookdata
WORKDIR /opt/mdbookdata
ENTRYPOINT ["mdbook"]
```

Build image, give it a tag name, mdbook:beta
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
sudo docker run --user $(id -u):$(id -g) --rm -it -v $(pwd):/opt/mdbookdata mdbook:beta build
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