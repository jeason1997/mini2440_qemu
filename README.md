# Mini2440 QEMU 仿真环境搭建指南

本指南基于原作者教程（[链接](https://blog.csdn.net/zhuwade/article/details/127701861)）整理，详细介绍如何在 Linux 系统上通过 QEMU 仿真 Mini2440 开发板，并使用 Buildroot 构建系统镜像。


## 一、环境准备

### 1. 拉取并启动 Ubuntu 14.04 容器
通过 Docker 启动 Ubuntu 14.04 容器，同时挂载当前目录到容器内 `/root`（便于文件共享）：
```bash
# 拉取 Ubuntu 14.04 镜像
docker pull ubuntu:14.04
# 启动交互式容器（--rm 表示退出后自动删除容器）
docker run -it --rm -v $(pwd):/root ubuntu:14.04 /bin/bash
```


## 二、容器内编译环境配置

### 1. 安装基础依赖工具
在容器内执行以下命令，安装编译所需的工具链和库：
```bash
# 更新容器内 apt 源
sudo apt update
# 安装依赖（gcc、make、库文件等）
sudo apt install -y gcc g++ make libc6-dev libncurses5-dev libncursesw5-dev \
    patch wget python unzip rsync bc
```

> 注意：原作者使用的 `gcc-3.4.5-glibc-2.3.6` 为 32 位工具链，在 64 位容器中可能无法直接运行。  
> 可直接使用 `apt` 安装，默认安装的版本是 GCC 4.8.4（版本不同但可正常编译）。


## 三、编译系统镜像（容器内操作）

### 1. 解压并准备源码
原作者已经缓存了构建需要的临时文件，执行以下命令解压并复制依赖文件：
```bash
# 解压 Buildroot 源码包
tar -xvf mini2440-buildroot.tar.gz
# 复制 kernel、tools、uboot 相关文件到 Buildroot 的 dl 目录（依赖文件目录）
cp ./kernel/* ./mini2440-buildroot/dl
cp ./tools/* ./mini2440-buildroot/dl
cp ./uboot/* ./mini2440-buildroot/dl
```

### 2. 配置 Buildroot
```bash
# 进入 Buildroot 目录
cd mini2440-buildroot
# 加载 Mini2440 默认配置
make mini2440_defconfig
# 进入图形化配置界面（调整文件系统选项）
make menuconfig
```

### 3. 图形化配置关键选项
在 `make menuconfig` 界面中，按以下路径调整配置：
```
Filesystem images --->
  [*] jffs2 root filesystem  # 勾选 JFFS2 文件系统（Mini2440 NAND Flash 适用）
  Flash Type (NAND flash with 512B Page and 16 kB erasesize) --->  # 选择 NAND Flash 类型
```
配置完成后，按 `ESC` 退出并保存。

### 4. 编译 Buildroot
```bash
# 多线程编译（-j8 表示 8 线程，可根据 CPU 核心数调整）
make -j8
```


## 四、生成 NAND 镜像（容器内操作）

### 1. 解压 flashimg 工具包
flashimg 是生成 NAND 镜像的工具，先解压工具包：
```bash
# 解压 flashimg.tar.gz 工具包
tar -xvf flashimg.tar.gz
```

### 2. 复制编译产物到 flashimg 目录
将 Buildroot 编译生成的 `u-boot.bin`、`uImage`（内核镜像）、`rootfs.jffs2`（根文件系统）复制到 flashimg 目录：
```bash
cp mini2440-buildroot/output/images/rootfs.jffs2 ./flashimg
cp mini2440-buildroot/output/images/u-boot.bin ./flashimg
cp mini2440-buildroot/output/images/uImage ./flashimg
```

### 3. 构建 flashimg 工具并生成 NAND 镜像
```bash
# 进入 flashimg 目录
cd flashimg
# 生成 configure 脚本
./autogen.sh
# 配置编译选项
./configure
# 编译 flashimg 工具
make -j8
# 生成 NAND 镜像（64M 大小，分区写入 uboot、kernel、rootfs）
./flashimg -s 64M -t nand -f nand.bin -p uboot.part \
    -w boot,u-boot.bin -w kernel,uImage -w root,rootfs.jffs2 -z 512
```
生成的 `nand.bin` 即为最终的 NAND Flash 镜像文件。


## 五、运行 QEMU 仿真（主机端操作）

### 1. 解压 QEMU 工具包
原作者已编译好 Mini2440 专用 QEMU，无需手动编译，直接解压使用：
```bash
# 解压 mini2440-qemu.tar.gz 工具包
tar -xvf mini2440-qemu.tar.gz
```

### 2. 解决 QEMU 依赖问题
QEMU 运行依赖 `libncurses5` 和 `SDL1.2`，Ubuntu 22.04 需通过软链接解决 `libncurses5` 依赖：
```bash
# 1. 确认系统已安装 libncurses6（Ubuntu 22.04 默认版本）
ls /usr/lib/x86_64-linux-gnu/libncurses.so.6
ls /usr/lib/x86_64-linux-gnu/libtinfo.so.6

# 2. 创建软链接，将 libncurses6 链接为 libncurses5（解决依赖冲突）
sudo ln -s /usr/lib/x86_64-linux-gnu/libncurses.so.6 /usr/lib/x86_64-linux-gnu/libncurses.so.5
sudo ln -s /usr/lib/x86_64-linux-gnu/libtinfo.so.6 /usr/lib/x86_64-linux-gnu/libtinfo.so.5

# 3. 安装 SDL1.2 依赖
sudo apt install libsdl1.2debian
```

### 3. 启动 QEMU 仿真
```bash
# 进入 QEMU 目录，启动 Mini2440 仿真（串口输出到终端，挂载 NAND 镜像）
./qemu/mini2440/bin/qemu-system-arm -M mini2440 -serial stdio -mtdblock flashimg/nand.bin -usbdevice mouse
```


## 六、QEMU 内启动系统（U-Boot 命令行）

QEMU 启动后会自动进入 U-Boot 命令行，执行以下命令配置并启动内核：
```bash
# 1. 从 NAND Flash 加载内核到内存
nboot kernel
# 2. 设置内核启动参数（根文件系统位置、类型、控制台）
setenv bootargs root=/dev/mtdblock3 rootfstype=jffs2 console=ttySAC0,115200
# 3. 保存启动参数（重启后生效）
saveenv
# 4. 启动内核
bootm
```

### 登录系统
内核启动完成后，进入登录界面：
```
buildroot login: root  # 账号：root（无密码）
```


## 七、常见问题解决

### 1. 编译时权限错误
现象：提示 `chmod: prof_err.h: new permissions are r--rw-rw-, not r--r--r--`  
原因：编译中断后重新挂载容器，导致文件权限异常  
解决：
```bash
# 递归修改 Buildroot 目录权限
chmod -R 755 mini2440-buildroot
# 重新编译
make -j8
```


## 八、最终效果
成功登录后，输入 `ls` 可查看根文件系统内容，此时已完成 Mini2440 QEMU 仿真环境搭建，可进行后续开发测试。
