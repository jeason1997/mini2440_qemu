in linux:
docker pull ubuntu:14.04
docker run -it --rm -v $(pwd):/root ubuntu:14.04 /bin/bash

in docker:
sudo apt update
sudo apt install -y gcc g++ make libc6-dev libncurses5-dev libncursesw5-dev patch wget python unzip rsync bc

注：原作者使用的GCC版本: gcc-3.4.5-glibc-2.3.6，git里也有这个压缩包，但是是32位的，在64位的docker里运行不了

　　可以在docker里直接apt install gcc安装一个，默认安装的是，gcc version 4.8.4 (Ubuntu 4.8.4-2ubuntu1~14.04.4)，虽然版本不同，但接下来的编译也能正常进行

tar -xvf mini2440-buildroot.tar.gz
cp ./kernel/* ./mini2440-buildroot/dl
cp ./tools/* ./mini2440-buildroot/dl
cp ./uboot/* ./mini2440-buildroot/dl

cd mini2440-buildroot
make mini2440_defconfig
make menuconfig
change:
Filesystem images --->
[*] jffs2 root filesystem
Flash Type (NAND flash with 512B Page and 16 kB erasesize) --->

make -j8

cp mini2440-buildroot/output/images/rootfs.jffs2 ./flashimg
cp mini2440-buildroot/output/images/u-boot.bin ./flashimg
cp mini2440-buildroot/output/images/uImage ./flashimg
cd flashimg
./autogen.sh
./configure
make -j8
./flashimg -s 64M -t nand -f nand.bin -p uboot.part -w boot,u-boot.bin -w kernel,uImage -w root,rootfs.jffs2 -z 512

in linux:
原作者编译的qemu依赖libncurses5
sudo apt install libncurses5
不过在Ubuntu 22.04的apt源里已经没这个了，大概率安装不了，可以自己编译一个，或者简单创建个软连接
# 先确认是否存在 libncurses.so.6
ls /usr/lib/x86_64-linux-gnu/libncurses.so.6
ls /usr/lib/x86_64-linux-gnu/libtinfo.so.6
# 创建软链接
sudo ln -s /usr/lib/x86_64-linux-gnu/libncurses.so.6 /usr/lib/x86_64-linux-gnu/libncurses.so.5
sudo ln -s /usr/lib/x86_64-linux-gnu/libtinfo.so.6 /usr/lib/x86_64-linux-gnu/libtinfo.so.5
它还依赖了sdl1.2
sudo apt install libsdl1.2debian
运行：
./qemu/mini2440/bin/qemu-system-arm -M mini2440 -serial stdio -mtdblock flashimg/nand.bin -usbdevice mouse

进入qemu后，会停在uboot，因为没有设置启动参数，不会自动加载内核运行，需要在uboot里执行以下命令：

nboot kernel
setenv bootargs root=/dev/mtdblock3 rootfstype=jffs2 console=ttySAC0,115200
saveenv
bootm

登录账号root

devtmpfs: mounted
Freeing init memory: 136K
Starting logging: OK
Initializing random number generator... done.
Starting network...

Welcome to Buildroot
buildroot login: root
JFFS2 notice: (788) check_node_data: wrong data CRC in data node at 0x02e3634c: read 0x76d37c5a, calculated 0xe92d9c85.
# ls
hello

 

如果在make的时候有报文件权限的错误：
chmod: prof_err.h: new permissions are r--rw-rw-, not r--r--r--
我是因为编译一半CodeSpaces关闭了，重新挂载容器进来编译才导致的
解决方法，临时给整个buildroot目录提权
chmod -R 755 mini2440-buildroot
make -j8
