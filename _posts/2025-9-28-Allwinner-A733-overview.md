---
layout: post
title: "Radxa Cubie A7z"
date:   2025-9-28
tags:
    - Cubie A7z
    - A733
comments: true
author: yangchao
---

<!-- more -->
- [Cubie A7z简介](#cubie-a7z简介)
- [Linux SDK](#linux-sdk)


## Cubie A7z简介

Cubie A7Z 搭载全志 A733 SoC，内置混合架构高性能八核 CPU（双核 Arm® Cortex®-A76 与六核 Arm Cortex-A55）、RISC-V E902、Imagination PowerVR BXM-4-64 MC1 GPU 以及 3 TOPS@INT8 NPU。

A7z使用A733MX-HN3带NPU和HDMI
- 全志 A733MX-HN3(带NPU和HDMI)
- 全志 A733MX-N3X (带 NPU，不带 HDMI)
- 全志 A733MX-HXX (带 HDMI 输出，不带 NPU)

Cubie-A7z-01.png

![Cubie A7z](https://yangchao315.github.io/images/Cubie-A7z-01.png)

A733 Soc Block框图如下

![A733 Block](https://yangchao315.github.io/images/A733-Block.png)

![A733 Block Simple](https://yangchao315.github.io/images/A733-Block2.png)

A733 Soc应用场景

![A733 Application](https://yangchao315.github.io/images/A733-Application.png)

## Linux SDK

![Tina SDK](https://yangchao315.github.io/images/Tina-sdk.png)

- brandy目录下主要存放BootLoader(boot0，uboot), arisc，dram库等代码
- bsp 目录下存放着Allwinner的驱动，以及对内核的改动，保证着原生内核(kernel/linux‑X.X)比较干净的状态，并且能够快速移植适配到新内核中
- build目录存放AIOT Linux的系统构建脚本，主要功能有：
    1. 提供编译需要的环境变量、函数、规则
    2. 提供各目标模块的编译方法、规则
    3. 对接 OpenWrt, Buildroot等不同构建系统
    4. 打包生成系统固件的脚本

- device目 录 用 于 存 放 芯 片 方 案 的 配 置 文 件， 包 括 内 核 配 置，env 配 置， 分 区 表 配 置，sys_config.fex，board.dts等
- kernel目录主要存放不同版本的内核代码
- openwrt目录存放着OpenWrt原生代码，及软件包、芯片方案目录
- out 目录用于保存编译相关的临时文件和最终镜像文件，编译后自动生成此目录，例如编译方案demo_linux_aiot
- platform目录存放着一些软件包源码，这些软件包的编译方式是通用的，分别可以用在Open‑Wrt、Buildroot、debian, yocto等不同构建系统中。这个目录的存在是为了不同构建系统共用软件包提供可能性
- prebuilt目录存放着一些预编译好的工具
- rtos目录用于存放异构系统构建环境，不同平台rtos构建存在差异
- target目录对应这device目录内容。本目录主要存放不同OS 对应不同板卡，不同芯片的业务逻辑配置
- AW AIOT Linux SDK的 Buildroot系统，其包含了基于Linux系统开发用到的各种系统源码，驱动，工具，应用软件包。Buildroot是 Linux平台上的一个开源的嵌入式Linux系统自动构建框架。整个Buildroot是由  Makefile脚本和Kconfig配置文件构成的。你可以通过Buildroot配置，编译出一个完整的可以直接烧写到机器上运行的Linux系统软件

---

感谢阅读！
