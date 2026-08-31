# 1 Linux三大件移植

当编译得到Uboot.bin, zImage, 及Rootfs后，在Uboot使用如下命令快速移植至板子

```textile
setenv ipaddr 192.168.137.50
setenv ethaddr b8:ae:1d:01:00:00
setenv gatewayip 192.168.137.1
setenv netmask 255.255.255.0
setenv serverip 192.168.137.230
setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-4.3-800x480-c.dtb; bootz 80800000 - 83000000'
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.137.230:/home/tengfei/linux/nfs/rootfs,proto=tcp rw ip=192.168.137.50:192.168.137.230:192.168.137.1:255.255.255.0::eth0:off'
```

# 2 Uboot编译

## 2.1 安装依赖库

Uboot 的图形化配置界面（menuconfig）依赖 ncurses 库，如果 Ubuntu 中没有安装该库，编译时（`make menuconfig`）会报错，需要先安装：

```shell
sudo apt-get install libncurses5-dev
```

## 2.2 编译命令

先清除以前的编译记录（第一次编译可跳过），然后配置默认配置文件，最后编译：

```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_14x14_ddr512_emmc_defconfig

make V=1 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j12
```

- `distclean`：清除之前的配置和编译产生的文件（.o、.d 等中间文件）。
- `mx6ull_14x14_ddr512_emmc_defconfig`：使用 `configs` 目录下对应的默认配置文件生成 `.config`。
- `V=1`：编译时输出详细的编译信息，方便定位错误。
- `-j12`：多线程编译，12 为线程数，根据主机 CPU 核心数调整，加快编译速度。

编译成功后会在根目录下生成 `u-boot.bin`。

## 2.3 U-Boot 烧写与启动

uboot 编译好以后就可以烧写到板子上使用了，这里跟前面裸机例程一样，将 uboot 烧写到 SD 卡中，然后通过 SD 卡启动运行 uboot。使用 imxdownload 软件烧写，命令如下：

```shell
chmod 777 imxdownload        # 给予 imxdownload 可执行权限，一次即可
./imxdownload u-boot.bin /dev/sdd   # 烧写到 SD 卡，不能烧写到 /dev/sda 或 /dev/sda1 设备里面！
```

# 3 Uboot命令

## 3.1 信息查询命令

常用的和信息查询有关的命令有 3 个：`bdinfo`、`printenv` 和 `version`。

```shell
bdinfo      # 查看板子的一些信息，比如 DRAM 起始地址、大小、波特率等
printenv    # 查看 uboot 的环境变量
version     # 查看 uboot 的版本号
```

## 3.2 网络相关命令

### 3.2.1 ping 命令

用于测试开发板网络连接是否正常，需要先设置好 `ipaddr`、`serverip` 等环境变量：

```shell
ping 192.168.137.230
```

若提示 `host 192.168.137.230 is alive` 说明开发板和 Ubuntu 主机网络连接正常。

### 3.2.2 nfs 命令

`nfs` 命令用于通过网络文件系统（NFS）从 Ubuntu 主机下载文件到开发板。

**Ubuntu 端开启 NFS 服务：**

```shell
sudo apt-get install nfs-kernel-server rpcbind
```

编辑 `/etc/exports`，添加要共享的目录（比如 nfs 根文件系统所在目录）：

```textile
/home/tengfei/linux/nfs/rootfs *(rw,sync,no_subtree_check,no_root_squash)
```

重启 NFS 服务使配置生效：

```shell
sudo /etc/init.d/nfs-kernel-server restart
```

**uboot 端使用：**

```shell
nfs 80800000 192.168.137.230:/home/tengfei/linux/nfs/rootfs/zImage
```

表示将 Ubuntu 主机 `192.168.137.230` 上 `/home/tengfei/linux/nfs/rootfs/zImage` 文件通过 NFS 下载到开发板 DRAM 的 `0x80800000` 地址处。

### 3.2.3 tftp 命令

`tftp` 命令用于通过网络从 Ubuntu 主机中的 tftp 服务器下载文件到开发板 DRAM 中，速度比 nfs 命令快，一般用于下载 zImage、设备树等镜像文件。

**Ubuntu 端开启 tftp 服务：**

安装 tftp 服务器：

```shell
sudo apt-get install tftp-hpa tftpd-hpa
```

编辑配置文件 `/etc/default/tftpd-hpa`，设置 tftp 根目录，比如设置为 `/home/tengfei/linux/tftp/`：

