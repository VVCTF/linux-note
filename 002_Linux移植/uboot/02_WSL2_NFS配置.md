# 1 WSL2 中 NFS 根文件系统配置

## 1.1 使用场景

NFS（Network File System）可将主机上的根文件系统目录直接挂载为开发板的 Linux 根文件系统。相较于将 rootfs 反复烧录到 eMMC、SD 卡或 NAND，NFS Root 的优势如下：

- 主机侧修改程序、库文件或脚本后，开发板重启即可使用新内容。
- 不需要频繁制作镜像和烧录，适合内核、驱动及应用程序开发阶段。
- 主机侧能够直接查看和修改开发板正在使用的根文件系统。

本章的主机环境为 Windows 11 + WSL2 Ubuntu，开发板使用 i.MX6ULL。WSL2 默认采用 NAT 网络，局域网中的开发板不能直接访问 WSL2 内部的虚拟 IP，因此应优先使用镜像网络模式。

## 1.2 网络地址与共享目录

本次配置使用以下地址和目录：

| 项目 | 值 |
| --- | --- |
| 开发板 IP | `192.168.1.50` |
| Windows 主机/NFS 服务地址 | `192.168.1.241` |
| 网关 | `192.168.1.1` |
| 子网掩码 | `255.255.255.0` |
| NFS 根文件系统目录 | `/home/tf/linux/nfs/buildroot` |

`192.168.1.241` 必须是局域网中可被开发板访问的 Windows 主机地址，不能填写 WSL2 NAT 模式下的 `172.x.x.x` 内部地址。

# 2 WSL2 网络模式

## 2.1 NAT 模式的限制

WSL2 默认采用 NAT（网络地址转换）模式。WSL2 发行版拥有独立的虚拟网段，通常为 `172.x.x.x`，该地址对 Windows 主机可见，但开发板等局域网设备通常无法直接访问。

NFSv3 不仅使用 2049 端口，还依赖 rpcbind、mountd、lockd、statd 等 RPC 服务。NAT 模式下需要在 Windows 做端口转发，并且必须固定服务端口；WSL2 内部 IP 在重启后可能变化，维护成本较高。

## 2.2 镜像网络模式

Windows 11 22H2 及之后版本推荐配置 WSL2 镜像网络模式。该模式允许 WSL2 服务直接接受局域网设备发起的访问，避免为 NFS 配置 NAT 端口转发。

在 Windows 用户目录创建或编辑 `C:\Users\<用户名>\.wslconfig`：

```ini
[wsl2]
networkingMode=mirrored
firewall=true
```

配置说明：

- `networkingMode=mirrored`：启用镜像网络模式。
- `firewall=true`：由 Windows 防火墙控制进入 WSL2 的网络流量。因此必须为 NFS 端口创建 Windows 防火墙规则。

保存文件后，在 Windows PowerShell 中执行：

```powershell
wsl --shutdown
```

重新打开 WSL2，再执行下列命令确认网络接口状态：

```shell
ip addr
ip route
```

> 镜像网络模式下，`nfsroot` 仍应填写 Windows 主机在局域网中的 `192.168.1.241`。无需依赖 WSL2 NAT 地址。

## 2.3 NAT 模式端口转发

无法使用镜像网络模式时，才使用 NAT 端口转发方案。该方案仅适合已明确使用 TCP 的 NFSv3 配置；`netsh interface portproxy` 只能转发 TCP，不能代替 UDP 服务转发。

先在 WSL2 中获取内部 IP：

```shell
ip -4 addr show eth0
```

假设 WSL2 IP 为 `172.20.10.2`，以管理员身份打开 PowerShell 并添加 TCP 转发规则：

```powershell
$wslIp = "172.20.10.2"

netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=111   connectaddress=$wslIp connectport=111
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=2049  connectaddress=$wslIp connectport=2049
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=20048 connectaddress=$wslIp connectport=20048
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=32803 connectaddress=$wslIp connectport=32803
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=32765 connectaddress=$wslIp connectport=32765
```

查看规则：

```powershell
netsh interface portproxy show v4tov4
```

删除示例：

```powershell
netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=2049
```

WSL2 或 Windows 重启后，应重新检查 `$wslIp`。内部 IP 发生变化时，原有规则的 `connectaddress` 必须同步更新。

# 3 WSL2 NFS 服务端配置

## 3.1 安装服务

在 WSL2 Ubuntu 中安装 NFS 服务端和 rpcbind：

```shell
sudo apt update
sudo apt install -y nfs-kernel-server rpcbind
```

启用 systemd 后，可检查服务状态：

```shell
sudo systemctl status rpcbind
sudo systemctl status nfs-kernel-server
```

## 3.2 配置 NFS 导出目录

创建 NFS 根文件系统目录。实际目录通常由 Buildroot 的输出目录或手工制作的 rootfs 提供：

