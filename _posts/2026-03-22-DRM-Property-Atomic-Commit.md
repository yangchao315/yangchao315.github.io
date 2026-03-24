---
layout: post
title: "DRM Property 与 Atomic Commit 机制深度分析"
date: 2026-03-22
tags: [DRM, Display]
comments: true
author: yangchao
---

<!-- more -->

- [一、为什么需要 Property？](#一为什么需要-property)
- [二、Property 类型详解](#二property-类型详解)
  - [2.1 RANGE 属性](#21-range-属性)
  - [2.2 ENUM 属性](#22-enum-属性)
  - [2.3 BITMASK 属性](#23-bitmask-属性)
  - [2.4 BLOB 属性](#24-blob-属性)
- [三、Property 创建与绑定](#三property-创建与绑定)
- [四、Atomic Commit 机制](#四atomic-commit-机制)
  - [4.1 核心 API](#41-核心-api)
  - [4.2 状态结构体](#42-状态结构体)
  - [4.3 Check vs Commit 流程](#43-check-vs-commit-流程)
  - [4.4 Non-blocking Commit 与 Fence](#44-non-blocking-commit-与-fence)
  - [4.5 Page Flip 在 Atomic 模式](#45-page-flip-在-atomic-模式)
- [五、典型驱动实现框架](#五典型驱动实现框架)
- [六、总结](#六总结)

---

## 一、为什么需要 Property？

DRM (Direct Rendering Manager) 中的 **Property 系统**是用户空间配置显示硬件的核心机制。在传统 legacy 模式中，应用程序直接操作具体的硬件寄存器或结构体，但这种做法存在诸多问题：

1. **硬件抽象**：不同厂商的 GPU/IP 有不同的寄存器接口
2. **原子操作**：所有配置变更必须**全部成功**或**全部失败**，避免中间状态导致显示异常
3. **统一接口**：无论是 Intel/AMD/ARM GPU，都可以通过统一的 property 接口进行配置

**Property 的本质**：将硬件参数抽象为**键值对**，用户空间只关心语义（如 "rotation"），而不关心底层如何实现。

```c
// 每个 DRM 对象 (CRTC/Plane/Connector) 都有一个属性列表
struct drm_object_properties {
    int count;
    struct drm_property *properties[DRM_OBJECT_MAX_PROPERTY];
    uint64_t values[DRM_OBJECT_MAX_PROPERTY];
};
```

---

## 二、Property 类型详解

### 2.1 RANGE 属性

用于**无符号整数范围**，内核会验证用户空间设置的值是否在范围内。

```c
// 创建 RANGE 属性
struct drm_property *drm_property_create_range(
    struct drm_device *dev,
    u32 flags, 
    const char *name,
    uint64_t min, 
    uint64_t max);

// 实际示例：SRC_X, SRC_Y, SRC_W, SRC_H
prop = drm_property_create_range(dev, DRM_MODE_PROP_ATOMIC,
            "SRC_X", 0, UINT_MAX);
dev->mode_config.prop_src_x = prop;
```

### 2.2 ENUM 属性

用于**枚举值**，每个值有一个符号名称。

```c
// 枚举值定义
struct drm_prop_enum_list {
    int type;
    const char *name;
};

// DPMS 示例
static const struct drm_prop_enum_list drm_dpms_enum_list[] = {
    { DRM_MODE_DPMS_ON, "On" },
    { DRM_MODE_DPMS_STANDBY, "Standby" },
    { DRM_MODE_DPMS_SUSPEND, "Suspend" },
    { DRM_MODE_DPMS_OFF, "Off" },
};

// 创建 ENUM 属性
prop = drm_property_create_enum(dev, 0,
            "DPMS", drm_dpms_enum_list,
            ARRAY_SIZE(drm_dpms_enum_list));
```

### 2.3 BITMASK 属性

用于**位掩码**，允许同时设置多个值（0-63 位）。

```c
// 旋转属性示例
static const struct drm_prop_enum_list rotation_enum[] = {
    { DRM_MODE_ROTATE_0, "rotate-0" },
    { DRM_MODE_ROTATE_90, "rotate-90" },
    { DRM_MODE_ROTATE_180, "rotate-180" },
    { DRM_MODE_ROTATE_270, "rotate-270" },
    { DRM_MODE_REFLECT_X, "reflect-x" },
    { DRM_MODE_REFLECT_Y, "reflect-y" },
};

prop = drm_property_create_bitmask(dev, 0, "rotation",
            rotation_enum, ARRAY_SIZE(rotation_enum));
```

### 2.4 BLOB 属性

用于存储**二进制数据**，如 EDID、MODE 信息、HDR 元数据等。

```c
// 创建 BLOB 属性
struct drm_property_blob *drm_property_create_blob(
    struct drm_device *dev,
    size_t length,
    const void *data);

// 常见用途：MODE_ID (显示模式)
blob = drm_property_create_blob(dev, sizeof(mode), &mode);
connector->mode_blob = blob;
```

---

## 三、Property 创建与绑定

完整的 property 实现流程：

```c
// 1. 驱动初始化阶段创建 property
static int my_driver_init(struct drm_device *dev) {
    struct drm_property *prop;
    
    // 创建各种 property
    prop = drm_property_create_range(dev, 0, "brightness", 0, 100);
    if (!prop) return -ENOMEM;
    priv->brightness_prop = prop;
    
    prop = drm_property_create_enum(dev, 0, "colormap",
            colormap_enum, ARRAY_SIZE(colormap_enum));
    if (!prop) return -ENOMEM;
    priv->colormap_prop = prop;
    
    return 0;
}

// 2. 绑定到具体的 DRM 对象
static int my_driver_bind(struct drm_device *dev) {
    // 绑定到 Plane
    drm_object_attach_property(&plane->base, 
        priv->rotation_prop, DRM_MODE_ROTATE_0);
    drm_object_attach_property(&plane->base,
        priv->alpha_prop, 255);
    
    // 绑定到 CRTC
    drm_object_attach_property(&crtc->base,
        dev->mode_config.prop_active, 1);
        
    // 绑定到 Connector
    drm_object_attach_property(&connector->base,
        dev->mode_config.dpms_property, DRM_MODE_DPMS_ON);
}
```

**标准 Property 速查表**：

| 对象 | 标准 Property |
|------|---------------|
| **CRTC** | ACTIVE, MODE_ID, VRR_ENABLED, GAMMA_LUT, DEGAMMA_LUT, CTM |
| **Plane** | FB_ID, CRTC_ID, SRC_X/Y/W/H, CRTC_X/Y/W/H, rotation, alpha, color_encoding |
| **Connector** | EDID, DPMS, link-status, scaling_mode, content_protection, max_bpc |

---

## 四、Atomic Commit 机制

### 4.1 核心 API

```c
// 1. 仅检查配置合法性，不提交
int drm_atomic_check_only(struct drm_atomic_state *state);

// 2. 阻塞式原子提交
int drm_atomic_commit(struct drm_atomic_state *state);

// 3. 非阻塞式原子提交
int drm_atomic_nonblocking_commit(struct drm_atomic_state *state);
```

### 4.2 状态结构体

Atomic 将整体状态拆分为**每个对象独立的状态结构**：

```
drm_atomic_state
    ├── drm_crtc_state     (每个 CRTC 的状态)
    ├── drm_plane_state   (每个 Plane 的状态)
    └── drm_connector_state (每个 Connector 的状态)
```

驱动可以通过 `drm_private_state` 扩展这些结构：

```c
// 驱动私有状态扩展示例
struct my_crtc_private_state {
    struct drm_private_state obj;
    u32 hw_config_reg;
    bool pipeline_enabled;
};

struct my_crtc_state {
    struct drm_crtc_state base;  // 必须内嵌
    struct my_crtc_private_state private;
};
```

### 4.3 Check vs Commit 流程

```
Userspace (Wayland Compositor/Mutter)
    │
    ▼ 创建 drm_atomic_state 并填充配置
drm_atomic_check_only()
    │
    ▼ 调用驱动 atomic_check() 回调
驱动 atomic_check():
    ├── 检查模式是否有效
    ├── 验证硬件能力
    ├── 计算带宽需求
    └── 返回 0 表示合法，-EINVAL 表示非法
    │
    ▼ 如返回错误，用户空间收到错误，终止
    │
drm_atomic_commit()
    │
    ▼ 调用驱动 atomic_commit() 回调
驱动 atomic_commit():
    ├── 禁用受影响的 CRTC/Plane
    ├── 配置新的显示模式
    ├── 启用新的 Plane 配置
    └── 触发 VBlank 等待
    │
    ▼ VBlank 到达后切换缓冲区
flip_done 事件 → 释放旧 framebuffer
```

### 4.4 Non-blocking Commit 与 Fence

非阻塞提交是现代显示系统的核心，支持多个应用同时提交而不阻塞渲染进程。

```c
// 非阻塞提交流程
int my_atomic_commit(struct drm_device *dev,
                     struct drm_atomic_state *state,
                     bool nonblock) {
    int ret;
    
    if (nonblock) {
        // 1. 初始化 commit 跟踪
        ret = drm_atomic_helper_setup_commit(state, DRM_MODE_PAGE_FLIP_ASYNC);
        if (ret) return ret;
        
        // 2. 提交到硬件（不等待 VBlank）
        ret = my_hw_commit(state);
        if (ret) return ret;
        
        // 3. 完成后通过 fence 通知
        return 0;
    }
    
    // 阻塞模式
    drm_atomic_helper_commit_modeset_disables(dev, state);
    drm_atomic_helper_commit_modesets(dev, state);
    drm_atomic_helper_commit_planes(dev, state, 0);
    drm_atomic_helper_wait_for_vblanks(dev, state);
}
```

**关键辅助函数**：

| 函数 | 用途 |
|------|------|
| `drm_atomic_helper_wait_for_fences()` | 等待所有 fence 信号 |
| `drm_atomic_helper_wait_for_vblanks()` | 等待所有 CRTC 的 VBlank |
| `drm_atomic_helper_wait_for_flip_done()` | 等待页面翻转完成 |

### 4.5 Page Flip 在 Atomic 模式

```c
// 页面翻转 (atomic 方式)
drm_atomic_commit() {
    // 在 plane_state 中指定新的 FB
    plane_state->fb = new_fb;
    plane_state->crtc = crtc;
    plane_state->crtc_x = 0;
    plane_state->crtc_y = 0;
    plane_state->crtc_w = mode->hdisplay;
    plane_state->crtc_h = mode->vdisplay;
    plane_state->src_x = 0;
    plane_state->src_y = 0;
    plane_state->src_w = mode->hdisplay << 16;
    plane_state->src_h = mode->vdisplay << 16;
    
    // 内核自动在下一个 VBlank 切换缓冲区
}

// flip 事件结构
struct drm_crtc_commit {
    struct drm_crtc *crtc;
    struct completion hw_done;      // 硬件操作完成
    struct completion flip_done;    // 页面翻转完成
    struct dma_fence *fence;       // 同步 fence
};
```

---

## 五、典型驱动实现框架

```c
static const struct drm_mode_config_funcs my_mode_config_funcs = {
    .fb_create = my_fb_create,
    .atomic_check = my_atomic_check,
    .atomic_commit = my_atomic_commit,
};

// atomic_check 实现
static int my_atomic_check(struct drm_device *dev, 
                           struct drm_atomic_state *state) {
    int ret;
    
    // 1. 调用 helper 检查基础配置
    ret = drm_atomic_helper_check(dev, state);
    if (ret) return ret;
    
    // 2. 驱动特定检查
    for_each_new_crtc_in_state(state, crtc, old_crtc, i) {
        struct drm_crtc_state *crtc_state = state->crtcs[i].new_state;
        
        // 检查模式是否支持
        if (!my_crtc_mode_valid(crtc_state->mode))
            return -EINVAL;
            
        // 检查带宽
        if (my_bandwidth_exceeds(crtc_state))
            return -ENOSPC;
    }
    
    return 0;
}

// atomic_commit 实现
static int my_atomic_commit(struct drm_device *dev,
                            struct drm_atomic_state *state,
                            bool nonblock) {
    // 使用 helper 函数
    if (nonblock) {
        return drm_atomic_helper_commit(dev, state, true);
    }
    
    drm_atomic_helper_commit_modeset_disables(dev, state);
    drm_atomic_helper_commit_modesets(dev, state);
    drm_atomic_helper_commit_planes(dev, state, 0);
    drm_atomic_helper_wait_for_vblanks(dev, state);
    
    return 0;
}
```

---

## 六、总结

| 概念 | 核心要点 |
|------|---------|
| **Property** | 用户空间配置硬件的抽象接口，支持 RANGE/ENUM/BITMASK/BLOB 等类型 |
| **Atomic Commit** | 所有配置变更原子性应用，避免中间状态 |
| **Check vs Commit** | Check 验证合法性，Commit 提交到硬件 |
| **Non-blocking** | 通过 fence 机制实现异步提交，提高并发性能 |
| **Page Flip** | 在 VBlank 时切换缓冲区，通过 flip_done 事件通知 |

**参考资料**：
- Linux Kernel DRM 文档: https://www.kernel.org/doc/html/latest/gpu/drm-kms.html
- Property 头文件: `include/drm/drm_property.h`
- Atomic 核心: `drivers/gpu/drm/drm_atomic.c`
- Atomic Helper: `drivers/gpu/drm/drm_atomic_helper.c`

---

## 感谢阅读！

