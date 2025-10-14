---
layout: post
title: "Radxa Cubie A7z SDK"
date:   2025-10-10
tags:
  - Cubie A7z
  - A733
comments: true
author: yangchao
---

<!-- more -->
- [下载Tina SDK](#下载tina-sdk)
  - [打上patch](#打上patch)


## 下载Tina SDK

[Radxa Cubie A7A](https://linux-sunxi.org/Radxa_Cubie_A7A) 的Allwinner SDK小节，可以直接从百度网盘下载到本地

通过 **repo sync -l**命令checkout 出源码

BaiduPan: https://pan.baidu.com/s/1zcVq4l-rij7RPmJ92nccZg password: 547b


[Tina SDK 1.4.6 gitee](https://gitlab.com/tina5.0_aiot)

### 打上patch

[参考](https://gitee.com/juping_zheng1/radxa-cubie-a7z-tina-bsp)按照gitee中的README进行打补丁

```c
$ ls
打补丁方法.md   brandy  bsp-patchs    build      build.sh   build-uboot.txt  device  kernel   out       prebuilt   rtos    test   ubuntu
bootloader.txt  bsp     bsp_patch.sh  buildroot  build.txt  debian           docs    LICENSE  platform  README.md  target  tools  yocto

```

**ubuntu24.04-base-arm64.tar.gz 选项需要在 ubuntu 目录下制作好 rootfs 后才能使用，参考ubuntu目录下方法即可，注意ubuntu的rootfs制作的文件不要使用扩展硬盘，会有权限问题**

- build 命令
```c
source ./build/envsetup.sh
./build.sh config
./build.sh
./build.sh pack
```

```c
$ ls /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu/
a733_linux_cubie_a7z_uart0.img  arisc   boot0_nand_sun60iw2p1.bin    boot.img  dtbo.img  dts_dep              lib            rootfs_def   sboot_sun60iw2p1.bin  System.map             vmlinux
all_modules                     bImage  boot0_sdcard_sun60iw2p1.bin  dist      dtc       fes1_sun60iw2p1.bin  ramfs.cpio.gz  rootfs.ext4  sunxi.dtb             u-boot-sun60iw2p1.bin  vmlinux.tar.bz2

```

- 生成img

**./build.sh pack**

> /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733_linux_cubie_a7z_uart0.img

---

感谢阅读！