```shell
mkdir -p /home/tf/linux/nfs/buildroot
```

编辑 `/etc/exports`：

```shell
sudo vim /etc/exports
```

添加导出规则：

```textile
/home/tf/linux/nfs/buildroot  192.168.1.0/24(rw,sync,no_root_squash,no_subtree_check)
```

导出规则格式如下：

```textile
<导出目录>  <客户端IP或网段>(<选项>)
```

常用选项说明：

- `192.168.1.0/24`：只允许本局域网网段访问。开发阶段不建议使用 `*`，避免无关主机访问根文件系统。
- `rw`：允许客户端读写。
- `sync`：服务端同步写入数据，数据一致性较好。
- `no_root_squash`：保留开发板 root 用户的服务端权限。根文件系统运行时需要创建文件、加载模块或修改配置，开发阶段应设置该选项。
- `no_subtree_check`：关闭子目录检查，降低文件被移动或重命名时的访问问题，并减少额外检查开销。

应用并验证导出规则：

```shell
sudo exportfs -ra
sudo exportfs -v
```

输出中应出现 `/home/tf/linux/nfs/buildroot` 及其授权网段。

## 3.3 固定 NFSv3 RPC 服务端口

NFSv3 的挂载依赖多个 RPC 服务。若 mountd、lockd、statd 使用默认随机端口，Windows 防火墙或 NAT 转发规则会在服务重启后失效。因此 WSL2 环境应固定服务端口。

编辑 `/etc/nfs.conf`：

```shell
sudo vim /etc/nfs.conf
```

加入或修改以下配置：

```ini
[nfsd]
port=2049

[mountd]
port=20048

[lockd]
port=32803
udp-port=32769

[statd]
port=32765
```

端口作用如下：

| 服务 | 端口 | 作用 |
| --- | --- | --- |
| rpcbind | 111 | 查询 RPC 服务端口 |
| nfsd | 2049 | 提供 NFS 文件读写服务 |
| mountd | 20048 | 校验导出目录并返回 NFS 文件句柄 |
| lockd | 32803 | NFS 文件锁服务 |
| statd | 32765 | 锁状态监控服务 |

重启服务使端口配置生效：

```shell
sudo systemctl restart rpcbind
sudo systemctl restart nfs-kernel-server
sudo exportfs -ra
```

验证 RPC 服务及端口：

```shell
rpcinfo -p localhost
```

重点确认 `mountd` 为 TCP 20048、`nfs` 为 TCP 2049、`nlockmgr` 为 TCP 32803、`status` 为 TCP 32765。由于开发板参数指定了 `proto=tcp`，NFS 挂载的关键通信使用 TCP。

## 3.4 systemd 启动支持

WSL2 默认可能未启用 systemd，导致 NFS 服务无法按普通 Ubuntu 的方式自启动。编辑 `/etc/wsl.conf`：

```ini
[boot]
systemd=true
```

在 Windows PowerShell 中执行 `wsl --shutdown` 后重新进入 WSL2，再设置服务自启动：

```shell
sudo systemctl enable rpcbind
sudo systemctl enable nfs-kernel-server
```

# 4 Windows 防火墙配置

## 4.1 创建入站规则

镜像网络模式中，Windows 防火墙规则会影响外部设备访问 WSL2 服务。以管理员身份打开 PowerShell，创建以下入站规则：

```powershell
New-NetFirewallRule -DisplayName "NFS-TCP-111"      -Direction Inbound -Protocol TCP -LocalPort 111   -Action Allow -Profile Any
New-NetFirewallRule -DisplayName "NFS-UDP-111"      -Direction Inbound -Protocol UDP -LocalPort 111   -Action Allow -Profile Any
New-NetFirewallRule -DisplayName "NFS-TCP-2049"     -Direction Inbound -Protocol TCP -LocalPort 2049  -Action Allow -Profile Any
New-NetFirewallRule -DisplayName "NFS-mountd-20048" -Direction Inbound -Protocol TCP -LocalPort 20048 -Action Allow -Profile Any
New-NetFirewallRule -DisplayName "NFS-lockd-32803"  -Direction Inbound -Protocol TCP -LocalPort 32803 -Action Allow -Profile Any
New-NetFirewallRule -DisplayName "NFS-statd-32765"  -Direction Inbound -Protocol TCP -LocalPort 32765 -Action Allow -Profile Any
```

查看已创建的规则：

```powershell
Get-NetFirewallRule | Where-Object { $_.DisplayName -like "NFS*" } | Format-Table -AutoSize
```

验证 Windows 主机端口连通性：

```powershell
Test-NetConnection -ComputerName 192.168.1.241 -Port 2049
```

`TcpTestSucceeded : True` 表示 TCP 2049 已能够连接。

## 4.2 网络配置文件

