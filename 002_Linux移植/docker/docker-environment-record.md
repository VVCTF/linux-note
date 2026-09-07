# Docker Environment Record

Updated: 2026-09-02

## Purpose

This WSL2 environment builds the i.MX6ULL Linux 4.1.15 kernel in a fixed Ubuntu 18.04 Docker image, avoiding host GCC and glibc version differences.

## Docker Status

- Docker Engine: 29.7.2
- Docker service: enabled and active
- WSL systemd: enabled
- Docker test: `docker run --rm hello-world` completed successfully.

Useful status commands:

```bash
systemctl is-active docker
docker version
docker image ls
docker ps -a
```

## Docker Engine Package Source

Docker Engine was installed from the Aliyun Docker CE repository:

```text
/etc/apt/sources.list.d/docker.list
```

Current repository:

```text
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] http://mirrors.aliyun.com/docker-ce/linux/ubuntu noble stable
```

Repository signing key:

```text
/etc/apt/keyrings/docker.asc
```

## Registry Mirrors

Docker daemon configuration:

```text
/etc/docker/daemon.json
```

Current configuration:

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live",
    "https://hub.rat.dev"
  ]
}
```

After changing this file, restart Docker:

```bash
sudo systemctl restart docker
```

## Zscaler Certificates

The network performs TLS interception with Zscaler. Docker image downloads require the following trusted certificates.

Certificate source files managed by the system:

```text
/usr/local/share/ca-certificates/zscaler-root.crt
/usr/local/share/ca-certificates/zscalerthree-root.crt
```

- `zscaler-root.crt`: Zscaler Root CA
- `zscalerthree-root.crt`: Zscaler Intermediate Root CA (zscalerthree.net) (t)

The second certificate is required by the current network TLS chain and fixed Docker image pull failures such as:

```text
x509: certificate signed by unknown authority
```

The original manually exported certificate remains at:

```text
/home/ctf/zscaler-root.cer
```

After adding or replacing a certificate:

```bash
sudo update-ca-certificates
sudo systemctl restart docker
```

The generated certificate links and combined trust bundle are under:

```text
/etc/ssl/certs/
```

## Kernel Build Image

Project Dockerfile:

```text
/home/ctf/linux/imx6ull/kernel/imx-kernel-4.1.15/Dockerfile
```

The image uses Ubuntu 18.04 as the host build environment and installs common kernel build dependencies. Ubuntu 18.04 is end-of-life, so the Dockerfile uses `old-releases.ubuntu.com` for apt packages.

Build the image:

```bash
cd ~/linux/imx6ull/kernel/imx-kernel-4.1.15
docker build -t imx6ull-kernel-builder:ubuntu18.04 .
```

Run an interactive container. The source tree is mounted read-write so build outputs remain on the host; the cross compiler directory is mounted read-only:

```bash
docker run --rm -it \
  -v "$PWD":/workspace \
  -v "$HOME/tools/cross_compiler":/root/tools/cross_compiler:ro \
  -w /workspace \
  imx6ull-kernel-builder:ubuntu18.04 \
  bash
```

Inside the container:

```bash
source env_setup.sh
make imx6ull_alientek_emmc_defconfig
make -j"$(nproc)"
```

## Notes

- The WSL host is Ubuntu 26.04. It supplies only the Docker daemon; the container supplies the older Ubuntu 18.04 host tool environment.
- The ARM cross compiler remains the existing Linaro GCC 4.9.4 toolchain selected by `env_setup.sh`.
- Do not remove the Zscaler CA certificates while this network performs TLS interception, or Docker image pulls may fail again.
