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
- [build info](#build-info)


## 下载Tina SDK

[Radxa Cubie A7A](https://linux-sunxi.org/Radxa_Cubie_A7A) 的Allwinner SDK小节，可以直接从百度网盘下载到本地

通过 **repo sync -l**命令checkout 出源码

BaiduPan: https://pan.baidu.com/s/1zcVq4l-rij7RPmJ92nccZg password: 547b


[Tina SDK 1.4.6 gitee](https://gitlab.com/tina5.0_aiot)

[cubie a7z tina bsp]（https://gitee.com/juping_zheng1/radxa-cubie-a7z-tina-bsp/tree/master/device/config/chips/a733/configs/cubie_a7z）

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

## build info

```c
INFO: chip: sun60iw2p1
INFO: platform: linux
INFO: kernel: linux-5.15
INFO: board: cubie_a7z
INFO: output: /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu

INFO: build_dtbo: make  dtboimg start.
create image file: /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu/dtbo.img...

INFO: build arisc
make: Entering directory '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/brandy/arisc'
arisc defconfig: generate ar100s/.config by sun60iw2p1_defconfig

cp scp.raw scp.bin
copy scp.bin to /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/device/config/chips/a733/bin

INFO: build_bootloader: brandy_path=/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/brandy/brandy-2.0
INFO: skip build brandy.
INFO: build kernel ...
INFO: prepare_buildserver
INFO: Prepare toolchain ...
Setup BSP files
2 ,/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/kernel/linux-5.15, /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/kernel/linux-5.15 

sunxi power version is 1.0.0
Building kernel
15983 blocks
make[1]: 进入目录“/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/kernel/build”

[ GPU]: Build module driver
make: Entering directory '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/modules/gpu'
make -C img-bxm/linux/rogue_km/build/linux/sunxi_linux BUILD=release
make[1]: Entering directory '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/modules/gpu/img-bxm/linux/rogue_km/build/linux/sunxi_linux'
******* Multiarch build:     no
******* Primary arch:        target_aarch64
******* Secondary arch:      none
******* PVR top-level arch:  rogue
******* PVR DEFS arch:       rogue
******* PVR USC arch:        rogue
******* PVR TPU arch:        rogue
******* HWDefs Root:         /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/modules/gpu/img-bxm/linux/rogue_km/hwdefs/rogue
******* Host OS:             linux
******* Target OS:           linux
WARNING: You are not specifying how to find dependent libraries, e.g., by specifying SYSROOT.
The build may fail.
  MODPOST /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/modules/gpu/img-bxm/linux/rogue_km/binary_sunxi_linux_nulldrmws_release/target_aarch64/kbuild/Module.symvers
  LD [M]  /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/modules/gpu/img-bxm/linux/rogue_km/binary_sunxi_linux_nulldrmws_release/target_aarch64/kbuild/pvrsrvkm.ko
make[1]: Leaving directory '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/modules/gpu/img-bxm/linux/rogue_km/build/linux/sunxi_linux'
make: Leaving directory '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/modules/gpu'
make: Entering directory '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/modules/gpu'
renamed 'img-bxm/linux/rogue_km/binary_sunxi_linux_nulldrmws_release/target_aarch64/kbuild/pvrsrvkm.ko' -> '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/kernel/staging/lib/modules/5.15.147/pvrsrvkm.ko'
make: Leaving directory '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/modules/gpu'
[ GPU]: Build done

---build dts for sun60iw2p1 cubie_a7z-----
INFO: Use die dtsi: /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/bsp/configs/linux-5.15/sun60iw2p1.dtsi

Use init ramdisk file: '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/kernel/linux-5.15/bsp/ramfs/ramfs_aarch64.cpio.gz'.
bootimg_build
Copy boot.img to output directory ...

sun60iw2p1 compile all(Kernel+modules+boot.img) successful


INFO: build dts ...
INFO: Prepare toolchain ...
Setup BSP files
'/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/kernel/staging/sunxi.dtb' -> '/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu/sunxi.dtb'
INFO: build rootfs ...
INFO: Prepare toolchain ...
INFO: Build ubuntu rootfs ...
./mk-image.sh cover
INFO: Build rootfs ...
INFO: 1. using exist rootfs: /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu/rootfs_def
INFO: 2. Copy the ko and others that system need...
install deb from packages/arm64
using /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/target/a733/ubuntu/package.config
pkg_list:
libcedarc=libcedarc-dev_2.0.0_arm64.deb
xserver=xserver-xorg-img-bxm_1.21.1-2_arm64.deb
INFO: install package:/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/ubuntu/packages/arm64/libcedarc/libcedarc-dev_2.0.0_arm64.deb
INFO: install package:/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/ubuntu/packages/arm64/xserver/xserver-xorg-img-bxm_1.21.1-2_arm64.deb
start copy wifi-firmware
wifi_firmware_list:
PACKAGE_AIC8800_USB_FIRMWARE
Copy wifi-firmware: PACKAGE_AIC8800_USB_FIRMWARE
cp -rf /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/ubuntu/overlay-firmware/etc /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu/rootfs_def
cp -rf /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/ubuntu/overlay-firmware/usr /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu/rootfs_def
Copy /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/ubuntu/overlay ...
Copy /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/ubuntu/overlay-debug ...
INFO: start copy firmware to /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu/rootfs_def/lib/firmware
INFO: copy firmware to /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu/rootfs_def/lib/firmware ok ...
INFO: 3. Prepare the rootfs img...
PARTITION_FEX:/work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/device/config/chips/a733/configs/cubie_a7z/ubuntu/sys_partition.fex
Creating filesystem with parameters:
    Size: 7802716160
    Block size: 4096
    Blocks per group: 32768
    Inodes per group: 8080
    Inode size: 256
    Journal blocks: 29765
    Label: 
    Blocks: 1904960
    Block groups: 59
    Reserved block group size: 471
Created filesystem with 24409/476720 inodes and 226119/1904960 blocks
INFO: ===== Build the rootfs finish =====
INFO: img path: /work/work_yc/allwinner/A7Z/tina_sdk_v1.4.6/out/a733/cubie_a7z/ubuntu/rootfs.ext4
INFO: ----------------------------------------
INFO: build OK.
INFO: ----------------------------------------

```
---

感谢阅读！
