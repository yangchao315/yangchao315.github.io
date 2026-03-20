---
layout: post
title: "K230 芯片介绍 Chapter 1"
date:   2026-03-19
tags:
    - K230
comments: true
author: yangchao
---

<!-- more -->
- [K230 芯片介绍 Chapter 1](#k230-芯片介绍-chapter-1)
  - [1.1 概述](#11-概述)
  - [1.2 特性](#12-特性)
    - [CPU 子系统](#cpu-子系统)
    - [KPU 子系统](#kpu-子系统)
    - [2D Engine](#2d-engine)
    - [DPU (深度处理单元)](#dpu-深度处理单元)
    - [内存](#内存)
    - [系统组件](#系统组件)
    - [视频输入](#视频输入)
    - [ISP](#isp)
    - [视频输出](#视频输出)
    - [视频编解码](#视频编解码)
    - [2.5D GPU](#25d-gpu)
    - [音频](#音频)
    - [外设](#外设)
    - [安全特性](#安全特性)
    - [PMU](#pmu)
  - [1.3 Block Diagram](#13-block-diagram)

# K230 芯片介绍 Chapter 1

本文档基于 Canaan K230 Product Full Datasheet V1.0 (2023-7-6)，介绍芯片 Overview、Feature 和 Block Diagram。

## 1.1 概述

K230 芯片是 Canaan Technology（NASDAQ: CAN）Kendryte® 系列 AIoT 芯片的最新一代 SoC 产品。

**核心架构特点**：
- 采用全新多异构单元加速计算架构
- 集成两个 **RISC-V C908** 计算核心
- 内置新一代 **KPU (Knowledge Process Unit)** 智能计算单元
- 支持 INT8 和 INT16 多精度 AI 算力
- 支持通用 AI 计算框架

**硬件加速单元**：
- 图像 2D Engine
- AI 2D Engine
- 2.5D GPU
- 3D 深度引擎

**应用领域**：
- 智能门锁
- 家用智能摄像头
- 词典笔
- 支付识别
- 3D 结构光视觉模块
- 无人机
- 交互机器人
- 智能家电/玩具
- 智能制造
- 智能车载座舱

## 1.2 特性

### CPU 子系统

| 参数 | CPU0 | CPU1 |
|------|------|------|
| 架构 | 64-bit RISC-V | 64-bit RISC-V |
| 最大频率 | 800 MHz | 1.6 GHz |
| 指令集 | RISC-V 64GCB | RISC-V Vector Extension 1.0 |
| FPU | 支持 | 支持 |
| VPU | - | 128-bit |
| L1 I-Cache | 32 KB | 32 KB |
| L1 D-Cache | 32 KB | 32 KB |
| L2 Cache | 128 KB | 256 KB |
| MMU | 支持 | 支持 |
| 中断源 | 208 个 | 208 个 |
| 调试接口 | JTAG | JTAG |

### KPU 子系统

KPU（Knowledge Process Unit）是 K230 的 AI 推理加速核心。

**支持的精度**：
- INT8
- INT16

**典型网络性能**：

| 网络模型 | 性能 |
|---------|------|
| ResNet50 | ≥ 85 fps @INT8 |
| MobileNet_v2 | ≥ 670 fps @INT8 |
| YoloV5S | ≥ 38 fps @INT8 |

**支持的框架**：
- TensorFlow
- PyTorch
- TFLite
- PaddlePaddle
- ONNX

**量化精度损失**：< 1%

### 2D Engine

**2D GDMA Engine**：
- X-Mirror / Y-Mirror / Rotation (90°/180°/270°)
- 典型图像旋转能力：2 × 1080×1280 YUV400 @15fps + 1 × 1080×1920 YUV420 @30fps
- AXI 数据宽度：64-bit
- 最大分辨率：64K × 64K
- 像素位宽：8/16/24/32 bits

**Non-AI 2D 功能**：
- OSD 模式
- CSC 模式
- 画边框模式
- 裁剪操作

**独立 AI 2D Engine**：
- 仿射变换 (Affine)
- 裁剪 (Crop)
- 缩放 (Resize)
- 填充 (Padding)
- 移位 (Shift)
- 色彩空间转换 (CSC)

### DPU (深度处理单元)

用于 3D 结构光深度计算：

| 参数 | 规格 |
|------|------|
| 横向最大分辨率 | 1920 × 1080 |
| 纵向最大分辨率 | 1080 × 1440 |
| 典型性能 | 1280×800@30fps |
| | 1280×1080@15fps |
| | 1920×1080@9fps |

**处理模块**：
- Img_check：输入 int8，输出 int1
- LCN：输入 int8，输出 int12
- SAD：输入 int12，输出 int16
- Post_proc：输入 int16，输出 int1
- Align：支持深度/视差对齐

### 内存

**DDR**：
- 16-bit/通道 LPDDR4，双通道，最大速度 3200 Mbps
- 32-bit LPDDR3，最大速度 2133 Mbps
- 最大容量：2GB
- 支持 1:1 / 1:2 频率比架构
- 5 个 AMBA AXI 主机端口

**SRAM**：
- 共享 SRAM：2MB
- 默认分配给 KPU：2MB
- 共享 SRAM 包含两个独立 128-bit AXI4 Slave 总线

**Flash**：
- 支持 SPI NOR Flash
- 支持 SPI NAND Flash
- 支持 XIP (Execute In Place) 模式
- 支持 Enhanced SPI (Dual/Quad/Octal)

### 系统组件

| 模块 | 功能 |
|------|------|
| **RMU** | 复位管理：上电复位去抖、WDT 复位、软件复位、子模块复位 |
| **CMU** | 时钟管理：子系统时钟生成、时钟分频、时钟切换、时钟门控。支持 DVFS |
| **PWR** | 电源控制：5 种电源模式（Power on/Sleep0/Sleep1/Standby/Powerdown） |
| **PDMA** | 8 通道，外设到 DDR/SRAM 的数据传输 |
| **SDMA** | 4 通道，64-bit AXI4 主设备，支持链接列表传输 |
| **Timer** | 最多 6 个可编程定时器，8-32 bit 可配置宽度 |
| **STC Timer** | 64-bit 定时器，用于音视频同步 |
| **Watchdog** | 32-bit 宽度，可生成系统复位 |
| **RTC** | 日历功能，支持闹钟和周期中断 |
| **Mailbox** | CPU0 和 CPU1 之间的通信，支持硬件锁和中断 |
| **温度传感器** | ±3°C 精度，测量范围 -40~125°C |

### 视频输入

- 3 × MIPI CSI（兼容 MIPI 1.2 RX 协议）
- 最大配置：3 × 2-lane 传感器 或 1 × 4-lane + 1 × 2-lane 传感器
- 支持结构光传感器，可分离 IR 数据和散斑数据
- 支持 HDR 传感器
- 支持 8/10/12/16 Bit Bayer RAW
- 支持时间戳

### ISP

**总体吞吐**：8MP @ 30fps

**主要功能**：
- 自动对焦 (AF)
- 自动白平衡 (AWB)
- 自动曝光 (AE)
- 2D/3D 降噪
- WDR 单帧宽动态
- 多曝光 HDR (DOL2/DOL3)
- 黑电平补偿
- 坏点校正
- 镜头阴影校正
- 鱼眼校正
- 数字增益
- 色彩校正矩阵 (CCM)
- Gamma 校正
- 直方图计算
- 防闪烁
- 视频稳定 (VSM)

**Multi-Context Management**：支持单 ISP Core 处理 3 个传感器

### 视频输出

- 1 × MIPI DSI（1 × 4-lane 或 1 × 2-lane）
- 分辨率：最高 2MP @ 60fps
- **13 层叠加**：
  - 4 个视频层
  - 8 个 OSD 层
  - 1 个背景层

| 视频层 | 缩放 | 旋转 | 数据格式 |
|--------|------|------|----------|
| Layer 0 | 支持 | 支持 90°/180°/270° | YUV420 2-plane |
| Layer 1 | 不支持 | 支持 90°/180°/270° | YUV420 2-plane |
| Layer 2/3 | 不支持 | 不支持 | YUV420/YUV422 2-plane |

**OSD 层支持格式**：RGB888, RGB565, ARGB8888, ARGB4444, ARGB1555, Monochrome

### 视频编解码

**编码性能**：最高 8MP @ 20fps

**支持编码格式**：
- HEVC (H.265) Main / Main10
- H.264 BP / MP / HP / High10
- JPEG (YUV420/YUV422)
- MJPEG

**解码性能**：最高 8MP @ 40fps

**支持解码格式**：
- HEVC (H.265) Main/Main10
- H.264 Baseline/Main/High/High10
- JPEG

### 2.5D GPU

**硬件组件**：
- 命令列表 DMA
- 原始光栅化器，16× 抗锯齿
- 纹理映射单元，4 Texel/cycle (双线性过滤)
- 硬件合成，帧缓冲压缩
- 曲面细分

**图像变换**：
- 纹理映射（点采样、双线性过滤）
- 拉伸、旋转、镜像
- 2.5D 透视校正投影

**绘制引擎**：
- 像素/直线绘制（任意角度）
- 渐变填充矩形
- 三角形、多边形
- 路径生成

**抗锯齿**：16× MSAA (4×4)

### 音频

**内置音频编解码**：
- 2 DAC 通道（立体声播放）：8-192 KHz
- 2 ADC 通道（麦克风录音）：8-192 KHz
- 自动电平控制 (ALC)
- 最多 8 × PDM DMIC 输入
- I2S 接口支持 2×2 扩展

**PDM 音频**：
- 采样率：2.048 / 2.8224 MHz
- 支持过采样率：×128 / ×64 / ×32
- 最多 4 个 IO

**I2S 音频**：
- 格式：Phillips / 左对齐 / 右对齐
- 采样率：8-192 KHz
- 数据宽度：32 bits

### 外设

| 外设 | 特性 |
|------|------|
| **UART** | 5 个接口，支持 9-bit 数据，32×32 FIFO |
| **GPIO** | 通用输入输出 |
| **I2C** | 内部/外部 I2C 总线 |
| **SPI** | SPI 主/从模式 |
| **USB** | USB 2.0 OTG |
| **SD/eMMC** | SD 卡和 eMMC 接口 |
| **PWM** | 脉宽调制 |
| **CAP** | 捕获功能 |
| **GMAC** | 千兆以太网 (可选) |
| **CSI** | MIPI CSI 相机接口 |

### 安全特性

K230 提供完整的安全特性，包括：
- 安全启动
- 硬件加密引擎
- 安全存储
- 完整性检查

### PMU

电源管理单元 (PMU) 负责：
- 电源域管理
- 电源模式切换
- 唤醒源管理（GPIO、PMU、定时器）

## 1.3 Block Diagram

K230 芯片官方 Block Diagram 如下：

![K230 Block Diagram](https://yangchao315.github.io/images/k230-block-diagram.png)

**官方框图解析**：

K230 采用 **C906 CPU + C908 CPU** 双核异构架构，主要模块包括：

| 模块 | 说明 |
|------|------|
| **CPU Subsystem** | C906 (800MHz) + C908 (1.6GHz) 双 RISC-V 核心 |
| **KPU** | Knowledge Process Unit，AI 推理加速单元 |
| **Video Codec** | HEVC/H.264 编解码器 |
| **ISP** | Image Signal Processor，图像信号处理器 |
| **Video Input** | MIPI CSI 接口，支持 3 路 sensor 输入 |
| **Video Output** | MIPI DSI 显示输出 |
| **DPU** | Depth Process Unit，3D 结构光深度处理 |
| **2D GE** | Graphics Engine，2D 图像加速 |
| **2.5D GPU** | 2.5D 图形处理单元 |
| **Audio** | 音频编解码模块 |
| **Security** | 安全引擎 (SEC ENG) |
| **PMU** | Power Management Unit，电源管理 |
| **DMAC** | DMA 控制器 |
| **DDR Ctrl** | LPDDR4/LPDDR3 内存控制器 |
| **SPI/Timer/RTC** | 定时器、RTC 等外设 |

---


感谢阅读！