```textile
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/home/tengfei/linux/tftp/"
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="-l -c -s"
```

启动（或重启）tftp 服务：

```shell
sudo service tftpd-hpa start
sudo service tftpd-hpa restart   # 修改配置后需要重启服务
```

将要下载的文件（比如 zImage、.dtb）拷贝到 `/home/tengfei/linux/tftp/` 目录下。

**uboot 端使用：**

```shell
tftp 80800000 zImage
```

表示将 tftp 根目录（`/home/tengfei/linux/tftp/`）下的 `zImage` 文件下载到开发板 DRAM 的 `0x80800000` 地址处。

## 3.3 emmc命令

- `mmc info`：输出 MMC 设备信息。
- `mmc read`：读取 MMC 中的数据。
- `mmc write`：向 MMC 设备写入数据。
- `mmc rescan`：扫描 MMC 设备。
- `mmc part`：列出 MMC 设备的分区。
- `mmc dev`：切换 MMC 设备。
- `mmc list`：列出当前有效的所有 MMC 设备。
- `mmc hwpartition`：设置 MMC 设备的分区。
- `mmc bootbus……`：设置指定 MMC 设备的 BOOT_BUS_WIDTH 域的值。
- `mmc bootpart……`：设置指定 MMC 设备的 boot 和 RPMB 分区的大小。
- `mmc partconf……`：设置指定 MMC 设备的 PARTITION_CONFG 域的值。

**使用示例：**

```shell
mmc list                # 列出当前开发板有效的 MMC 设备，比如 FSL_SDHC: 1 (eMMC), FSL_SDHC: 2 (SD 卡)
mmc dev 1                # 切换到 1 号 MMC 设备（emmc）
mmc info                 # 查看当前 MMC 设备（emmc）的信息，比如容量、位宽等
mmc rescan               # 重新扫描 MMC 设备

mmc part                 # 查看当前 MMC 设备的分区表

# 从 emmc 的第 0x800 个块开始，读取 0x1000 个块（每块 512 字节）的数据到 DRAM 的 0x80800000 处
mmc read 80800000 800 1000

# 将 DRAM 0x80800000 处的 0x1000 个块的数据写入到 emmc 第 0x800 个块开始的地方
mmc write 80800000 800 1000

# 设置 emmc 的分区，将 boot0、boot1 分区大小设置为 8MB，用户数据区为剩余容量，enh 表示用户区开启增强属性
mmc hwpartition dev 1 boot 8 8 enh 0 0

# 设置 1 号 MMC 设备的 BOOT_BUS_WIDTH 域
mmc bootbus 1 2 1 2

# 设置 1 号 MMC 设备的 boot 分区大小为 8MB，RPMB 分区大小为 128KB
mmc bootpart 1 8 128 0

# 设置 1 号 MMC 设备的 PARTITION_CONFIG 域，使能 boot1 分区，并且从 boot 分区启动
mmc partconf 1 1 1 1
```

## 3.4 BOOT 操作命令

### 3.4.1 bootz 命令

`bootz` 命令用于启动 zImage 镜像文件，命令格式如下：

```shell
bootz [addr [initrd[:size]] [fdt]]
```

- `addr`：zImage 镜像文件在 DRAM 中的存储地址。
- `initrd`：initrd 文件的地址，如果不需要使用 initrd，用 `-` 代替即可。
- `fdt`：设备树（.dtb）文件在 DRAM 中的存储地址。

比如将 zImage 和设备树文件通过 tftp 下载到 DRAM 中后启动 Linux 内核：

```shell
tftp 80800000 zImage
tftp 83000000 imx6ull-14x14-emmc-4.3-800x480-c.dtb
bootz 80800000 - 83000000
```

### 3.4.2 boot 命令

`boot` 命令也是用来启动 Linux 系统的，只是 `boot` 会读取环境变量 `bootcmd` 来启动 Linux 系统，`bootcmd` 是一个很重要的环境变量！其名字分为“boot”和“cmd”，也就是“引导”和“命令”，说明这个环境变量保存着引导命令，其实就是启动的命令集合，具体的引导命令内容是可以修改的。

比如设置好 `bootcmd` 后，直接执行 `boot` 即可完成启动：

```shell
setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-4.3-800x480-c.dtb; bootz 80800000 - 83000000'
saveenv
boot
```
