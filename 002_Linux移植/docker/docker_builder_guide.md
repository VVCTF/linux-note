# i.MX6ULL 内核 Docker 构建指南

本文记录在 Docker Engine 已安装并验证可用的前提下，如何使用 Docker 构建 i.MX6ULL Linux 4.1.15 内核。

该构建镜像将主机构建环境固定为 Ubuntu 18.04，包括对应的 host GCC 和 glibc；镜像中还包含本项目使用的 Linaro ARM 交叉编译器。因此，宿主机 Ubuntu 的版本不会影响内核构建环境。

## 1. 基本概念

本项目主要会接触 Docker 的三种对象：

- **Dockerfile**：描述如何制作镜像的文本配方。
- **镜像（image）**：可重复使用的只读构建环境。本项目的镜像名为 `imx6ull-kernel-builder:ubuntu18.04`。
- **容器（container）**：由镜像创建出的实际运行实例。执行 `make` 等命令的地方就是容器。

三者关系如下：

```text
Dockerfile --docker build--> 镜像 --docker run--> 容器
```

内核源码不会被复制进镜像。实际构建时，将宿主机源码目录挂载到容器中，因此编译生成的文件仍保存在本地源码树。

## 2. 项目文件与固定工具版本

下列文件和目录参与构建环境：

| 文件或目录 | 作用 |
| --- | --- |
| `Dockerfile` | 定义 Ubuntu 18.04、host 编译依赖，以及镜像内的交叉编译器路径。 |
| `toolchains/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/` | 构建镜像时复制到镜像内部的 ARM 交叉编译器。 |
| `env_setup.sh` | 设置 `ARCH=arm` 和 `CROSS_COMPILE=arm-linux-gnueabihf-`。 |
| `imx6ull_alientek_emmc.sh` | 项目原有编译辅助脚本，其中使用 `imx_alientek_emmc_defconfig` 配置目标。 |

最终镜像包含：

```text
Ubuntu 18.04 host 构建环境
Host GCC、make、bison、flex、OpenSSL、ncurses、bc、dtc 等依赖
Linaro ARM GNU Toolchain 4.9.4-2017.01
```

交叉编译器在镜像内的路径为：

```text
/root/tools/cross_compiler/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin
```

该目录已经加入镜像的 `PATH`。但 `env_setup.sh` 仍然需要使用，因为它会为内核 Makefile 设置 ARM 架构和交叉编译器前缀。

## 3. 构建编译环境镜像

进入内核源码目录后执行：

```bash
cd ~/linux/imx6ull/kernel/imx-kernel-4.1.15
docker build -t imx6ull-kernel-builder:ubuntu18.04 .
```

命令说明：

| 部分 | 含义 |
| --- | --- |
| `docker build` | 根据 Dockerfile 构建镜像。 |
| `-t imx6ull-kernel-builder:ubuntu18.04` | 给构建结果指定镜像名和标签。镜像名是 `imx6ull-kernel-builder`，标签是 `ubuntu18.04`。 |
| `.` | 将当前目录作为构建上下文。Docker 从这里读取 Dockerfile，也只能复制该目录下的文件，包括 `toolchains/`。 |

第一次构建会下载 Ubuntu 基础镜像并安装依赖。后续构建会复用未变化的缓存层。若修改了 Dockerfile 或 `toolchains/` 中的工具链，需要用相同命令重新构建；同一个 tag 会自动指向新镜像。

查看本地镜像：

```bash
docker image ls
```

无需进入容器，直接验证镜像中的工具：

```bash
docker run --rm imx6ull-kernel-builder:ubuntu18.04 \
  bash -c 'gcc --version | head -n1; bison --version | head -n1; arm-linux-gnueabihf-gcc --version | head -n1'
```

预期结果：host GCC 和 Bison 显示 Ubuntu 18.04 中的版本，交叉编译器显示 Linaro GCC 4.9.4。

## 4. 在容器中编译内核

在内核源码目录中启动交互式构建容器：

```bash
cd ~/linux/imx6ull/kernel/imx-kernel-4.1.15

docker run --rm -it \
  --name imx6ull-kernel-build \
  -v "$PWD":/workspace \
  -w /workspace \
  imx6ull-kernel-builder:ubuntu18.04 \
  bash
```

命令说明：

| 部分 | 含义 |
| --- | --- |
| `docker run` | 基于镜像创建并启动容器。 |
| `--rm` | 执行 `exit` 后自动删除容器。镜像和源码文件不会删除。 |
| `-i` | 保持标准输入，允许交互输入命令。 |
| `-t` | 分配终端，使 shell 正常交互。 |
| `--name imx6ull-kernel-build` | 给临时容器指定易读名称。 |
| `-v "$PWD":/workspace` | 将当前宿主机源码目录以读写方式挂载到容器 `/workspace`。 |
| `-w /workspace` | 设置容器启动后的默认工作目录。 |
| `imx6ull-kernel-builder:ubuntu18.04` | 要运行的镜像。 |
| `bash` | 容器内启动的程序。 |

进入容器后，先加载架构和交叉编译器配置：

```bash
source env_setup.sh
```

验证当前环境：

```bash
echo "$ARCH"
echo "$CROSS_COMPILE"
arm-linux-gnueabihf-gcc --version
```

使用项目 defconfig 并编译：

```bash
make imx_alientek_emmc_defconfig
make -j"$(nproc)"
```

`$(nproc)` 会替换为容器可见的 CPU 核数。若想明确限制并行数，可以写为 `make -j8`。

进行干净的重新编译前，执行：

```bash
make distclean
```

交互式修改内核配置：

```bash
make menuconfig
```

完成后退出容器：

```bash
exit
```

由于源码目录是挂载的，即使容器自动删除，所有内核输出文件仍保留在宿主机源码目录中。

