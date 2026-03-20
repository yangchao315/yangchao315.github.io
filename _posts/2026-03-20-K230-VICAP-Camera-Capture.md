---
layout: post
title: "K230 VICAP摄像头图像捕获实验"
date:   2026-03-20
tags:
    - Canaan
    - K230
    - VICAP
    - Camera
    - SDK
comments: true
author: yangchao
---

<!-- more -->
- [K230 VICAP摄像头图像捕获实验](#k230-vicap摄像头图像捕获实验)
  - [概述](#概述)
  - [编译方法](#编译方法)
    - [前置条件](#前置条件)
    - [编译步骤](#编译步骤)
  - [测试方法](#测试方法)
    - [运行程序](#运行程序)
    - [保存图像](#保存图像)
    - [测试结果](#测试结果)
  - [代码详解](#代码详解)
    - [VB视频缓存池初始化](#vb视频缓存池初始化)
    - [VICAP设备属性配置](#vicap设备属性配置)
    - [VICAP通道属性配置](#vicap通道属性配置)
    - [图像Dump流程](#图像dump流程)
  - [涉及的驱动与内核模块](#涉及的驱动与内核模块)
    - [VICAP驱动](#vicap驱动)
    - [Sensor驱动](#sensor驱动)
    - [ISP驱动](#isp驱动)
    - [VB视频缓存驱动](#vb视频缓存驱动)

# K230 VICAP摄像头图像捕获实验

本文基于 K230 SDK 中的 `01_sample_vi` 示例，介绍如何使用 VICAP（Video Capture）模块捕获摄像头图像数据。

## 概述

VICAP（Video Capture）是 K230 芯片中负责视频图像采集的重要模块。该示例演示了：

- 初始化 VICAP 设备并配置 Sensor
- 配置视频缓存池（VB）
- 捕获 YUV420 格式的图像数据
- 将图像数据保存为文件

使用的默认 Sensor 为 **GC2093**，这是一款 1080P MIPI CSI-2 接口的图像传感器。

## 编译方法

### 前置条件

确保已完成以下步骤：

1. SDK 环境搭建（参考 [K230 SDK环境搭建](https://yangchao315.github.io/2026/03/19/K230-SDK-Environment-Setup/)）
2. 已下载 toolchain 和源码
3. 已挂载工具链目录

### 编译步骤

1. 进入 `01_sample_vi` 目录：

```bash
cd k230_sdk/userapps/sample/01_sample_vi
```

2. 执行编译：

```bash
make
```

3. 编译产物位于：

```bash
k230_sdk/userapps/sample/elf/01_sample_vi.elf
```

> **注意**: 编译依赖 `MPP_SRC_DIR` 环境变量，该变量在 SDK 的顶层 Makefile 中设置。请确保从 SDK 根目录执行编译，或正确设置环境变量。

## 测试方法

### 运行程序

1. 将编译好的 ELF 文件拷贝到开发板

2. 使用以下命令运行：

```bash
./sample_vicap -dev 0
```

参数说明：

| 参数 | 说明 | 可选值 |
|------|------|--------|
| `-dev` | VICAP设备号 | 0, 1, 2 |
| `-help` | 显示帮助信息 | - |

### 保存图像

程序启动后会进入交互模式，按下键盘对应按键执行操作：

```
---------------------------------------
 Input character to select test option
---------------------------------------
 d: dump data addr test
 q: to exit
---------------------------------------
please Input:
```

- 按 `d` 键：捕获一帧图像并保存为 YUV420SP 格式文件
- 按 `q` 键：退出程序

### 测试结果

每次成功保存图像后，会生成文件名格式为：

```
dev_00_chn_00_1920x1080_XXXX.yuv420sp
```

- `dev_00`：设备号
- `chn_00`：通道号
- `1920x1080`：图像分辨率
- `XXXX`：保存序号

## 代码详解

### VB视频缓存池初始化

VICAP 模块依赖视频缓存池（Video Buffer）来管理图像数据内存：

```c
static k_s32 sample_vicap_vb_init(vicap_device_obj *dev_obj)
{
    k_s32 ret = 0;
    k_vb_config config;
    
    memset(&config, 0, sizeof(config));
    config.max_pool_cnt = 64;

    // 配置公共缓存池
    config.comm_pool[k].blk_cnt = VICAP_OUTPUT_BUF_NUM;  // 缓冲区数量：6
    config.comm_pool[k].mode = VB_REMAP_MODE_NOCACHE;    // 无缓存重映射

    // 计算缓冲区大小（YUV420 = width * height * 3/2）
    k_u16 out_width = dev_obj[i].out_win[j].width;
    k_u16 out_height = dev_obj[i].out_win[j].height;
    config.comm_pool[k].blk_size = VICAP_ALIGN_UP(
        (out_width * out_height * 3 / 2), 
        VICAP_ALIGN_1K
    );

    // 设置VB配置并初始化
    ret = kd_mpi_vb_set_config(&config);
    ret = kd_mpi_vb_init();
    
    return 0;
}
```

### VICAP设备属性配置

配置采集窗口、输入类型和ISP管道控制：

```c
// 获取Sensor信息
ret = kd_mpi_vicap_get_sensor_info(
    device_obj[dev_num].sensor_type,  // GC2093_MIPI_CSI2_1920X1080_30FPS_10BIT_LINEAR
    &device_obj[dev_num].sensor_info
);

// 配置设备属性
dev_attr.input_type = VICAP_INPUT_TYPE_SENSOR;    // 输入类型：Sensor
dev_attr.acq_win.h_start = 0;                     // 采集窗口水平起始
dev_attr.acq_win.v_start = 0;                     // 采集窗口垂直起始
dev_attr.acq_win.width = device_obj[dev_num].in_width;   // 1920
dev_attr.acq_win.height = device_obj[dev_num].in_height; // 1080
dev_attr.mode = VICAP_WORK_ONLINE_MODE;           // 在线模式

// ISP管道控制
dev_attr.pipe_ctrl.bits.ae_enable = K_TRUE;      // 自动曝光
dev_attr.pipe_ctrl.bits.awb_enable = K_TRUE;      // 自动白平衡
dev_attr.pipe_ctrl.bits.dnr3_enable = K_FALSE;   // 3D降噪
dev_attr.pipe_ctrl.bits.ahdr_enable = K_FALSE;   // HDR

// 设置设备属性
ret = kd_mpi_vicap_set_dev_attr(dev_num, dev_attr);
```

### VICAP通道属性配置

配置通道输出格式、分辨率和缓冲区：

```c
// 配置通道属性
chn_attr.out_win.width = device_obj[dev_num].out_win[chn_num].width;
chn_attr.out_win.height = device_obj[dev_num].out_win[chn_num].height;
chn_attr.crop_win.width = device_obj[dev_num].in_width;
chn_attr.crop_win.height = device_obj[dev_num].in_height;
chn_attr.pix_format = PIXEL_FORMAT_YUV_SEMIPLANAR_420;  // 输出格式
chn_attr.buffer_num = VICAP_OUTPUT_BUF_NUM;              // 缓冲区数量
chn_attr.chn_enable = K_TRUE;                            // 通道使能

// 设置通道属性
ret = kd_mpi_vicap_set_chn_attr(dev_num, chn_num, chn_attr);

// 初始化设备并启动流
ret = kd_mpi_vicap_init(dev_num);
ret = kd_mpi_vicap_start_stream(dev_num);
```

### 图像Dump流程

从 VICAP 设备获取一帧图像并保存：

```c
// 1. Dump一帧图像
memset(&dump_info, 0, sizeof(k_video_frame_info));
ret = kd_mpi_vicap_dump_frame(
    dev_num, chn_num, 
    VICAP_DUMP_YUV,      // DUMP YUV格式
    &dump_info, 
    1000                 // 超时时间(ms)
);

// 2. 计算图像大小
data_size = dump_info.v_frame.width * dump_info.v_frame.height * 3 / 2;

// 3. 映射物理地址到虚拟地址
virt_addr = kd_mpi_sys_mmap(dump_info.v_frame.phys_addr[0], data_size);

// 4. 保存到文件
snprintf(filename, sizeof(filename), 
    "dev_%02d_chn_%02d_%dx%d_%04d.yuv420sp",
    dev_num, chn_num, dump_info.v_frame.width, 
    dump_info.v_frame.height, dump_count);
    
FILE *file = fopen(filename, "wb+");
fwrite(virt_addr, 1, data_size, file);
fclose(file);

// 5. 解除映射
kd_mpi_sys_munmap(virt_addr, data_size);

// 6. 释放Dump帧
ret = kd_mpi_vicap_dump_release(dev_num, chn_num, &dump_info);
```

## 涉及的驱动与内核模块

### VICAP驱动

VICAP（Video Capture）驱动负责：

- MIPI CSI-2 接口数据接收
- 图像格式转换
- 缩放、裁剪等图像处理
- 与 ISP 模块对接

相关 API：

- `kd_mpi_vicap_set_dev_attr()` - 设置设备属性
- `kd_mpi_vicap_set_chn_attr()` - 设置通道属性
- `kd_mpi_vicap_init()` - 初始化设备
- `kd_mpi_vicap_start_stream()` - 启动数据流
- `kd_mpi_vicap_dump_frame()` - 获取帧数据
- `kd_mpi_vicap_dump_release()` - 释放帧数据

### Sensor驱动

Sensor（GC2093）驱动负责：

- 图像传感器初始化
- 曝光、白平衡控制
- 输出分辨率和帧率配置

该示例使用 `GC2093_MIPI_CSI2_1920X1080_30FPS_10BIT_LINEAR` 配置：

| 参数 | 值 |
|------|-----|
| 分辨率 | 1920x1080 |
| 帧率 | 30fps |
| 位深 | 10bit |
| 接口 | MIPI CSI-2 |
| 模式 | Linear |

### ISP驱动

ISP（Image Signal Processor）驱动负责：

- 自动曝光（AE）
- 自动白平衡（AWB）
- 3D降噪（3DNR）
- HDR处理

通过 `pipe_ctrl` 字段控制各项功能的开关。

### VB视频缓存驱动

VB（Video Buffer）驱动负责：

- 分配和管理视频帧缓冲区
- 物理地址到虚拟地址的映射
- 缓冲区池管理

相关 API：

- `kd_mpi_vb_set_config()` - 设置VB配置
- `kd_mpi_vb_init()` - 初始化VB
- `kd_mpi_vb_exit()` - 退出VB
- `kd_mpi_sys_mmap()` - 内存映射
- `kd_mpi_sys_munmap()` - 解除映射

---
感谢阅读！
