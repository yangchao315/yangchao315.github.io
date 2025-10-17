---
layout: post
title: "GPU Open Resource"
date:   2025-10-17
tags:
    - GPU
    - DRM
comments: true
author: yangchao
---

<!-- more -->
- [PVR 开源驱动](#pvr-开源驱动)
  - [开发者文档](#开发者文档)
    - [Native\_SDK](#native_sdk)
    - [PowerVR GPU 图形驱动栈](#powervr-gpu-图形驱动栈)
  - [开发者论坛](#开发者论坛)
  - [官方网站](#官方网站)
- [nvdia gpu开源驱动](#nvdia-gpu开源驱动)
  - [https://joyxu.github.io/2022/05/16/gpu-nvdia-open-source/](#httpsjoyxugithubio20220516gpu-nvdia-open-source)

# PVR 开源驱动
## 开发者文档

https://docs.imgtec.com/html/index.html

### Native_SDK

https://github.com/powervr-graphics/Native_SDK

> git clone https://github.com/powervr-graphics/Native_SDK.git

```c
cd Native_SDK
mkdir build && cd build
#cmake ..
cmake -DPVR_WINDOW_SYSTEM=X11 ..
make
```

### PowerVR GPU 图形驱动栈

https://developer.imaginationtech.com/solutions/open-source-gpu-driver/

![PowerVR GPU 开源软件栈](https://yangchao315.github.io/images/PVR-GPU-Stack.png)

开源驱动目前支持以下 Imagination GPU：

IMG AXE-1-16 —— 一款具备极高面积效率的 GPU，集成于德州仪器的 AM62 SoC中，非常适合工业人机界面（HMI）、零售自动化以及驾驶员监控系统等应用。

IMG BXS-4-64 —— 我们最新一代功能安全 GPU 中的最小核，已用于诸如德州仪器 AM68 SoC等智能视觉相机应用。

Imagination’s AXE-1-16(M) and BXS-4-64 cores are supported with Linux kernel 6.16 and later, with additional GPUs becoming available in later Linux kernel releases.

> git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git

> git clone https://gitlab.freedesktop.org/frankbinns/powervr/

## 开发者论坛

https://forums.imgtec.com/

## 官方网站

https://www.imaginationtech.com/



# nvdia gpu开源驱动

https://github.com/NVIDIA/open-gpu-kernel-modules

https://joyxu.github.io/2022/05/16/gpu-nvdia-open-source/
---

感谢阅读！
