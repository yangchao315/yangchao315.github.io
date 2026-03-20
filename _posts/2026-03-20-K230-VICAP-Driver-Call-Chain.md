---
layout: post
title: "K230 VICAP应用层到驱动调用链分析"
date:   2026-03-20
tags:
    - Canaan
    - K230
    - VICAP
    - MIPI
    - Driver
    - Camera
comments: true
author: yangchao
---

<!-- more -->
- [K230 VICAP应用层到驱动调用链分析](#k230-vicap应用层到驱动调用链分析)
  - [K230 架构概述](#k230-架构概述)
  - [调用链详解](#调用链详解)
    - [应用层 (User Space)](#应用层-user-space)
    - [MPI层 (Media Process Interface)](#mpi层-media-process-interface)
    - [Middleware层](#middleware层)
    - [驱动层 (Driver)](#驱动层-driver)
    - [硬件层 (Hardware)](#硬件层-hardware)
  - [K230 MIPI驱动分析](#k230-mipi驱动分析)
    - [MIPI接口配置](#mipi接口配置)
    - [MIPI Lane配置](#mipi-lane配置)
    - [MIPI PHY频率配置](#mipi-phy频率配置)
  - [MIPI CSI-2协议详解](#mipi-csi-2协议详解)
    - [协议概述](#协议概述)
    - [LP和HS模式](#lp和hs模式)
    - [数据传输格式](#数据传输格式)
    - [GC2093的MIPI配置](#gc2093的mipi配置)
  - [关键数据结构](#关键数据结构)
  - [ISP驱动分析](#isp驱动分析)
    - [ISP子模块配置](#isp子模块配置)
    - [ISP API函数](#isp-api函数)
    - [ISP与VICAP集成](#isp与vicap集成)
  - [MIPI CSI Linux驱动分析](#mipi-csi-linux驱动分析)
    - [Cadence CSI2RX驱动](#cadence-csi2rx驱动)
    - [TI CAL驱动](#ti-cal驱动)
    - [寄存器定义](#寄存器定义)
  - [文件路径汇总](#文件路径汇总)

# K230 VICAP应用层到驱动调用链分析

本文深入分析K230芯片中，从应用层调用到MIPI驱动硬件的完整调用链。

## K230 架构概述

K230采用**双核异构架构**：

```
┌─────────────────────────────────────────────────────┐
│                    K230 芯片                          │
├─────────────────────────────────────────────────────┤
│  大核 (Big Core)         │   小核 (Little Core)     │
│  - ARM Cortex-A         │   - RISC-V              │
│  - RT-Smart操作系统     │   - Linux               │
│  - MPP多媒体处理         │   - 基础外设驱动         │
│  - VICAP/ISP驱动        │   - 网络/存储            │
│  - 视频编解码            │                         │
└─────────────────────────────────────────────────────┘
```

**关键发现**：VICAP（视频采集）和ISP驱动运行在**大核**的RT-Smart系统上，而非Linux小核。

## 调用链详解

### 应用层 (User Space)

应用层代码位于 `01_sample_vi/sample_vicap_100ask.c`：

```c
// 1. 设置设备属性
kd_mpi_vicap_set_dev_attr(dev_num, dev_attr);

// 2. 设置通道属性  
kd_mpi_vicap_set_chn_attr(dev_num, chn_num, chn_attr);

// 3. 初始化VICAP
kd_mpi_vicap_init(dev_num);

// 4. 启动数据流
kd_mpi_vicap_start_stream(dev_num);

// 5. 捕获图像帧
kd_mpi_vicap_dump_frame(dev_num, chn_num, VICAP_DUMP_YUV, &dump_info, 1000);
```

### MPI层 (Media Process Interface)

MPI层提供统一的API接口，定义在：

**文件路径**: `k230_sdk/src/big/mpp/userapps/api/mpi_vicap_api.h`

```c
// 核心API函数
k_s32 kd_mpi_vicap_set_dev_attr(k_vicap_dev dev_num, k_vicap_dev_attr dev_attr);
k_s32 kd_mpi_vicap_set_chn_attr(k_vicap_dev dev_num, k_vicap_chn chn_num, k_vicap_chn_attr chn_attr);
k_s32 kd_mpi_vicap_init(k_vicap_dev dev_num);
k_s32 kd_mpi_vicap_start_stream(k_vicap_dev dev_num);
k_s32 kd_mpi_vicap_dump_frame(k_vicap_dev dev_num, k_vicap_chn chn_num, k_vicap_dump_format foramt,
                              k_video_frame_info *vf_info, k_u32 milli_sec);
```

### Middleware层

Middleware层实现具体的媒体处理逻辑，位于：

**文件路径**: `k230_sdk/src/big/mpp/middleware/src/kdmedia/`

Middleware层负责：
- VICAP设备管理
- ISP处理管道
- 视频缓冲管理
- 与底层驱动的交互

### 驱动层 (Driver)

驱动层是访问硬件的关键，位于RT-Smart内核：

**文件路径**: `k230_sdk/src/big/rt-smart/kernel/bsp/maix3/board/mpp/`

驱动层实现：
- 寄存器读写操作
- 中断处理
- DMA数据传输
- 时钟管理

### 硬件层 (Hardware)

最终访问VICAP硬件模块：

```
┌─────────────────────────────────────────────────────────────┐
│                      VICAP 硬件模块                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐  │
│  │ CSI RX   │──▶│ ISP Pipe │──▶│ Crop/   │──▶│ Output  │  │
│  │ (MIPI)   │   │         │   │ Scale   │   │ Buffer  │  │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘  │
│      ▲                                                    │
│      │ MIPI CSI-2                                        │
│  ┌───┴───┐                                               │
│  │ GC2093 │  摄像头Sensor                                 │
│  │ Sensor │  (1920x1080@30fps, RAW10, 2Lane)            │
│  └─────────┘                                               │
└─────────────────────────────────────────────────────────────┘
```

## K230 MIPI驱动分析

### MIPI接口配置

在 `k_vicap_comm.h` 中定义了MIPI相关枚举：

```c
// CSI编号
typedef enum {
    VICAP_CSI0 = 1,
    VICAP_CSI1 = 2,
    VICAP_CSI2 = 3,
} k_vicap_csi_num;
```

### MIPI Lane配置

```c
// MIPI Lane数量
typedef enum {
    VICAP_MIPI_1LANE = 0,  // 1通道
    VICAP_MIPI_2LANE = 1,  // 2通道 (GC2093使用)
    VICAP_MIPI_4LANE = 3,  // 4通道
} k_vicap_mipi_lanes;
```

### MIPI PHY频率配置

```c
// MIPI PHY时钟频率
typedef enum {
    VICAP_MIPI_PHY_800M  = 1,  // 800Mbps
    VICAP_MIPI_PHY_1200M = 2,  // 1200Mbps
    VICAP_MIPI_PHY_1600M = 3,  // 1600Mbps
} k_vicap_mipi_phy_freq;
```

### MIPI数据类型

```c
// 支持的数据类型
typedef enum {
    VICAP_CSI_DATA_TYPE_RAW8   = 0x2A,  // RAW8
    VICAP_CSI_DATA_TYPE_RAW10  = 0x2B,  // RAW10 (GC2093使用)
    VICAP_CSI_DATA_TYPE_RAW12  = 0x2C,  // RAW12
    VICAP_CSI_DATA_TYPE_RAW16  = 0x2E,  // RAW16
    VICAP_CSI_DATA_TYPE_YUV422_8 = 0x1E, // YUV422
} k_vicap_csi_data_type;
```

## MIPI CSI-2协议详解

### 协议概述

MIPI CSI-2 (Camera Serial Interface 2) 是移动行业定义的摄像头接口标准。

```
┌────────────────────────────────────────────────────────────┐
│                    MIPI CSI-2 分层结构                      │
├────────────────────────────────────────────────────────────┤
│  应用层        │  像素数据 (Pixel Data)                      │
├────────────────────────────────────────────────────────────┤
│  协议层        │  帧起始/结束、短帧、长包、短包               │
├────────────────────────────────────────────────────────────┤
│  物理层        │  LP/HS 模式、D-PHY                         │
├────────────────────────────────────────────────────────────┤
│  Lane层        │  1/2/4 Lane配置                           │
└────────────────────────────────────────────────────────────┘
```

### LP和HS模式

MIPI CSI-2使用两种信号模式：

| 模式 | 全称 | 用途 | 速率 |
|------|------|------|------|
| **LP** | Low Power | 控制、配置 | ≤10Mbps |
| **HS** | High Speed | 数据传输 | 最高1.6Gbps/lane |

**时序图**:

```
Lane State
    │
HS  ──┐     ┌─────────────────────────────┐     ┌─
     │     │     HS Data Transmission     │     │
LP  ──┴─────┘                             └─────┴──
    │                                           │
    ▼                                           ▼
   LP-11    LP-01    LP-00      HS-000...HS-000   LP-11
    │        │        │            │              │
    └────────┴────────┴────────────┴──────────────┘
    
    State    State   HS-Request              Exit
    Entry    Entry     & HS                         
             Sync
```

### 数据传输格式

#### 包结构

**短包** (Short Packet) - 用于控制：
```
┌─────────────────────────────────────────────────────────────┐
│ 8bit      │ 8bit     │  16bit    │ 8bit                     │
├─────────────────────────────────────────────────────────────┤
│ Virtual   │ Data     │  WC=0     │ ECC                      │
│ Channel   │ Type     │  Word Count│                         │
│ (DI)      │ (DT)     │           │                         │
└─────────────────────────────────────────────────────────────┘
```

**长包** (Long Packet) - 用于传输像素数据：
```
┌─────────────────────────────────────────────────────────────┐
│ 8bit      │ 8bit     │  16bit    │ N×8bit    │ 16bit │ 8bit │
├─────────────────────────────────────────────────────────────┤
│ Virtual   │ Data     │  Packet   │  Data     │ CRC   │ WC   │
│ Channel   │ Type     │  Length   │  Payload  │       │      │
│ (DI)      │ (DT)     │  (WC)     │           │       │      │
└─────────────────────────────────────────────────────────────┘
```

### GC2093的MIPI配置

GC2093传感器驱动位于：

**文件路径**: `k230_sdk/src/big/mpp/kernel/sensor/src/gc2093_csi2_drv.c`

#### GC2093模式配置表

```c
static k_sensor_mode gc2093_mode_info[] = {
    {
        .sensor_type = GC2093_MIPI_CSI2_1920X1080_30FPS_10BIT_LINEAR,
        .size = {
            .width = 1920,
            .height = 1080,
        },
        .fps = 30000,
        .hdr_mode = SENSOR_MODE_LINEAR,
        .bit_width = 10,
        .mipi_info = {
            .csi_id = 0,           // CSI0接口
            .mipi_lanes = 2,      // 2 Lane
            .data_type = 0x2B,    // RAW10
        },
    },
};
```

#### 关键寄存器配置

MIPI输出配置寄存器 (`gc2093_mipi2lane_1080p_30fps_linear`):

| 寄存器 | 值 | 说明 |
|--------|-----|------|
| `0x007b` | 0x2a | MIPI输出使能 |
| `0x0023` | 0x2d | MIPI时钟分频 |
| `0x0201` | 0x27 | MIPI同步码 |
| `0x0202` | 0x56 | MIPI通道选择 |
| `0x0203` | 0xb6 | MIPI数据延迟 |
| `0x0212` | 0x80 | MIPI输出控制 |
| `0x003e` | 0x91 | MIPI使能+RAW10 |

#### 计算带宽需求

对于GC2093 1080P@30fps RAW10 2Lane:

```
理论带宽 = 1920 × 1080 × 10bit × 30fps = 622.08 Mbps
每Lane速率 ≈ 311.04 Mbps (2 Lane)
考虑开销 ≈ 350 Mbps/Lane

选择PHY频率: VICAP_MIPI_PHY_800M (800Mbps) 留有充足余量
```

## 关键数据结构

### VICAP设备属性

```c
typedef struct {
    k_vicap_window acq_win;           // 采集窗口
    k_vicap_work_mode mode;          // 工作模式
    k_vicap_input_type input_type;   // 输入类型
    k_vicap_image_pattern image_pat; // Bayer格式
    k_vicap_isp_pipe_ctrl pipe_ctrl; // ISP管道控制
    k_vicap_sensor_info sensor_info; // Sensor信息
} k_vicap_dev_attr;
```

### Sensor信息

```c
typedef struct {
    const char *sensor_name;
    const char *database_name;
    k_u16 width;                      // 1920
    k_u16 height;                     // 1080
    k_vicap_csi_num csi_num;          // CSI0
    k_vicap_mipi_lanes mipi_lanes;   // 2 Lane
    k_vicap_mipi_phy_freq phy_freq;   // PHY频率
    k_vicap_csi_data_type data_type; // RAW10
    k_u16 fps;                        // 30fps
} k_vicap_sensor_info;
```

## ISP驱动分析

ISP（Image Signal Processing）图像信号处理器是VICAP的重要组成部分，负责RAW图像的处理和优化。

### ISP子模块配置

ISP包含多个图像处理子模块，通过`k_isp_submodule_cfg`结构体统一配置：

```c
typedef struct {
    k_isp_module_cfg ae;      // 自动曝光
    k_isp_module_cfg awb;    // 自动白平衡
    k_isp_module_cfg af;      // 自动对焦
    k_isp_module_cfg wb;     // 白平衡
    k_isp_module_cfg wdr;    // 宽动态范围
    k_isp_module_cfg hdr;    // HDR处理
    k_isp_module_cfg dnr2;   // 2D降噪
    k_isp_module_cfg dnr3;   // 3D降噪
    k_isp_module_cfg bls;    // 黑电平校正
    k_isp_module_cfg cac;    // 色差校正
    k_isp_module_cfg cnr;    // 色度降噪
    k_isp_module_cfg lsc;    // 镜头阴影校正
    k_isp_module_cfg rgbir;  // RGB-IR处理
    k_isp_module_cfg ee;     // 边缘增强
    k_isp_module_cfg gc;     // gamma校正
    k_isp_module_cfg ge;     // 灰度扩展
    k_isp_module_cfg gtm;    // Gamma曲线
    k_isp_module_cfg ynr;    // 亮度降噪
    k_isp_module_cfg lut3d;  // 3D查找表
    k_isp_module_cfg ccm;    // 色彩矩阵
} k_isp_submodule_cfg;
```

### ISP API函数

ISP的MPI API定义在 `mpi_isp_api.h`：

```c
// 基础API
k_s32 kd_mpi_isp_set_dev_attr(k_isp_dev dev_num, k_isp_dev_attr dev_attr);
k_s32 kd_mpi_isp_set_chn_attr(k_isp_dev dev_num, k_isp_chn chn_num, k_isp_chn_attr chn_attr);
k_s32 kd_mpi_isp_init(k_isp_dev dev_num);
k_s32 kd_mpi_isp_deinit(k_isp_dev dev_num);

// 连接管理
k_s32 kd_mpi_isp_connect(k_isp_dev dev_num);     // 连接Sensor
k_s32 kd_mpi_isp_disconnect(k_isp_dev dev_num);   // 断开连接

// 3A库注册
k_s32 kd_mpi_isp_register_3alib(k_isp_dev dev_num, 
                                  void *usr_awb_lib, 
                                  void *usr_ae_lib, 
                                  void *usr_af_lib);
k_s32 kd_mpi_isp_unregister_3alib(k_isp_dev dev_num);

// 图像输出
k_s32 kd_mpi_isp_start_stream(k_isp_dev dev_num);
k_s32 kd_mpi_isp_stop_stream(k_isp_dev dev_num);
k_s32 kd_mpi_isp_dump_frame(k_isp_dev dev_num, k_isp_chn chn_num, ...);
k_s32 kd_mpi_isp_dump_release(k_isp_dev dev_num, k_isp_chn chn_num, ...);

// AE ROI配置
k_s32 kd_mpi_isp_ae_set_roi(k_isp_dev dev_num, k_isp_ae_roi *roi);
k_s32 kd_mpi_isp_ae_get_roi(k_isp_dev dev_num, k_isp_ae_roi *roi);
```

### ISP与VICAP集成

ISP通过VICAP的`pipe_ctrl`位字段进行控制：

```c
typedef union {
    struct {
        k_u32 ae_enable : 1;      // 自动曝光
        k_u32 af_enable : 1;      // 自动对焦
        k_u32 ahdr_enable : 1;    // HDR
        k_u32 awb_enable : 1;    // 自动白平衡
        k_u32 ccm_enable : 1;    // 色彩矩阵
        k_u32 compress_enable : 1;// 压缩
        k_u32 cnr_enable : 1;   // 色度降噪
        k_u32 ynr_enable : 1;   // 亮度降噪
        k_u32 dnr3_enable : 1;   // 3D降噪
        k_u32 dnr2_enable : 1;   // 2D降噪
        k_u32 wdr_enable : 1;    // 宽动态
        k_u32 lsc_enable : 1;    // 镜头阴影校正
        k_u32 lut3d_enable : 1;  // 3D LUT
        // ... 更多位字段
    } bits;
    k_u32 data;
} k_vicap_isp_pipe_ctrl;
```

**VICAP设备属性中的ISP配置**：

```c
typedef struct {
    k_vicap_window acq_win;           // 采集窗口
    k_vicap_work_mode mode;           // 工作模式
    k_vicap_isp_pipe_ctrl pipe_ctrl; // ISP管道控制 ⚠️
    k_isp_submodule_cfg module_cfg;  // ISP子模块配置 ⚠️
    k_isp_work_mode work_mode;        // ISP工作模式
    k_isp_input_type input_type;      // 输入类型
    k_sensor_mode sensor_mode;       // Sensor模式
} k_isp_dev_attr;
```

## MIPI CSI 硬件接收架构

> ⚠️ **重要说明**：K230的MIPI CSI数据接收由**大核VICAP硬件模块**完成，**不依赖Linux CSI驱动**。Linux内核中的Cadence/ TI CSI驱动是SDK自带的通用上游代码，K230实际不使用。

### K230 实际数据路径

```
┌──────────────────────────────────────────────────────────────┐
│ 大核 RT-Smart (VICAP硬件)                                    │
│ ┌────────────────────────────────────────────────────────┐  │
│ │                   VICAP 模块                            │  │
│ │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌───────┐ │  │
│ │  │ CSI RX  │──▶│ ISP     │──▶│ Crop/   │──▶│ Output│ │  │
│ │  │ 硬件    │   │ Pipeline│   │ Scale   │   │ Buffer│ │  │
│ │  └─────────┘   └─────────┘   └─────────┘   └───────┘ │  │
│ └────────────────────────────────────────────────────────┘  │
│        ▲                                                      │
│        │ MIPI CSI-2 数据流                                   │
└────────┼─────────────────────────────────────────────────────┘
         │
         │ I2C/SPI配置
         ▼
┌──────────────────────────────────────────────────────────────┐
│ 小核 Linux                                                   │
│ ┌─────────────────┐    ┌─────────────────┐                 │
│ │ I2C/SPI驱动     │───▶│ Sensor寄存器配置 │                 │
│ └─────────────────┘    └─────────────────┘                 │
└──────────────────────────────────────────────────────────────┘
```

### SDK中的Linux CSI驱动（不用于K230）

SDK在Linux内核源码中包含了标准的CSI接收器驱动，这些是通用上游代码：

### Cadence CSI2RX驱动

Cadence CSI2RX是一个通用的MIPI CSI-2接收器IP核驱动：

**文件路径**: `little/linux/drivers/media/platform/cadence/cdns-csi2rx.c`

**关键特性**：
- 支持1-4条MIPI Lane
- 支持4个虚拟通道/数据流
- 支持多种像素格式

**关键寄存器**：

| 寄存器 | 地址偏移 | 功能 |
|--------|----------|------|
| `CSI2RX_DEVICE_CFG_REG` | 0x000 | 设备配置 |
| `CSI2RX_SOFT_RESET_REG` | 0x004 | 软件复位 |
| `CSI2RX_STATIC_CFG_REG` | 0x008 | 静态配置(Lane映射) |
| `CSI2RX_STREAM_BASE(n)` | 0x100 | 每个数据流的配置 |

**驱动数据结构**：

```c
struct csi2rx_priv {
    void __iomem *base;              // 寄存器基地址
    struct clk *sys_clk;              // 系统时钟
    struct clk *p_clk;                // 像素时钟
    struct clk *pixel_clk[4];        // 每个Lane的时钟
    struct phy *dphy;                 // D-PHY物理层
    u8 lanes[4];                      // Lane映射
    u8 num_lanes;                    // Lane数量
};
```

### TI CAL驱动

TI Camera Access Layer (CAL)是一个功能完整的CSI-2接收器驱动：

**文件路径**: `little/linux/drivers/media/platform/ti-vpe/cal.c`

**关键寄存器块**：

| 寄存器组 | 功能 |
|----------|------|
| `CAL_CSI2_PPI_CTRL(m)` | CSI2协议控制 |
| `CAL_CSI2_COMPLEXIO_CFG(m)` | Lane配置 |
| `CAL_CSI2_CTX0-7(m)` | 虚拟通道上下文 |
| `CAL_WR_DMA_CTRL` | DMA控制 |
| `CAL_WR_DMA_ADDR` | DMA地址 |
| `CAL_WR_DMA_OFST` | DMA偏移 |
| `CAL_WR_DMA_XSIZE` | DMA宽度 |
| `CAL_WR_DMA_YSIZE` | DMA高度 |

**CAL设备结构**：

```c
struct cal_dev {
    void __iomem *base;                        // 寄存器基地址
    struct cal_camerarx *phy[CAL_NUM_CSI2_PORTS];  // 2个CSI2 PHY
    struct cal_ctx *ctx[CAL_NUM_CONTEXT];     // 2个捕获上下文
};

struct cal_camerarx {
    void __iomem *base;              // PHY寄存器
    struct cal_dev *cal;             // 父设备
    unsigned int instance;            // PHY实例
};

struct cal_ctx {
    struct cal_dev *cal;             // 父设备
    struct cal_camerarx *phy;        // 关联的PHY
    unsigned int cport;               // CSI端口
};
```

### 寄存器定义

CAL驱动的关键寄存器定义位于 `ti-vpe/cal_regs.h`：

```c
// CSI2协议控制寄存器
#define CAL_CSI2_PPI_CTRL(m)           (0x200 + (m) * 0x100)

// Lane配置寄存器
#define CAL_CSI2_COMPLEXIO_CFG(m)      (0x208 + (m) * 0x100)
#define CAL_CSI2_COMPLEXIO_PHY_CFG(m)  (0x20C + (m) * 0x100)

// 虚拟通道上下文寄存器
#define CAL_CSI2_CTX0(m)               (0x300 + (m) * 0x20)
#define CAL_CSI2_CTX1(m)               (0x304 + (m) * 0x20)

// DMA写入寄存器
#define CAL_WR_DMA_CTRL                0x400
#define CAL_WR_DMA_ADDR                0x404
#define CAL_WR_DMA_OFST                0x408
#define CAL_WR_DMA_XSIZE               0x40C
#define CAL_WR_DMA_YSIZE               0x410
```

## 文件路径汇总

### VICAP相关

| 层级 | 文件路径 | 说明 |
|------|----------|------|
| 应用层 | `userapps/sample/01_sample_vi/sample_vicap_100ask.c` | 示例代码 |
| MPI API | `mpp/userapps/api/mpi_vicap_api.h` | VICAP接口定义 |
| 类型定义 | `mpp/include/comm/k_vicap_comm.h` | VICAP数据结构 |
| Sensor驱动 | `mpp/kernel/sensor/src/gc2093_csi2_drv.c` | GC2093驱动 |
| Sensor通用 | `mpp/kernel/sensor/src/sensor_dev.c` | Sensor设备框架 |

### ISP相关

| 层级 | 文件路径 | 说明 |
|------|----------|------|
| ISP API | `mpp/userapps/api/mpi_isp_api.h` | ISP接口定义 |
| ISP类型 | `mpp/include/comm/k_isp_comm.h` | ISP数据结构 |
| ISP驱动库 | `mpp/userapps/lib/libisp_drv.a` | ISP驱动静态库 |
| 3A库 | `mpp/userapps/lib/lib3a.a` | AE/AWB/AF算法库 |
| ISP示例 | `userapps/sample/sample_vicap/sample_vicap.c` | VICAP+ISP示例 |

### MIPI CSI Linux驱动（SDK通用代码，非K230实际使用）

| 组件 | 文件路径 | 说明 |
|------|----------|------|
| Cadence CSI2RX | `little/linux/drivers/media/platform/cadence/cdns-csi2rx.c` | 通用CSI2驱动（K230不用） |
| TI CAL | `little/linux/drivers/media/platform/ti-vpe/cal.c` | 通用Camera Access Layer（K230不用） |
| CAL寄存器 | `little/linux/drivers/media/platform/ti-vpe/cal_regs.h` | 寄存器定义（参考） |
| 设备树 | `little/linux/arch/riscv/boot/dts/kendryte/k230.dtsi` | 硬件配置 |

### K230 实际使用的组件

| 组件 | 文件路径 | 说明 |
|------|----------|------|
| VICAP驱动 | `big/rt-smart/kernel/bsp/maix3/board/mpp/` | 大核驱动，实际处理CSI数据 |
| Sensor驱动 | `big/mpp/kernel/sensor/src/` | Sensor寄存器配置 |
| ISP库 | `big/mpp/userapps/lib/libisp_drv.a` | ISP处理库 |

---
感谢阅读！
