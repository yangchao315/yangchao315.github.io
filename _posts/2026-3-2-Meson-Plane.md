---
layout: post
title: "Meson Plane 深度解析"
date:   2026-3-2
tags:
    - Amlogic
    - DRM
    - Meson
comments: true
author: yangchao
---

本文基于 `linux-mainline-5.18/drivers/gpu/drm/meson/meson_plane.c` 进行分析，聚焦 Meson DRM 中 primary plane 的检查、配置、使能和 AFBC 路径。

<!-- more -->
- [Meson Plane 深度解析](#meson-plane-深度解析)
  - [1. 文件定位与职责](#1-文件定位与职责)
  - [2. 关键数据结构](#2-关键数据结构)
  - [3. Plane 原子检查：`meson_plane_atomic_check`](#3-plane-原子检查meson_plane_atomic_check)
  - [4. Plane 更新主路径：`meson_plane_atomic_update`](#4-plane-更新主路径meson_plane_atomic_update)
    - [4.1 AFBC 路径选择](#41-afbc-路径选择)
    - [4.2 像素格式与 alpha 处理](#42-像素格式与-alpha-处理)
    - [4.3 缩放器参数与隔行模式](#43-缩放器参数与隔行模式)
    - [4.4 源/目标窗口寄存器打包](#44-源目标窗口寄存器打包)
    - [4.5 Buffer 地址与 stride 更新](#45-buffer-地址与-stride-更新)
    - [4.6 首次使能行为](#46-首次使能行为)
  - [5. Plane 关闭路径：`meson_plane_atomic_disable`](#5-plane-关闭路径meson_plane_atomic_disable)
  - [6. 格式与 modifier 协商机制](#6-格式与-modifier-协商机制)
    - [6.1 支持的像素格式](#61-支持的像素格式)
    - [6.2 SoC 相关 AFBC modifier 列表](#62-soc-相关-afbc-modifier-列表)
    - [6.3 `format_mod_supported` 判定流程](#63-format_mod_supported-判定流程)
  - [7. Plane 创建与注册：`meson_plane_create`](#7-plane-创建与注册meson_plane_create)
  - [8. 关键实现细节总结](#8-关键实现细节总结)
  - [9. 调试建议](#9-调试建议)
  - [10. 跨文件时序图（`meson_plane.c` + `meson_crtc.c` + `meson_viu.c`）](#10-跨文件时序图meson_planec--meson_crtcc--meson_viuc)
    - [10.1 Atomic 提交到 VSync 生效（OSD1）](#101-atomic-提交到-vsync-生效osd1)
    - [10.2 AFBC 路径切换关键时序](#102-afbc-路径切换关键时序)
    - [10.3 CRTC enable/disable 与 VIU 初始化关系](#103-crtc-enabledisable-与-viu-初始化关系)

## Meson Plane 深度解析

## 1. 文件定位与职责

`meson_plane.c` 是 Meson DRM 驱动里 primary plane 的核心实现，主要完成三件事：

1. 对 plane 原子状态做约束检查（缩放能力、坐标合法性）。
2. 将 framebuffer 和目标显示参数转换为 `priv->viu.*` 一组“待刷寄存器语义”的字段。
3. 定义 plane 的格式/压缩格式能力，并在驱动初始化时注册 plane。

这个文件本身不直接把每个寄存器都立即写硬件，而是配合 Meson DRM 的整体提交路径完成一次 atomic commit 的生效。

## 2. 关键数据结构

核心私有结构很简单：

```c
struct meson_plane {
	struct drm_plane base;
	struct meson_drm *priv;
	bool enabled;
};
```

- `base`：DRM 核心 plane 对象。
- `priv`：指向驱动总私有结构，后续大量字段通过 `priv->viu`、`priv->afbcd` 更新。
- `enabled`：记录 plane 是否已进入使能状态，避免重复做首次使能流程。

## 3. Plane 原子检查：`meson_plane_atomic_check`

该函数调用 `drm_atomic_helper_check_plane_state()` 做统一校验，参数里约束了缩放能力：

- 最小缩放比：`FRAC_16_16(1, 5)`，即最多放大到 5x（注释也说明了 upscaling up to 5x）。
- 最大缩放比：`DRM_PLANE_HELPER_NO_SCALING`，表示不允许超出 helper 可接受上限。
- 同时要求和 CRTC 状态匹配，避免超出显示窗口。

简化理解：用户态提交 plane state 后，先在这里卡掉非法缩放和非法坐标，再进入 update。

## 4. Plane 更新主路径：`meson_plane_atomic_update`

这是全文件最核心函数，逻辑按“格式路径选择 -> 缩放参数 -> 窗口寄存器 -> 缓冲区地址 -> 使能状态”推进。

### 4.1 AFBC 路径选择

先判断当前 SoC 与 framebuffer modifier：

- SoC 需为 `GXM` 或 `G12A`。
- modifier 需匹配 `DRM_FORMAT_MOD_ARM_AFBC(...)` 且位于 `MESON_MOD_AFBC_VALID_BITS` 范围。

满足则 `priv->viu.osd1_afbcd = true`，否则走线性（Linear）路径。

AFBC 与非 AFBC 下会设置不同端序/路径控制位：

- G12A AFBC：设置 `MESON_G12A_AFBCD_OUT_ADDR`、`OSD_PENDING_STAT_CLEAN`、`VIU_OSD1_CFG_SYN_EN` 等。
- GXM AFBC：设置 `OSD_DPATH_MALI_AFBCD`。
- 非 AFBC：清除 AFBC 路径位并走普通 OSD canvas 读取。

### 4.2 像素格式与 alpha 处理

根据 `fb->format->format` 选择 OSD block mode 和 color matrix：

- `XRGB/ARGB8888` -> `OSD_BLK_MODE_32`，再区分 ARGB/ABGR matrix。
- `RGB888` -> `OSD_BLK_MODE_24`。
- `RGB565` -> `OSD_BLK_MODE_16`。

alpha 处理点：

- `XRGB/XBGR`：没有有效 alpha，驱动置 `OSD_REPLACE_EN`，用固定 `0xFF` 替代像素 alpha。
- `ARGB/ABGR`：清 `OSD_REPLACE_EN`，使用 framebuffer 自带 alpha。

### 4.3 缩放器参数与隔行模式

代码先取源/目标尺寸：

```c
src_w = fixed16_to_int(new_state->src_w);
src_h = fixed16_to_int(new_state->src_h);
dst_w = new_state->crtc_w;
dst_h = new_state->crtc_h;
```

然后算相位步进：

- `hf_phase_step = ((src_w << 18) / dst_w) << 6`
- `vf_phase_step = (src_h << 20) / dst_h`

隔行输出（`DRM_MODE_FLAG_INTERLACE`）时，目标高度与 y 坐标会按 field 处理（减半），并启用 `VSC_PROG_INTERLACE` 相关参数。代码注释说明了一个关键点：隔行场切换可借助垂直 scaler 的配置完成。

缩放器使能策略：

- 垂直方向：`src_h != dst_h` 时启用。
- 水平方向：`src_w != dst_w` 时启用。
- 只要任一方向缩放，`osd_sc_ctrl0` 开启 scaler path。

### 4.4 源/目标窗口寄存器打包

源码明确了窗口寄存器格式：

`(x2 << 16) | x1`，其中 `x2` 是“排他上界”，所以通常写 `(end - 1)`。

因此你会看到：

- 源窗口来自 `new_state->src.{x1,x2,y1,y2}`（16.16 定点转整数）。
- 目标窗口来自 `dest.{x1,x2,y1,y2}`（CRTC 坐标）。

G12A 下还会更新 blend scope 与 blend size，保证后级混合单元范围一致。

### 4.5 Buffer 地址与 stride 更新

通过 `drm_fb_cma_get_gem_obj(fb, 0)` 获取 CMA GEM 对象后，填入：

- `priv->viu.osd1_addr = gem->paddr`
- `priv->viu.osd1_stride = fb->pitches[0]`
- `priv->viu.osd1_width/height = fb->width/height`

AFBC 下还会记录：

- `priv->afbcd.modifier`
- `priv->afbcd.format`

并在 G12A 计算 AFBCD 输出行跨度（`meson_g12a_afbcd_line_stride()`），写入 `osd1_blk2_cfg4`。

### 4.6 首次使能行为

首次打开 plane（`!meson_plane->enabled`）时：

- GXM/GXL 会先执行 `meson_viu_osd1_reset(priv)`。
- 再置 `meson_plane->enabled = true` 和 `priv->viu.osd1_enabled = true`。

这里的 reset 是很典型的硬件稳定性处理，避免前序状态残留。

## 5. Plane 关闭路径：`meson_plane_atomic_disable`

disable 顺序很清晰：

1. 若 AFBCD ops 存在，先 `reset` 再 `disable`。
2. 关闭 OSD1 后混合输入：
   - G12A：清 `OSD1_BLEND_SRC_CTRL` 对应位。
   - 非 G12A：清 `VPP_MISC` 的 `VPP_OSD1_POSTBLEND`。
3. 清软件态：
   - `meson_plane->enabled = false`
   - `priv->viu.osd1_enabled = false`

## 6. 格式与 modifier 协商机制

### 6.1 支持的像素格式

`supported_drm_formats[]` 提供 6 种 RGB 格式：

- `ARGB8888`
- `ABGR8888`
- `XRGB8888`
- `XBGR8888`
- `RGB888`
- `RGB565`

### 6.2 SoC 相关 AFBC modifier 列表

- `format_modifiers_default`：仅 `LINEAR`。
- `format_modifiers_afbc_gxm`：16x16 + YTR/SPARSE/SPLIT 的组合。
- `format_modifiers_afbc_g12a`：支持 16x16 与 32x8 多组合（含对性能的约束注释）。

这体现了不同代际 Meson VPU 对 AFBC 的硬件能力差异。

### 6.3 `format_mod_supported` 判定流程

`meson_plane_format_mod_supported()` 的判定链：

1. `INVALID` 直接拒绝。
2. `LINEAR` 直接接受。
3. 若 SoC 不是 GXM/G12A，拒绝 AFBC。
4. modifier 若含非法位（超出 `MESON_MOD_AFBC_VALID_BITS`）拒绝。
5. modifier 必须出现在 `plane->modifiers` 列表中。
6. 最后调用 `priv->afbcd.ops->supported_fmt()` 做 AFBC 细粒度格式验证。

这套机制把“硬件代际能力 + 驱动声明能力 + AFBCD 子模块能力”三层都校验到了。

## 7. Plane 创建与注册：`meson_plane_create`

创建流程：

1. `devm_kzalloc()` 申请 `struct meson_plane`。
2. 根据 SoC 选择 modifier 表（default / gxm / g12a）。
3. 调用 `drm_universal_plane_init()` 注册为 `DRM_PLANE_TYPE_PRIMARY`，名称 `meson_primary_plane`。
4. `drm_plane_helper_add()` 绑定 helper 回调：
   - `atomic_check`
   - `atomic_update`
   - `atomic_disable`
5. 设置不可变 zpos 为 1：`drm_plane_create_zpos_immutable_property(plane, 1)`。
6. 保存 `priv->primary_plane = plane`。

## 8. 关键实现细节总结

1. Meson plane 是典型的“DRM atomic 状态 -> 私有寄存器语义缓存”的设计。
2. AFBC 支持不是全平台统一能力，代码按 SoC 精细分支。
3. 缩放器和隔行处理耦合较深，尤其垂直 scaler 在 interlace 下承担场切换相关功能。
4. XRGB 与 ARGB 的 alpha 语义在硬件侧显式区分，避免 UI 合成透明度错误。
5. 首次使能时的 OSD reset 是稳定性关键点。

## 9. 调试建议

1. 先确认 userspace 传入的是 `LINEAR` 还是 `AFBC` modifier。
2. 若 AFBC 不生效，优先检查：
   - SoC 兼容分支是否命中；
   - modifier 是否在 `plane->modifiers` 中；
   - `supported_fmt()` 返回值。
3. 若出现缩放异常，核对：
   - `src_w/src_h` 与 `crtc_w/crtc_h`；
   - `hf_phase_step/vf_phase_step`；
   - interlace 场景下 `dest.y` 与 `dst_h` 的减半逻辑。
4. 若首帧异常，关注 `meson_viu_osd1_reset()` 是否被执行以及执行时机。

## 10. 跨文件时序图（`meson_plane.c` + `meson_crtc.c` + `meson_viu.c`）

下面把三个文件放在一条时序线上看，会更容易理解“为什么 `atomic_update` 里只写 `priv->viu`，而真正寄存器写在中断里做”。

### 10.1 Atomic 提交到 VSync 生效（OSD1）

```text
Userspace
  | drmModeAtomicCommit()
  v
DRM Atomic Core
  |-> meson_plane_atomic_check()           [meson_plane.c]
  |-> meson_plane_atomic_update()          [meson_plane.c]
  |     - 计算格式/缩放/窗口/stride
  |     - 写入 priv->viu.osd1_* 缓存字段
  |
  |-> meson_crtc_atomic_begin()            [meson_crtc.c]
  |     - 暂存 vblank event
  |
  |-> meson_crtc_atomic_flush()            [meson_crtc.c]
  |     - priv->viu.osd1_commit = true
  v
等待下一次 VSync IRQ
  v
meson_crtc_irq()                           [meson_crtc.c]
  |- if (osd1_enabled && osd1_commit):
  |    - writel(VIU_OSD1_CTRL_STAT/STAT2/BLK0_CFGx...)
  |    - writel(OSD scaler寄存器)
  |    - 非AFBC: meson_canvas_config(...)
  |    - enable_osd1()
  |    - AFBC时: afbcd reset/setup/enable
  |    - osd1_commit = false
  |
  |- drm_crtc_handle_vblank()
  |- drm_crtc_send_vblank_event()
  v
硬件在场同步边界完成切换，避免撕裂
```

关键结论：  
`meson_plane_atomic_update()` 负责“准备参数”，`meson_crtc_irq()` 负责“在安全时刻落寄存器”。

### 10.2 AFBC 路径切换关键时序

```text
meson_plane_atomic_update()                [meson_plane.c]
  |- 根据 SoC + fb->modifier 判定 osd1_afbcd
  |- 设置 osd1_blk0_cfg/osd1_ctrl_stat2 等路径位
  |- 记录 priv->afbcd.{modifier,format}
  |- G12A: 计算 osd1_blk2_cfg4 (line stride)
  v
meson_crtc_irq()                           [meson_crtc.c]
  |- if osd1_afbcd == true:
  |    - enable_osd1_afbc() (函数指针, SoC相关)
  |      - G12A: meson_crtc_g12a_enable_osd1_afbc()
  |              -> meson_viu_g12a_enable_osd1_afbc() [meson_viu.c]
  |      - GXM : meson_viu_gxm_enable_osd1_afbc()     [meson_viu.c]
  |    - priv->afbcd.ops->reset/setup/enable
  |
  |- else:
  |    - disable_osd1_afbc()
  |    - priv->afbcd.ops->reset/disable
  v
OSD1 在 AFBC 与 Linear 两条读路径间切换
```

关键结论：  
AFBC 不是单点开关，而是 `plane` 预配置 + `crtc irq` 生效 + `viu` 路径控制 + `afbcd ops` 解码模块配置共同完成。

### 10.3 CRTC enable/disable 与 VIU 初始化关系

```text
Driver bind/init 阶段
  -> meson_viu_init()                      [meson_viu.c]
     - 关 OSD1/OSD2
     - 初始化 FIFO / alpha replace / blend
     - 关闭 AFBC 默认路径
     - 清 osd1_enabled/osd1_commit

模式设置启用阶段
  -> meson_crtc_atomic_enable()            [meson_crtc.c]
     或 meson_g12a_crtc_atomic_enable()
     - 配置 VPP blend 输出尺寸
     - drm_crtc_vblank_on()

提交显示更新
  -> meson_plane_atomic_update() + meson_crtc_atomic_flush()
  -> meson_crtc_irq() 在 VSync 应用新参数

关闭阶段
  -> meson_crtc_atomic_disable()
     - drm_crtc_vblank_off()
     - 清 osd1/vd1 enabled/commit
     - 关闭 postblend 路径
  -> meson_plane_atomic_disable() (按提交状态触发)
     - AFBCD reset/disable
```

关键结论：  
`meson_viu_init()` 解决“初始硬件基线”，`meson_crtc_*` 解决“显示管线状态机”，`meson_plane_*` 解决“每帧图层参数”。

---
感谢阅读！
