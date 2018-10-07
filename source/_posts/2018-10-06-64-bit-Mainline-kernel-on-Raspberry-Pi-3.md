---
title: 64-bit Mainline kernel on Raspberry Pi 3
date: 2018-10-06 18:42:20
tags:
 - raspberrypi
 - linux
 - kernel
---

I've struggled a little recently on running vanilla kernel on Raspberry Pi 3. Still I didn't completely understand the internals. Anyway sharing the steps may be useful for someone like me.

Download [toolchain](http://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/gcc-linaro-7.3.1-2018.05-i686_aarch64-linux-gnu.tar.xz) and [rootfs](https://releases.linaro.org/archive/13.09/openembedded/images/lamp-armv8/linaro-image-lamp-genericarmv8-20130912-487.rootfs.tar.gz) from [Linaro](http://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/).

And clone following repos
 * [Vanilla kernel](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git)
 * [Raspberry Pi kernel](https://github.com/raspberrypi/linux) - Checkout same version as vanilla kernel you are going to use
 * [Raspberry pi firmware](https://github.com/raspberrypi/firmware) - Or download only the files under boot directory of this repo

I've created a directory structure as below. You can have similar one based on your convenience.

```sh
$ ls ~/Desktop/kernel/
total 44K
drwxr-xr-x  2 kaba kaba 4.0K Sep 23 20:04 downloads
drwxrwxr-x  2 kaba kaba 4.0K Oct  4 10:22 firmware
drwxr-xr-x 22 kaba kaba 4.0K Oct  6 11:55 kernel_out
drwxr-xr-x 18 kaba kaba 4.0K Sep 12  2013 rootfs
drwxr-xr-x 26 kaba kaba 4.0K Oct  3 21:30 rpi_kernel
drwxr-xr-x  2 kaba kaba 4.0K Oct  7 12:13 rpi_out
drwxr-xr-x  3 kaba kaba 4.0K Sep 23 19:43 toolchain
drwxr-xr-x 26 kaba kaba 4.0K Oct  3 22:04 vanila_kernel
kaba@kaba-Vostro-1550:~/Desktop/kernel
$
```
Directory		| Purpose													|
----------------|-----------------------------------------------------------|
downloads		| Having tarballs of rootfs and toolchain					|
firmware		| boot directory of Raspberry Pi firmware repo				|
kernel_out		| Output directory for Mainline kernel						|
rootfs			| rootfs tarball extracted									|
rpi_kernel		| Raspberry Pi kernel repo									|
rpi_out			| Output directory for Raspberry Pi kernel					|
toolchain		| toolchain tarball extracted								|
vanilla_kernel	| Mainline kernel repo										|

Export `PATH` variable to include toolchain directory.
```sh
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/home/kaba/Desktop/kernel/toolchain/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu/bin/
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$
```

Configure and build 64-bit Vanilla kernel.
```sh
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$ make O=../kernel_out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
.
.
.
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$ make -j4 O=../kernel_out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```
Change the suffix to `-j` according to your machine. And wait for the build to complete.

Now build device-tree in Raspberry Pi kernel repo.
```sh
kaba@kaba-Vostro-1550:~/Desktop/kernel/rpi_kernel
$ make O=../rpi_out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make[1]: Entering directory '/home/kaba/Desktop/kernel/rpi_out'
  HOSTCC  scripts/basic/fixdep
  GEN     ./Makefile
  HOSTCC  scripts/kconfig/conf.o
  YACC    scripts/kconfig/zconf.tab.c
  LEX     scripts/kconfig/zconf.lex.c
  HOSTCC  scripts/kconfig/zconf.tab.o
  HOSTLD  scripts/kconfig/conf
*** Default configuration is based on 'defconfig'
#
# configuration written to .config
#
make[1]: Leaving directory '/home/kaba/Desktop/kernel/rpi_out'
kaba@kaba-Vostro-1550:~/Desktop/kernel/rpi_kernel
$ make O=../rpi_out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- dtbs
.
.
.
```

Partition your memory card into two. The first one should be FAT and second one should be EXT4. The first partition should be a boot partition.
```sh
$ sudo mkfs.fat /dev/sdb1
mkfs.fat 4.1 (2017-01-24)
kaba@kaba-Vostro-1550:/media/kaba
$ sudo mkfs.ext4 /dev/sdb2
mke2fs 1.43.5 (04-Aug-2017)
Creating filesystem with 3863424 4k blocks and 966656 inodes
Filesystem UUID: 5f719b7c-e898-42fa-843a-b6e8e8bddbef
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done   

kaba@kaba-Vostro-1550:/media/kaba
$ sudo fdisk -l /dev/sdb
Disk /dev/sdb: 14.9 GiB, 15931539456 bytes, 31116288 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device     Boot  Start      End  Sectors  Size Id Type
/dev/sdb1  *      2048   206848   204801  100M  b W95 FAT32
/dev/sdb2       208896 31116287 30907392 14.8G 83 Linux
kaba@kaba-Vostro-1550:/media/kaba
$
```

Copy firmware and kernel to boot partition.
```sh
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$ sudo mount /dev/sdb1 /mnt/boot/
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$ sudo cp ../kernel_out/arch/arm64/boot/Image /mnt/boot/kernel8.img
[sudo] password for kaba: 
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$ sudo cp ../firmware/* /mnt/boot/
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$
```

Install device-tree blobs from Raspberry Pi repo into boot partition. The device-tree in upstream kernel is not working for some reason. I couldn't get more information regarding that.
```sh
kaba@kaba-Vostro-1550:~/Desktop/kernel/rpi_kernel
$ make O=../rpi_out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcmrpi3_defconfig
kaba@kaba-Vostro-1550:~/Desktop/kernel/rpi_kernel
$ sudo make O=../rpi_out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_PATH=/mnt/boot/ dtbs_install
```

Copy rootfs into second partition. Also install kernel modules into that.
```sh
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$ sudo mount /dev/sdb2 /mnt/rootfs/
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$ sudo cp -rf ../rootfs/* /mnt/rootfs/
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$ sudo make O=../kernel_out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=/mnt/rootfs/ modules_install
kaba@kaba-Vostro-1550:~/Desktop/kernel/vanila_kernel
$ sync
```

Create `config.txt` and `cmdline.txt` as follows. Make sure you update `device-tree` and `overlay_prefix` based on your configuration.
```sh
kaba@kaba-Vostro-1550:/mnt/boot
$ cat cmdline.txt 
dwc_otg.lpm_enable=0 console=serial0,115200 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait    
kaba@kaba-Vostro-1550:/mnt/boot
$ cat config.txt 
dtoverlay=vc4-fkms-v3d,cma-256
disable_overscan=1
dtparam=audio=on
device_tree=dtbs/4.19.0-rc5-v8+/broadcom/bcm2710-rpi-3-b.dtb
overlay_prefix=dtbs/4.19.0-rc5-v8+/overlays/
enable_uart=1
kaba@kaba-Vostro-1550:/mnt/boot
$
```

Put the SD card in Raspberry Pi 3 and boot.
