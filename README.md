# Mini2440 QEMU 仿真环境搭建指南

本指南介绍如何在 Linux 系统上通过 QEMU 仿真 Mini2440 开发板，并使用 Buildroot 构建编译系统镜像。

## 环境准备

### 安装 Docker
确保主机已安装 Docker，用于构建编译环境（以 Ubuntu 为例）：
```bash
# 安装 Docker 步骤参考官方文档
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

### 拉取并启动 Ubuntu 14.04 容器
```bash
docker pull ubuntu:14.04
docker run -it --rm -v $(pwd):/root ubuntu:14.04 /bin/bash
```

## 容器内编译环境配置

1. **安装依赖工具**：
   ```bash
   sudo apt update
   sudo apt install -y gcc g++ make libc6-dev libncurses5-dev libncursesw5-dev \
       patch wget python unzip rsync bc
   ```

   > 注意：原作者使用的 `gcc-3.4.5-glibc-2.3.6` 为 32 位工具链，在 64 位容器中可能无法直接运行。  
   > 推荐直接使用 `apt` 安装的 GCC 4.8.4（版本不同但可正常编译）。


## 编译系统镜像

1. **解压并准备源码**：
   ```bash
   tar -xvf mini2440-buildroot.tar.gz
   cp ./kernel/* ./mini2440-buildroot/dl
   cp ./tools/* ./mini2440-buildroot/dl
   cp ./uboot/* ./mini2440-buildroot/dl
   ```

2. **配置并编译 Buildroot**：
   ```bash
   cd mini2440-buildroot
   make mini2440_defconfig
   make menuconfig
   ```

3. **菜单配置选项**：
   ```
   Filesystem images --->
     [*] jffs2 root filesystem
     Flash Type (NAND flash with 512B Page and 16 kB erasesize) --->
   ```

4. **开始编译**：
   ```bash
   make -j8
   ```


## 生成 NAND 镜像

1. **复制编译产物到 flashimg 目录**：
   ```bash
   cp mini2440-buildroot/output/images/rootfs.jffs2 ./flashimg
   cp mini2440-buildroot/output/images/u-boot.bin ./flashimg
   cp mini2440-buildroot/output/images/uImage ./flashimg
   ```

2. **构建 flashimg 工具并生成镜像**：
   ```bash
   cd flashimg
   ./autogen.sh
   ./configure
   make -j8
   ./flashimg -s 64M -t nand -f nand.bin -p uboot.part \
       -w boot,u-boot.bin -w kernel,uImage -w root,rootfs.jffs2 -z 512
   ```


## 运行 QEMU 仿真

### 主机环境配置（Ubuntu 22.04 为例）

1. **解决 QEMU 依赖问题**：
   ```bash
   # 处理 libncurses 依赖
   sudo ln -s /usr/lib/x86_64-linux-gnu/libncurses.so.6 /usr/lib/x86_64-linux-gnu/libncurses.so.5
   sudo ln -s /usr/lib/x86_64-linux-gnu/libtinfo.so.6 /usr/lib/x86_64-linux-gnu/libtinfo.so.5

   # 安装 SDL 依赖
   sudo apt install libsdl1.2debian
   ```

2. **启动 QEMU**：
   ```bash
   ./qemu/mini2440/bin/qemu-system-arm -M mini2440 -serial stdio -mtdblock flashimg/nand.bin -usbdevice mouse
   ```


## QEMU 内启动系统

QEMU 启动后会进入 U-Boot 命令行，执行以下命令启动内核：
```bash
nboot kernel
setenv bootargs root=/dev/mtdblock3 rootfstype=jffs2 console=ttySAC0,115200
saveenv
bootm
```

### 登录系统
```
登录账号：root（无密码）
```


## 常见问题解决

1. **编译时权限错误**：
   ```bash
   # 提示类似 "chmod: prof_err.h: new permissions are ..." 时
   chmod -R 755 mini2440-buildroot
   make -j8
   ```

2. **32 位工具链无法运行**：
   - 推荐使用 `apt` 安装的 64 位 GCC 替代
   - 若必须使用 32 位工具链，需安装 32 位兼容库：`sudo apt install libc6:i386`


## 最终效果
成功启动后可看到 Buildroot 系统界面，输入 `ls` 可查看文件系统内容。
