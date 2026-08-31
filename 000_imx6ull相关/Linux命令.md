# LINUX命令



## **一、文件与目录操作**

| 命令            | 说明                       | 示例                                                         |
| :-------------- | :------------------------- | :----------------------------------------------------------- |
| `ls`            | 列出目录内容               | `ls -l`（详细列表）                                          |
| `cd`            | 切换目录                   | `cd /home` 或 `cd ..`（返回上级）                            |
| `pwd`           | 显示当前目录路径           | `pwd`                                                        |
| `mkdir`         | 创建目录                   | `mkdir dir1` 或 `mkdir -p a/b/c`（递归创建）                 |
| `rm`            | 删除文件或目录             | `rm file.txt`，`rm -r dir`（递归删除）                       |
| `cp`            | 复制文件或目录             | `cp file1.txt file2.txt`，`cp -r dir1 dir2`                  |
| `mv`            | 移动/重命名文件或目录      | `mv old.txt new.txt`，`mv file dir/`                         |
| `touch`         | 创建空文件或更新文件时间戳 | `touch newfile.txt`                                          |
| `cat`           | 查看文件内容               | `cat file.txt`                                               |
| `less` / `more` | 分页查看文件内容           | `less log.txt`（支持上下翻页）                               |
| `head` / `tail` | 查看文件开头/结尾内容      | `head -n 5 file.txt`，`tail -f log.txt`（实时跟踪）          |
| `find`          | 查找文件                   | `find /home -name "*.txt"`                                   |
| `grep`          | 文本搜索                   | `grep "error" log.txt`，`grep -r "pattern" /dir`（递归搜索） |

**二、压缩与解压缩**

| 命令              | 说明                 | 示例                                                         |
| :---------------- | :------------------- | :----------------------------------------------------------- |
| `tar`             | 打包/解包文件        | `tar -cvf archive.tar dir/`（打包），`tar -xvf archive.tar`（解包） |
| `gzip` / `gunzip` | 压缩/解压 `.gz` 文件 | `gzip file.txt`，`gunzip file.gz`                            |

| **参数** | **全称**          | **功能**                                              |
| :------- | :---------------- | :---------------------------------------------------- |
| `-v`     | `--verbose`       | 显示详细操作过程（列出处理的文件）。                  |
| `-f`     | `--file=FILE`     | **指定归档文件名**（必须紧跟文件名）。                |
| `-z`     | `--gzip`          | 使用 **gzip** 压缩（处理 `.tar.gz` 或 `.tgz` 文件）。 |
| `-j`     | `--bzip2`         | 使用 **bzip2** 压缩（处理 `.tar.bz2` 文件）。         |
| `-J`     | `--xz`            | 使用 **xz** 压缩（处理 `.tar.xz` 文件）。             |
| `-C`     | `--directory=DIR` | 解压到指定目录（需在 `-x` 模式下使用）。              |

| **场景**               | **命令**                                              | **说明**                     |
| :--------------------- | :---------------------------------------------------- | :--------------------------- |
| **打包目录**           | `tar -cvf archive.tar /path/to/dir`                   | 将目录打包为 `archive.tar`。 |
| **打包并压缩（gzip）** | `tar -czvf archive.tar.gz /path/to/dir`               | 使用 gzip 压缩。             |
| **解压到当前目录**     | `tar -xvf archive.tar`                                | 解压 `.tar` 文件。           |
| **解压到指定目录**     | `tar -xvf archive.tar.gz -C /target/dir`              | 结合 `-C` 指定目标路径。     |
| **查看压缩包内容**     | `tar -tzvf archive.tar.gz`                            | 列出 `.tar.gz` 中的文件。    |
| **排除特定文件**       | `tar -cvf archive.tar --exclude="*.log" /path/to/dir` | 打包时忽略所有 `.log` 文件。 |

### **三、系统信息与监控**

| 命令           | 说明                       | 示例                            |
| :------------- | :------------------------- | :------------------------------ |
| `top` / `htop` | 实时查看系统进程和资源占用 | `top`，`htop`（交互式更友好）   |
| `df`           | 查看磁盘空间使用情况       | `df -h`（人类可读格式）         |
| `du`           | 查看目录/文件占用空间      | `du -sh /dir`（汇总大小）       |
| `free`         | 查看内存使用情况           | `free -h`                       |
| `uname`        | 查看系统信息               | `uname -a`（显示内核版本等）    |
| `ps`           | 查看进程状态               | `ps aux`，`ps -ef | grep nginx` |
| `kill`         | 终止进程                   | `kill -9 PID`（强制终止）       |



# 配置

## 1 TFTP

TFTP = Trivial File Transfer Protocol， 只有 **UDP 协议**（不保证可靠，但速度快、简单），没有用户名、没有密码、没有权限，嵌入式 U-Boot **100% 必支持**，专门用来：**U-Boot 下载 zImage + dtb**。

```shell
sudo gedit /etc/default/tftpd-hpa
```

替换为如下内容。

```txt
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/home/tengfei/linux/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="-l -c -s"
```

```txt
TFTP_USERNAME="tftp"
运行服务的用户（不用改）

TFTP_DIRECTORY="/home/tengfei/linux/tftpboot"
你的 TFTP 目录（放 zImage、dtb 的地方）

TFTP_ADDRESS=":69"
监听所有网卡的 69 端口（TFTP 默认端口）

TFTP_OPTIONS="-l -c -s"
-l：一直监听
-c：允许创建文件（上传）
-s：锁定目录，更安全
```

```txt
sudo systemctl restart tftpd-hpa
sudo systemctl enable tftpd-hpa  # 开机自启
```