Windows 可能将网桥或当前网络识别为 `Public`。若规则未使用 `-Profile Any`，即使规则存在也可能无法匹配当前网络配置文件。

查看当前网络类别：

```powershell
Get-NetConnectionProfile
```

确认网络可信后，可将指定网卡设为专用网络：

```powershell
Set-NetConnectionProfile -InterfaceAlias "网桥" -NetworkCategory Private
```

规则已经存在但配置文件不匹配时，可将现有 NFS 规则覆盖到全部配置文件：

```powershell
Get-NetFirewallRule | Where-Object { $_.DisplayName -like "NFS*" } | Set-NetFirewallRule -Profile Any
```

# 5 开发板 U-Boot 与内核配置

## 5.1 U-Boot 环境变量

在 U-Boot 命令行设置网络参数、下载内核和设备树，并配置 NFS 根文件系统：

```textile
setenv ipaddr 192.168.1.50
setenv ethaddr b8:ae:1d:01:00:00
setenv gatewayip 192.168.1.1
setenv netmask 255.255.255.0
setenv serverip 192.168.1.241
setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-4.3-800x480-c.dtb; bootz 80800000 - 83000000'
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.241:/home/tf/linux/nfs/buildroot,vers=3,proto=tcp rw ip=192.168.1.50:192.168.1.241:192.168.1.1:255.255.255.0::eth0:off'
saveenv
```

启动前核对参数：

```shell
printenv ipaddr
printenv serverip
printenv bootargs
ping 192.168.1.241
```

`bootargs` 参数说明：

- `console=ttymxc0,115200`：指定 i.MX6ULL 串口控制台和波特率。
- `root=/dev/nfs`：根文件系统由 NFS 提供。
- `nfsroot=192.168.1.241:/home/tf/linux/nfs/buildroot,vers=3,proto=tcp`：指定 NFS 服务端地址、导出目录、NFSv3 和 TCP 协议。
- `rw`：以读写方式挂载根文件系统。
- `ip=192.168.1.50:192.168.1.241:192.168.1.1:255.255.255.0::eth0:off`：指定开发板 IP、服务器 IP、网关、掩码、网卡及关闭 DHCP。

`nfsroot` 的路径与选项之间必须使用逗号连接，逗号后不能插入空格：

```textile
# 正确
nfsroot=192.168.1.241:/home/tf/linux/nfs/buildroot,vers=3,proto=tcp

# 错误：逗号后的空格会使参数解析异常
nfsroot=192.168.1.241:/home/tf/linux/nfs/buildroot, vers=3, proto=tcp
```

## 5.2 内核配置

内核必须在挂载根文件系统之前具备网络与 NFS 客户端能力。检查内核配置时，至少应启用以下选项：

```textile
Networking support --->
    Networking options --->
        TCP/IP networking
        IP: kernel level autoconfiguration
            IP: BOOTP support

File systems --->
    Network File Systems --->
        NFS client support
        NFS client support for NFS version 3
```

用于 NFS Root 的网络驱动、NFS 客户端和 IP 自动配置不能仅编译为模块，必须能够在挂载根文件系统前使用。

# 6 NFSv3 挂载流程与验证

## 6.1 启动流程

开发板网络启动的顺序如下：

1. U-Boot 根据 `ipaddr`、`serverip` 配置网络。
2. U-Boot 通过 TFTP 下载 `zImage` 和 `.dtb`。
3. `bootz` 启动内核，内核解析 `bootargs`。
4. 内核根据 `ip=` 配置 `eth0`，输出 `IP-Config: Complete`。
5. 内核通过 TCP 111 访问 rpcbind，查询 mountd 服务端口。
6. 内核访问 TCP 20048 的 mountd，验证 `/etc/exports` 中的导出权限并取得文件句柄。
7. 内核访问 TCP 2049 的 nfsd，挂载根文件系统。
8. 内核挂载成功后启动 `/sbin/init` 或 BusyBox `init`。

成功日志通常包含：

```textile
IP-Config: Complete:
VFS: Mounted root (nfs filesystem).
```

若挂载成功但最后出现 `Kernel panic - not syncing: No init found`，则网络和 NFS 已成功，问题在 rootfs：检查 `/sbin/init`、`/init` 或 BusyBox 链接是否存在且具备可执行权限。

## 6.2 服务端验证

服务端执行：

```shell
sudo exportfs -v
rpcinfo -p localhost
sudo ss -tnlp | grep -E ':(111|2049|20048|32803|32765)\b'
```

可使用本机挂载验证导出目录：

```shell
sudo mkdir -p /mnt/nfs-test
sudo mount -t nfs -o vers=3,proto=tcp 127.0.0.1:/home/tf/linux/nfs/buildroot /mnt/nfs-test
mount | grep nfs
sudo umount /mnt/nfs-test
```

# 7 常见问题排查

## 7.1 `Root-NFS: No NFS server available`

