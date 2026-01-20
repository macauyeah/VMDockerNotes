https://github.com/qemus/qemu-docker

https://github.com/qemus/qemu-arm/

git clone

docker compose -f compose.yml up -d

visit 8006

os must support kvm

# in steam deck
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install podman-compose
podman image pull docker.io/qemux/qemu
podman-compose -f compose.yml up -d
```

```yml
services:
  qemu:
    image: qemux/qemu
    container_name: qemu
    environment:
      BOOT: "mint"
    devices:
      - /dev/kvm
      - /dev/net/tun
    cap_add:
      - NET_ADMIN
    ports:
      - 8006:8006
    volumes:
      - ./openEuler-24.03-LTS-SP3-x86_64.qcow2:/boot.qcow2
    restart: always
    stop_grace_period: 2m
```