## 5. 不进入容器，直接执行完整构建

交互式容器便于学习和排错。日常重复构建可以用一条命令完成：

```bash
cd ~/linux/imx6ull/kernel/imx-kernel-4.1.15

docker run --rm \
  -v "$PWD":/workspace \
  -w /workspace \
  imx6ull-kernel-builder:ubuntu18.04 \
  bash -lc 'source env_setup.sh && make imx_alientek_emmc_defconfig && make -j"$(nproc)"'
```

`bash -lc` 启动 Bash 命令 shell 并执行单引号中的命令。`&&` 表示前一步成功后才会继续下一步。

## 6. Docker 常用命令

### 6.1 镜像命令

拉取公共镜像：

```bash
docker pull ubuntu:18.04
```

`docker pull` 将镜像及其分层下载到本地镜像缓存中。

列出本地镜像：

```bash
docker image ls
```

列出全部镜像，包括没有标签的中间镜像：

```bash
docker image ls -a
```

查看镜像详细元数据：

```bash
docker image inspect imx6ull-kernel-builder:ubuntu18.04
```

删除一个镜像：

```bash
docker image rm imx6ull-kernel-builder:ubuntu18.04
```

如果存在由该镜像创建的容器，Docker 不允许删除镜像。先删除相关容器，或者使用其他镜像标签。

仅清理无标签的悬空镜像：

```bash
docker image prune
```

Docker 会要求确认。多次使用同一个 tag 重新构建后，这个命令可清理旧镜像。

### 6.2 容器命令

查看正在运行的容器：

```bash
docker ps
```

查看全部容器，包括已停止容器：

```bash
docker ps -a
```

创建并立即进入一个会保留的容器：

```bash
docker run -it --name ubuntu1804-test ubuntu:18.04 bash
```

这个命令适合实验。若希望容器在 `exit` 后仍保留，不要加 `--rm`。

启动并重新进入已停止的容器：

```bash
docker start -ai ubuntu1804-test
```

| 参数 | 含义 |
| --- | --- |
| `-a` | 将容器的标准输出和错误输出连接到当前终端。 |
| `-i` | 保持标准输入可用。 |

在一个已经运行的容器中执行 Bash：

```bash
docker exec -it <容器名或容器ID> bash
```

`docker exec` 不会创建新容器，而是在已有的运行容器中启动一个新进程。

停止运行中的容器：

```bash
docker stop <容器名或容器ID>
```

删除已停止的容器：

```bash
docker rm <容器名或容器ID>
```

删除全部已停止容器：

```bash
docker container prune
```

### 6.3 日志与诊断

查看容器日志：

```bash
docker logs <容器名或容器ID>
```

持续跟踪新日志：

```bash
docker logs -f <容器名或容器ID>
```

查看 Docker Engine 客户端和服务端版本：

```bash
docker version
```

查看镜像、容器、数据卷和构建缓存占用的磁盘空间：

```bash
docker system df
```

查看 Docker daemon 信息，包括当前配置的镜像加速器：

```bash
docker info
```

在 WSL 中检查 Docker 服务：

```bash
systemctl status docker --no-pager
systemctl is-active docker
```

修改 `/etc/docker/daemon.json` 或证书后，重启 Docker：

```bash
sudo systemctl restart docker
```

### 6.4 清理命令

清理已停止容器、未使用网络、未使用镜像和未使用构建缓存：

```bash
docker system prune
```

连同未被使用的镜像一起清理：

```bash
docker system prune -a
```

请谨慎使用 `docker system prune -a`。它不会删除被已有容器使用的镜像，但可能删除已下载的基础镜像或构建镜像，之后需重新下载或重新构建。

## 7. 导出和导入完整编译镜像

要将固定的完整编译环境迁移到另一台电脑，先导出镜像：

```bash
docker save -o imx6ull-kernel-builder-ubuntu18.04.tar \
  imx6ull-kernel-builder:ubuntu18.04
```

`docker save` 会将 Ubuntu 18.04、host 编译依赖和内置 ARM 交叉编译器一起写入 tar 文件。

在另一台已安装 Docker 的电脑中导入：

```bash
docker load -i imx6ull-kernel-builder-ubuntu18.04.tar
```

确认镜像已导入：

```bash
docker image ls
```

另一台电脑只需要 Docker 和内核源码，不需要单独准备 ARM 交叉编译器，因为工具链已经在镜像内部。

## 8. 镜像加速器和 Zscaler 证书

Docker daemon 当前的镜像加速配置文件为：

```text
/etc/docker/daemon.json
```

当前网络使用 Zscaler TLS 拦截。系统信任的 CA 证书源文件为：

```text
/usr/local/share/ca-certificates/zscaler-root.crt
/usr/local/share/ca-certificates/zscalerthree-root.crt
```

如果 Docker 拉取镜像再次出现下列错误：

```text
x509: certificate signed by unknown authority
```

刷新系统证书库并重启 Docker：

```bash
sudo update-ca-certificates
sudo systemctl restart docker
```

完整的机器级 Docker 和证书记录见 `~/docker-environment-record.md`。

## 9. 日常最短流程

镜像已经构建完成后，日常编译只需要：

```bash
cd ~/linux/imx6ull/kernel/imx-kernel-4.1.15

docker run --rm -it \
  -v "$PWD":/workspace \
  -w /workspace \
  imx6ull-kernel-builder:ubuntu18.04 \
  bash
```

进入容器后：

```bash
source env_setup.sh
make imx_alientek_emmc_defconfig
make -j"$(nproc)"
```

使用 `exit` 退出。除非刻意修改 Dockerfile 并重新构建镜像，否则不需要再次安装依赖或交叉编译器。
