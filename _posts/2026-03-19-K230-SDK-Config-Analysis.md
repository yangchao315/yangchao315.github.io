---
layout: post
title: "K230 SDK 编译配置解析"
date:   2026-03-19
tags:
    - K230
comments: true
author: yangchao
---

<!-- more -->
- [K230 SDK 编译配置解析](#k230-sdk-编译配置解析)
  - [1. 概述](#1-概述)
  - [2. SDK 目录结构](#2-sdk-目录结构)
  - [3. defconfig 配置文件解析](#3-defconfig-配置文件解析)
    - [3.1 板级配置](#31-板级配置)
    - [3.2 工具链配置](#32-工具链配置)
    - [3.3 内存布局配置](#33-内存布局配置)
    - [3.4 存储配置](#34-存储配置)
    - [3.5 双核配置](#35-双核配置)
  - [4. Linux 内核](#4-linux-内核)
  - [5. Device Tree (DTS)](#5-device-tree-dts)
    - [5.1 DTS 文件结构](#51-dts-文件结构)
    - [5.2 K230 核心 DTS](#52-k230-核心-dts)
    - [5.3 板级 DTS](#53-板级-dts)
  - [6. 大小核架构](#6-大小核架构)
    - [6.1 Little Core (Linux)](#61-little-core-linux)
    - [6.2 Big Core (RT-Smart)](#62-big-core-rt-smart)
  - [7. 镜像构建流程](#7-镜像构建流程)
    - [7.1 镜像分区结构](#71-镜像分区结构)
    - [7.2 编译命令](#72-编译命令)
  - [8. 总结](#8-总结)

# K230 SDK 编译配置解析

本文基于 `make CONF=k230_canmv_dongshanpi_defconfig` 命令，深入解析 K230 SDK 的配置系统、DTS 设备树、大小核架构和镜像构建流程。

## 1. 概述

K230 SDK 是面向 Canaan K230 开发板的软件开发包，基于 **Linux + RT-Smart 双核异构系统**。

**官方资源**：
- GitHub: https://github.com/kendryte/k230_sdk
- 文档: https://github.com/kendryte/k230_docs

## 2. SDK 目录结构

```
k230_sdk/
├── configs/                 # 配置文件目录
│   ├── k230_canmv_dongshanpi_defconfig
│   ├── k230_evb_defconfig
│   └── ...
├── board/                   # 板级相关配置
│   ├── common/             # 通用板级配置
│   │   ├── gen_image_cfg/  # 镜像分区配置
│   │   ├── gen_image_script/  # 镜像生成脚本
│   │   └── env/            # 环境变量
│   ├── k230_evb_xxx/       # EVB 板级配置
│   └── k230_canmv_xxx/     # CanMV 板级配置
├── src/
│   ├── big/                # 大核代码
│   │   ├── rt-smart/       # RT-Smart 操作系统
│   │   ├── mpp/            # 多媒体处理组件
│   │   └── kmodel/         # AI 模型
│   ├── little/              # 小核代码
│   │   ├── linux/          # Linux 内核
│   │   ├── uboot/          # U-Boot 引导程序
│   │   └── buildroot-ext/  # Buildroot 根文件系统
│   └── common/              # 公共代码
│       ├── cdk/             # CDK 开发套件
│       └── opensbi/         # OpenSBI 固件
├── tools/                   # 工具
│   ├── kconfig/             # Kconfig 配置工具
│   └── gen_image_cfg/       # 镜像配置
├── Makefile                 # 主构建文件
├── repo.mak                 # 仓库版本定义
├── parse.mak                # 解析配置文件
└── Kconfig*                 # 配置系统定义
```

## 3. defconfig 配置文件解析

### 3.1 板级配置

`k230_canmv_dongshanpi_defconfig` 定义：

```bash
# 板级选择
CONFIG_BOARD_K230_CANMV_DONGSHANPI=y
CONFIG_BOARD_NAME="k230_evb"

# 引导配置
CONFIG_UBOOT_DEFCONFIG="k230_canmv_dongshanpi"
CONFIG_LINUX_DEFCONFIG="k230_canmv_dongshanpi"
CONFIG_LINUX_DTB="k230_canmv_dongshanpi"

# RT-Smart 控制台 UART
CONFIG_RTT_CONSOLE_ID=3
```

**支持的板级配置**：

| 配置项 | 说明 |
|--------|------|
| `CONFIG_BOARD_K230_EVB` | K230 EVB 开发板 |
| `CONFIG_BOARD_K230_CANMV` | CanMV-K230 开发板 |
| `CONFIG_BOARD_K230_CANMV_DONGSHANPI` | 东山 PI CanMV-K230 |
| `CONFIG_BOARD_K230D` | K230D 版本 |
| `CONFIG_BOARD_K230_FPGA` | FPGA 版本 |

### 3.2 工具链配置

```bash
# RT-Smart 工具链 (小核控制, 800MHz)
CONFIG_TOOLCHAIN_PREFIX_RTT="riscv64-unknown-linux-musl-"
CONFIG_TOOLCHAIN_PATH_RTT="/opt/toolchain/riscv64-linux-musleabi_for_x86_64-pc-linux-gnu/bin"

# Linux 工具链 (大核计算, 1.6GHz)
CONFIG_TOOLCHAIN_PREFIX_LINUX="riscv64-unknown-linux-gnu-"
CONFIG_TOOLCHAIN_PATH_LINUX="/opt/toolchain/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.0/bin"
```

**工具链版本** (定义在 `repo.mak`)：

| 组件 | 版本 Commit |
|------|------------|
| RT-Smart | `b71e7ba` |
| Linux | `67b5789` (v5.10.4) |
| U-Boot | `1f0308a` |
| OpenSBI | `1f89326` |
| MPP | `c48e112` |
| CDK | `c106662` |
| Buildroot-Ext | `3b7945c` |

### 3.3 内存布局配置

```bash
# 总内存: 1GB (0x40000000)
CONFIG_MEM_TOTAL_SIZE=0x40000000

# IPC 通信区
CONFIG_MEM_IPCM_BASE=0x00100000
CONFIG_MEM_IPCM_SIZE=0x00100000

# RT-Smart 大核内存 (126MB)
CONFIG_MEM_RTT_SYS_BASE=0x00200000
CONFIG_MEM_RTT_SYS_SIZE=0x07E00000

# Linux 小核内存 (128MB)
CONFIG_MEM_LINUX_SYS_BASE=0x08000000
CONFIG_MEM_LINUX_SYS_SIZE=0x08000000

# 多媒体内存区域 (MMZ) (252MB)
CONFIG_MEM_MMZ_BASE=0x10000000
CONFIG_MEM_MMZ_SIZE=0x0FC00000
```

**内存布局图**：

```
0x00000000 ┌─────────────────────────┐
            │       保留区           │
0x00100000 ├─────────────────────────┤
            │    IPC (1MB)          │
0x00200000 ├─────────────────────────┤
            │  RT-Smart (126MB)     │
0x08000000 ├─────────────────────────┤
            │   Linux (128MB)       │
0x10000000 ├─────────────────────────┤
            │   MMZ (252MB)         │
            │  (多媒体/AI 加速)      │
0x40000000 └─────────────────────────┘
            总计: 1GB
```

### 3.4 存储配置

```bash
# SPI NOR/NAND (未启用)
# CONFIG_SPI_NOR is not set
# CONFIG_SPI_NAND is not set

# SD/eMMC 存储
CONFIG_SDCAED=y
```

### 3.5 双核配置

```bash
# 同时支持 RT-Smart 和 Linux
CONFIG_SUPPORT_RTSMART=y
CONFIG_SUPPORT_LINUX=y

# Linux 运行在 Core 0
CONFIG_LINUX_RUN_CORE_ID=0

# 编译版本
CONFIG_BUILD_RELEASE_VER=y
```

## 4. Linux 内核

**内核版本**：Linux 5.10.4 (Kleptomaniac Octopus)

```makefile
# src/little/linux/Makefile
VERSION = 5
PATCHLEVEL = 10
SUBLEVEL = 4
EXTRAVERSION =
NAME = Kleptomaniac Octopus
```

**Linux 内核源码结构**：

```
src/little/linux/
├── arch/riscv/            # RISC-V 架构
│   └── boot/dts/
│       └── kendryte/      # K230 设备树
│           ├── k230.dtsi           # 核心 DTSI
│           ├── k230_canmv.dts      # CanMV 板级
│           ├── k230_canmv_dongshanpi.dts  # 东山PI
│           └── ...
├── drivers/               # 驱动
│   ├── media/             # ISP、视频驱动
│   ├── gpu/               # 2.5D GPU 驱动
│   └── ...
└── ...
```

## 5. Device Tree (DTS)

### 5.1 DTS 文件结构

```
k230_canmv_dongshanpi.dts  (板级 DTS)
    │
    ├── k230.dtsi          (SOC 核心 DTS)
    ├── clock_provider.dtsi (时钟树)
    ├── clock_consumer.dtsi
    ├── gpio_provider.dtsi  (GPIO)
    ├── gpio_consumer.dtsi
    ├── reset_provider.dtsi (复位)
    ├── reset_consumer.dtsi
    ├── power_provider.dtsi (电源)
    └── power_consumer.dtsi
```

### 5.2 K230 核心 DTS

`k230.dtsi` 定义了 SOC 级别的硬件：

```dts
/ {
    model = "kendryte,k230";
    compatible = "kendryte,k230";

    cpus {
        cpu@0 {
            compatible = "riscv";
            riscv,isa = "rv64imafdcvxthead";  // C906: RV64GCV
            mmu-type = "riscv,sv39";
        };
    };

    soc {
        // UART (5个)
        uart0: serial@91400000 { ... };  // Console
        uart1: serial@91401000 { ... };
        uart2: serial@91402000 { ... };
        uart3: serial@91403000 { ... };  // RT-Smart Console
        uart4: serial@91404000 { ... };

        // SD/eMMC
        mmc_sd0: sdhci0@91580000 { ... };  // eMMC/SD0
        mmc_sd1: sdhci1@91581000 { ... };  // SD Card

        // SPI
        spi0: spi@91584000 { ... };
        spi1: spi@91582000 { ... };

        // I2C, GPIO, etc.
    };
};
```

### 5.3 板级 DTS

`k230_canmv_dongshanpi.dts`：

```dts
#include "k230.dtsi"

/ {
    aliases {
        serial0 = &uart0;  // Linux console
    };

    chosen {
        stdout-path = "serial0:115200";
    };
};

&ddr {
    reg = <0x0 0x8200000 0x0 0x7dff000>;  // Linux memory
};

&mmc_sd0 {
    status = "okay";
    io_fixed_1v8;
};

&uart0 {
    status = "okay";  // Enable UART0 for Linux
};
```

## 6. 大小核架构

### 6.1 Little Core (Linux)

**定位**：系统控制、应用程序、文件系统

**配置**：
```bash
CONFIG_LINUX_RUN_CORE_ID=0              # 运行在 Core 0
CONFIG_MEM_LINUX_SYS_SIZE=0x08000000     # 128MB 内存
```

**源码路径**：`src/little/linux/`

**编译产物**：
```
output/k230_canmv_dongshanpi_defconfig/images/little-core/
├── linux_system.bin      # Linux 内核 + DTB
├── rootfs.ext4           # 根文件系统
└── uboot/                # U-Boot
    ├── fn_u-boot-spl.bin
    └── fn_ug_u-boot.bin
```

### 6.2 Big Core (RT-Smart)

**定位**：AI 推理、多媒体处理、实时计算

**配置**：
```bash
CONFIG_MEM_RTT_SYS_SIZE=0x07E00000      # 126MB 内存
CONFIG_RTT_CONSOLE_ID=3                 # UART3 控制台
```

**源码路径**：`src/big/rt-smart/`

**编译产物**：
```
output/k230_canmv_dongshanpi_defconfig/images/big-core/
├── rtt_system.bin        # RT-Smart 系统
├── fastboot_app.elf      # 启动应用
└── ai_mode.bin           # AI 模式
```

### 大小核分工

| 特性 | Little Core (Linux) | Big Core (RT-Smart) |
|------|---------------------|---------------------|
| CPU | C906 @ 800MHz | C908 @ 1.6GHz |
| 内存 | 128MB | 126MB |
| 用途 | 控制、文件系统、网络 | AI、媒体、实时 |
| 工具链 | riscv64-unknown-linux-gnu | riscv64-unknown-linux-musl |

## 7. 镜像构建流程

### 7.1 镜像分区结构

`genimage-sdcard.cfg` 定义分区：

```
SD Card / eMMC Image Layout:
┌──────────────────────────────────────────────────────────┐
│ Offset   │ Size    │ Partition      │ Description        │
├──────────┼─────────┼────────────────┼────────────────────│
│ 1MB      │ 512KB   │ uboot_spl_1   │ SPL 1st copy       │
│ 1.5MB    │ 512KB   │ uboot_spl_2   │ SPL 2nd copy       │
│ 2MB      │ 1.5MB   │ uboot         │ U-Boot             │
│ 3.5MB    │ 128KB   │ uboot_env     │ U-Boot 环境变量     │
│ 10MB     │ 20MB    │ rtt           │ RT-Smart 系统       │
│ 30MB     │ 50MB    │ linux         │ Linux 内核+DTB      │
│ 80MB     │ -       │ rootfs        │ 根文件系统 (ext4)   │
│ 128MB    │ 32MB    │ fat32appfs    │ App 分区 (vfat)    │
└──────────┴─────────┴────────────────┴────────────────────┘
```

**镜像构建脚本流程** (`gen_image.sh`)：

```bash
1. copy_file_to_images    # 复制文件到输出目录
2. gen_version             # 生成版本信息
3. add_dev_firmware        # 添加设备固件
4. gen_linux_bin           # 生成 Linux 镜像
5. gen_final_ext2          # 生成根文件系统
6. gen_rtt_bin             # 生成 RT-Smart 镜像
7. gen_uboot_bin           # 生成 U-Boot
8. gen_env_bin             # 生成环境变量
9. copy_app                # 复制应用程序
10. gen_image              # 使用 genimage 工具生成最终镜像
```

### 7.2 编译命令

**完整编译**：
```bash
# 1. 准备源码和工具链
make prepare_sourcecode

# 2. 编译指定配置
make CONF=k230_canmv_dongshanpi_defconfig

# 3. 清理
make clean
```

**编译输出**：
```
output/k230_canmv_dongshanpi_defconfig/images/
├── big-core/              # RT-Smart 大核产物
│   ├── rtt_system.bin
│   └── fastboot_app.elf
├── little-core/           # Linux 小核产物
│   ├── linux_system.bin
│   ├── rootfs.ext4
│   └── uboot/
├── sysimage-sdcard.img    # 可烧录镜像 (非安全)
└── sysimage-sdcard.img.gz # 压缩版本
```

## 8. 总结

| 项目 | 配置 |
|------|------|
| **开发板** | CanMV-K230 东山 PI |
| **Linux 内核** | 5.10.4 (Little Core) |
| **RT-Smart** | 最新版本 (Big Core) |
| **DTS** | `k230_canmv_dongshanpi.dts` |
| **Little Core** | C906 @ 800MHz, 128MB |
| **Big Core** | C908 @ 1.6GHz, 126MB |
| **存储** | SD/eMMC |
| **镜像格式** | GPT 分区表 |

**关键源码路径**：

| 组件 | 路径 |
|------|------|
| Linux 内核 | `src/little/linux/` |
| RT-Smart | `src/big/rt-smart/` |
| U-Boot | `src/little/uboot/` |
| 设备树 | `src/little/linux/arch/riscv/boot/dts/kendryte/` |
| 镜像配置 | `board/common/gen_image_cfg/` |
| 板级配置 | `board/k230_canmv_dongshanpi/` |

---

感谢阅读！
