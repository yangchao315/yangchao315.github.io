---
layout: post
title: "Meson Plane 深度分析"
date:   2026-03-02
tags: [DRM, Meson, Plane, Amlogic]
comments: true
author: yangchao
---

<!-- more -->
- [Meson Plane 深度分析](#meson-plane-深度分析)
  - [Plane 基础概念](#plane-基础概念)
  - [Meson Plane 结构体](#meson-plane-结构体)
  - [Atomic Check 流程](#atomic-check-流程)
  - [Atomic Update 实现](#atomic-update-实现)
  - [Atomic Disable 处理](#atomic-disable-处理)
  - [AFBC 支持](#afbc-支持)
  - [性能优化要点](#性能优化要点)

# Meson Plane 深度分析

在 Linux DRM (Direct Rendering Manager) 子系统中，Plane 是显示合成的核心组件。Meson DRM 驱动通过 `meson_plane.c` 实现了对 Amlogic SoC 显示平面的完整支持。本文将深入分析 `meson_plane.c` 的关键实现细节。

## Plane 基础概念

DRM Plane 代表一个可以独立控制的显示层，通常包括：
- **Primary Plane**: 主显示平面，通常用于桌面内容
- **Cursor Plane**: 光标平面，用于鼠标光标
- **Overlay Planes**: 叠加平面，用于视频播放等特殊用途

在 Meson SoC 中，主要使用 OSD1 (On-Screen Display 1) 作为 Primary Plane，支持 AFBC (ARM Frame Buffer Compression) 等高级特性。

## Meson Plane 结构体

```c
struct meson_plane {
	struct drm_plane base;
	struct meson_drm *priv;
	bool enabled;
};
```

- `base`: 继承自 DRM 核心的 plane 结构
- `priv`: 指向 Meson DRM 私有数据的指针
- `enabled`: 平面启用状态标志

## Atomic Check 流程

`meson_plane_atomic_check` 函数负责验证 plane 配置的合法性：

```c
static int meson_plane_atomic_check(struct drm_plane *plane,
				    struct drm_atomic_state *state)
{
	struct drm_plane_state *new_plane_state = drm_atomic_get_new_plane_state(state, plane);
	struct drm_crtc_state *crtc_state;

	if (!new_plane_state->crtc)
		return 0;

	crtc_state = drm_atomic_get_crtc_state(state, new_plane_state->crtc);
	if (IS_ERR(crtc_state))
		return PTR_ERR(crtc_state);

	/*
	 * Only allow :
	 * - Upscaling up to 5x, vertical and horizontal
	 * - Final coordinates must match crtc size
	 */
	return drm_atomic_helper_check_plane_state(new_plane_state,
						   crtc_state,
						   FRAC_16_16(1, 5),
						   DRM_PLANE_HELPER_NO_SCALING,
						   false, true);
}
```

关键限制：
- **缩放限制**: 最多支持 5 倍放大（`FRAC_16_16(1, 5)`）
- **坐标匹配**: 最终坐标必须与 CRTC 尺寸匹配
- **无缩小**: 不支持缩小操作（`DRM_PLANE_HELPER_NO_SCALING`）

## Atomic Update 实现

`meson_plane_atomic_update` 是核心函数，负责实际的硬件配置：

### 坐标更新 (Update Coordinates)
- 设置源矩形 (`src_x`, `src_y`, `src_w`, `src_h`)
- 设置目标矩形 (`crtc_x`, `crtc_y`, `crtc_w`, `crtc_h`)
- 配置 VIU (Video Input Unit) 寄存器

### 格式更新 (Update Formats)
- 解析 framebuffer 像素格式
- 配置色彩空间转换 (CSC)
- 设置字节对齐和 stride

### 缓冲区更新 (Update Buffer)
- 获取 GEM buffer 物理地址
- 配置 Canvas ID (通过 `meson_canvas_config`)
- 设置 AFBC 相关寄存器（如果启用）

### 平面启用 (Enable Plane)
- 配置 OSD1 控制寄存器
- 启用 VIU 通道
- 更新 VPP (Video Post Processing) 混合设置

## Atomic Disable 处理

`meson_plane_atomic_disable` 负责安全地禁用平面：

```c
static void meson_plane_atomic_disable(struct drm_plane *plane,
				       struct drm_atomic_state *state)
{
	struct meson_plane *meson_plane = to_meson_plane(plane);
	struct meson_drm *priv = meson_plane->priv;

	if (priv->afbcd.ops) {
		priv->afbcd.ops->reset(priv);
		priv->afbcd.ops->disable(priv);
	}

	/* Disable OSD1 */
	if (meson_vpu_is_compatible(priv, VPU_COMPATIBLE_G12A))
		writel_bits_relaxed(VIU_OSD1_POSTBLD_SRC_OSD1, 0,
				    priv->io_base + _REG(OSD1_BLEND_SRC_CTRL));
	else
		writel_bits_relaxed(VPP_OSD1_POSTBLEND, 0,
				    priv->io_base + _REG(VPP_MISC));

	meson_plane->enabled = false;
	priv->viu.osd1_enabled = false;
}
```

关键步骤：
1. **AFBC 复位**: 如果启用了 AFBC，先复位并禁用 AFBC 模块
2. **OSD1 禁用**: 根据 SoC 版本（G12A 或其他）禁用相应的混合控制
3. **状态更新**: 更新软件状态标志

## AFBC 支持

Meson Plane 对 ARM Frame Buffer Compression (AFBC) 提供完整支持：

- **AFBC 检测**: 通过 framebuffer modifier 自动检测 AFBC 格式
- **硬件配置**: 配置 AFBC 解码器寄存器
- **性能优势**: 减少内存带宽占用，提升系统性能
- **兼容性**: 支持 GXM、G12A 等不同 SoC 的 AFBC 实现

AFBC 相关操作通过 `priv->afbcd.ops` 接口抽象，支持不同硬件版本的差异。

## 性能优化要点

1. **Canvas 管理**: 使用 `meson_canvas_alloc/config` 高效管理显示缓冲区
2. **内存对齐**: 64 字节对齐的 stride 和页对齐的 buffer size
3. **原子操作**: 所有配置都在 atomic commit 中完成，避免撕裂
4. **硬件加速**: 充分利用 VIU、VPP 等硬件模块进行色彩转换和混合

Meson Plane 的实现充分考虑了嵌入式系统的性能和功耗要求，通过硬件加速和内存优化提供了流畅的显示体验。

---
感谢阅读！