该错误表示内核在网络配置完成后无法完成 NFS 服务访问。按以下顺序排查：

1. 在 U-Boot 中执行 `ping 192.168.1.241`，先确认基础网络连通。
2. 在 WSL2 中执行 `sudo systemctl status rpcbind nfs-kernel-server`，确认服务处于 `active (running)`。
3. 执行 `rpcinfo -p localhost`，确认 111、2049、20048 等端口已经注册。
4. 检查 Windows 防火墙规则是否覆盖 `Profile Any`。
5. 检查 `.wslconfig` 是否启用镜像网络，配置后是否执行过 `wsl --shutdown`。
6. 检查 `printenv bootargs` 中的 NFS 服务 IP 与路径。

## 7.2 `rootpath=` 为空或 `nfsroot` 路径错误

启动日志中的 `rootpath=` 为空，通常表示内核未正确解析 `nfsroot` 参数。重点检查：

- `nfsroot=服务器IP:绝对路径,vers=3,proto=tcp` 格式完整。
- 路径与 `/etc/exports` 中的路径完全一致。
- `nfsroot` 选项之间使用逗号，逗号后没有空格。
- `ip=` 参数中服务器地址同样为 `192.168.1.241`。

每次设置环境变量后都应执行：

```shell
printenv bootargs
```

逐字核对输出，再执行 `saveenv`。

## 7.3 `access denied by server while mounting`

该错误通常是服务端导出规则或目录权限问题：

- `/etc/exports` 的授权网段未包含 `192.168.1.50`。
- 修改 `/etc/exports` 后未执行 `sudo exportfs -ra`。
- `nfsroot` 指向的目录与导出目录不一致。
- 目录及其父目录缺少必要的遍历权限。可使用 `namei -l /home/tf/linux/nfs/buildroot` 检查路径各层权限。

## 7.4 挂载成功后出现 `Permission denied`

开发板以 root 身份运行根文件系统。如果 `/etc/exports` 使用默认的 `root_squash`，远端 root 会被映射为匿名用户，创建文件、修改配置等操作可能失败。

开发阶段导出规则应包含：

```textile
no_root_squash
```

该选项会将远端 root 映射为服务端 root，只应在受控的开发局域网中使用。量产或不可信网络环境应收紧客户端范围并重新评估权限策略。

## 7.5 已放行 111 和 2049，仍无法挂载

NFSv3 还需要 mountd，文件锁相关场景还会访问 lockd 和 statd。若这些服务端口随机变化，固定的防火墙规则无法匹配。

检查：

```shell
rpcinfo -p localhost
```

若 `mountd`、`nlockmgr`、`status` 没有使用 20048、32803、32765，检查 `/etc/nfs.conf` 后重启 `nfs-kernel-server`，再重新放行对应端口。

## 7.6 Windows 防火墙规则存在但端口不通

检查网络配置文件：

```powershell
Get-NetConnectionProfile
Get-NetFirewallRule -DisplayName "NFS-TCP-2049" | Select-Object DisplayName, Profile, Enabled
```

网卡可能被识别为 `Public`，而规则只适用于 `Private`。本章的规则使用 `-Profile Any`，可避免该问题；已有规则可按第 4.2 节修改。

## 7.7 重启 Windows 或 WSL2 后失效

常见原因如下：

- NAT 模式下 WSL2 内部 IP 变化，端口转发的 `connectaddress` 已失效。
- systemd 未启用，NFS 服务没有启动。
- 修改 `.wslconfig` 后未执行 `wsl --shutdown`。

优先切换到镜像网络模式，并确认 `/etc/wsl.conf` 启用了 systemd，随后检查：

```shell
sudo systemctl status rpcbind nfs-kernel-server
sudo exportfs -v
```

# 8 快速自检清单

1. Windows 主机 `192.168.1.241` 与开发板 `192.168.1.50` 位于同一网段，U-Boot 中 `ping 192.168.1.241` 成功。
2. `.wslconfig` 已配置 `networkingMode=mirrored` 和 `firewall=true`，并已执行 `wsl --shutdown`。
3. `/etc/exports` 导出了 `/home/tf/linux/nfs/buildroot`，授权范围为 `192.168.1.0/24`，并带有 `rw,sync,no_root_squash,no_subtree_check`。
4. `/etc/nfs.conf` 固定了 2049、20048、32803、32765 端口。
5. `rpcbind` 和 `nfs-kernel-server` 均为运行状态。
6. Windows 防火墙放行 TCP 111、2049、20048、32803、32765，规则适用于 `Any` 配置文件。
7. `bootargs` 中的服务器 IP、根文件系统路径与 `/etc/exports` 完全一致，且设置后已通过 `printenv bootargs` 核对。
8. 内核已内建 NFSv3 客户端、网络驱动和 IP 自动配置能力。
