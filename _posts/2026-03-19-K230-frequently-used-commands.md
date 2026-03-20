---
layout: post
title: "K230 SDK常用命令"
date:   2026-03-19
tags:
    - K230
comments: true
author: yangchao
---

<!-- more -->
- [K230 SDK常用命令](#k230-sdk常用命令)
  - [基础编译命令](#基础编译命令)
  - [所有命令解析](#所有命令解析)
    - [SDK编译命令](#sdk编译命令)
    - [RT-Smart相关](#rt-smart相关)
    - [CDK开发](#cdk开发)
    - [U-Boot相关](#u-boot相关)
    - [Linux内核](#linux内核)
    - [Buildroot](#buildroot)
    - [镜像构建](#镜像构建)

# K230 SDK常用命令

本文整理了嘉楠K230开发板的SDK编译常用命令，涵盖从环境准备到镜像构建的完整流程。

## 基础编译命令

| 命令 | 解释 |
|------|------|
| `source tools/get_download_url.sh && make prepare_sourcecode` | 下载toolchain和准备源码 |
| `make CONF=k230_canmv_dongshanpi_defconfig prepare_memory` | 配置板级型号 |
| `make CONF=k230_canmv_dongshanpi_defconfig` | 编译SDK |

## 所有命令解析

### SDK编译命令

| 命令 | 解释 |
|------|------|
| `make CONF=k230_canmv_dongshanpi_defconfig` | 编译dshanpi-canmv开发板配置，会编译生成相应配置的固件 |
| `make` | 构建k230 SDK所有配置项 |
| `make prepare_sourcecode` | 下载并准备源码 |

### RT-Smart相关

| 命令 | 解释 |
|------|------|
| `make little-core-opensbi` | 构建k230小核心opensbi |
| `make big-core-opensbi` | 构建k230大核心opensbi |
| `make rt-smart` | 构建mpp rtsmart内核、userapps和opensbi |
| `make rt-smart-kernel` | 构建rtsmart内核 |
| `make rt-smart-apps` | 构建rtsmart用户应用程序 |

### CDK开发

| 命令 | 解释 |
|------|------|
| `make cdk-kernel` | 构建CDK内核代码 |
| `make cdk-kernel-install` | 将CDK内核的编译产品安装到rt-smart和rootfs |
| `make cdk-user` | 构建CDK用户代码 |
| `make cdk-user-install` | 将CDK用户的编译产品安装到rt-smart和rootfs |

### U-Boot相关

| 命令 | 解释 |
|------|------|
| `make uboot` | 用defconfig构建k230 uboot代码 |
| `make uboot-menuconfig` | uboot的Menuconfig，选择保存将保存到 `output/xxx_defconfig/little/uboot/.config` |
| `make uboot-savedefconfig` | 将uboot配置保存到 `output/xxx_defconfig/little/uboot/defconfig` |
| `make uboot-rebuild` | 重建k230 uboot |
| `make uboot-clean` | 在k230 uboot构建目录中执行clean |

### Linux内核

| 命令 | 解释 |
|------|------|
| `make linux` | 用defconfig构建k230 Linux代码 |
| `make linux-rebuild` | 重建k230 Linux内核 |
| `make linux-menuconfig` | Linux内核的Menuconfig，选择保存将保存到 `output/xxx_defconfig/little/linux/.config` |
| `make linux-savedefconfig` | 将Linux内核配置保存到 `output/xxx_defconfig/little/linux/defconfig` |
| `make linux-clean` | 在Linux内核构建目录中进行clean |

### Buildroot

| 命令 | 解释 |
|------|------|
| `make buildroot` | 用defconfig构建k230 buildroot |
| `make buildroot-rebuild` | 重建k230 buildroot |
| `make buildroot-menuconfig` | k230 buildroot的Menuconfig，选择保存将保存到 `output/xxx_defconfig/little/buildroot-ext/.config` |
| `make buildroot-savedefconfig` | 将buildroot配置保存到 `src/little/buildroot-ext/configs/xxx_defconfig` |
| `make buildroot-clean` | 清理k230 buildroot构建目录 |

### 镜像构建

| 命令 | 解释 |
|------|------|
| `make build-image` | 构建k230 rootfs镜像 |

> **注意**: SDK不支持多进程编译，不要增加类似 `-j32` 多进程编译参数。

---

感谢阅读！
