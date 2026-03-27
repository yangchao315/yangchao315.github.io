---
layout: post
title: "Linux DRM 驱动中的 Atomic Commit 机制详解"
date:   2026-03-27
tags: [DRM]
comments: true
author: yangchao
---

<!-- more -->
- [Linux DRM 驱动中的 Atomic Commit 机制详解](#linux-drm-驱动中的-atomic-commit-机制详解)
  - [1. 背景与动机](#1-背景与动机)
  - [2. 核心数据结构](#2-核心数据结构)
    - [2.1 drm_atomic_state](#21-drm_atomic_state)
    - [2.2 各对象的 state 结构](#22-各对象的-state-结构)
  - [3. Atomic Commit 流程](#3-atomic-commit-流程)
    - [3.1 用户空间接口](#31-用户空间接口)
    - [3.2 内核处理流程](#32-内核处理流程)
    - [3.3 check 阶段](#33-check-阶段)
    - [3.4 commit 阶段](#34-commit-阶段)
  - [4. 驱动实现要点](#4-驱动实现要点)
    - [4.1 atomic_check 回调](#41-atomic_check-回调)
    - [4.2 atomic_commit_tail 回调](#42-atomic_commit_tail-回调)
  - [5. 异步提交与 commit_work](#5-异步提交与-commit_work)
  - [6. 与旧版 legacy 接口的对比](#6-与旧版-legacy-接口的对比)
  - [7. 常见问题排查](#7-常见问题排查)

# Linux DRM 驱动中的 Atomic Commit 机制详解

DRM（Direct Rendering Manager）子系统从 Linux 4.2 起引入了 atomic modeset 机制，用于原子地更新显示硬件状态，避免中间状态的撕裂（tearing）和不一致。本文深入分析 atomic commit 的设计思路、核心数据结构以及驱动实现方式。

## 1. 背景与动机

传统的 legacy KMS 接口（`drmModeSetCrtc`、`drmModePageFlip` 等）每次只能更新单一对象（CRTC/Plane/Connector），多个对象的协同更新无法保证原子性，容易产生以下问题：

- 屏幕闪烁（中间帧显示了不完整的配置）
- 多 CRTC 同步困难（多屏场景）
- 驱动内部难以做硬件校验（check 与 commit 分离不清晰）

Atomic modeset 将一次显示更新封装为一个"事务"：先 check，再 commit，要么全部成功，要么回滚到原有状态。

```
用户空间                 内核 DRM 核心               驱动
   |                         |                        |
   |  DRM_IOCTL_MODE_ATOMIC  |                        |
   |------------------------>|                        |
   |                         |  atomic_check()        |
   |                         |----------------------->|
   |                         |  <------ ok/err -------|
   |                         |  atomic_commit()       |
   |                         |----------------------->|
   |                         |  <------ done ---------|
   |<------------------------|                        |
```

## 2. 核心数据结构

### 2.1 drm_atomic_state

`drm_atomic_state` 是整个事务的容器，记录了本次 commit 涉及的所有对象（CRTC、Plane、Connector）的新旧状态。

```c
/* include/drm/drm_atomic.h */
struct drm_atomic_state {
    struct kref ref;
    struct drm_device *dev;

    bool allow_modeset : 1;
    bool legacy_cursor_update : 1;
    bool async_update : 1;
    bool duplicated : 1;

    struct __drm_planes_state   *planes;
    struct __drm_crtcs_state    *crtcs;
    struct __drm_connectors_state *connectors;

    int num_connector;
    struct drm_modeset_acquire_ctx *acquire_ctx;

    struct drm_crtc_commit *fake_commit;
    struct work_struct commit_work;
};
```

每个条目同时保存旧 state 指针（`old_state`）和新 state 指针（`new_state`），便于在 check 失败时恢复。

### 2.2 各对象的 state 结构

DRM 中三类可更新对象各自有对应的 state 结构：

```c
/* CRTC state */
struct drm_crtc_state {
    struct drm_crtc *crtc;
    bool enable;
    bool active;
    bool planes_changed : 1;
    bool mode_changed : 1;
    bool active_changed : 1;
    bool connectors_changed : 1;

    struct drm_display_mode mode;
    struct drm_display_mode adjusted_mode;

    u32 plane_mask;
    u32 connector_mask;
    u32 encoder_mask;

    struct drm_crtc_commit *commit;
    /* ... */
};

/* Plane state */
struct drm_plane_state {
    struct drm_plane *plane;
    struct drm_crtc *crtc;
    struct drm_framebuffer *fb;

    /* 源矩形（Q16.16 定点数，像素坐标）*/
    uint32_t src_x, src_y, src_w, src_h;
    /* 目标矩形（整数，屏幕坐标）*/
    int32_t crtc_x, crtc_y;
    uint32_t crtc_w, crtc_h;

    /* rotation / z-order / alpha */
    unsigned int rotation;
    unsigned int zpos;
    u16 alpha;
    /* ... */
};
```

驱动可以通过继承这些结构扩展私有字段：

```c
struct meson_plane_state {
    struct drm_plane_state base; /* 必须放第一位 */
    /* 驱动私有字段 */
    bool premult_alpha;
};
```

## 3. Atomic Commit 流程

### 3.1 用户空间接口

用户空间通过 `DRM_IOCTL_MODE_ATOMIC` ioctl 发起请求，传入若干 `(object_id, property_id, value)` 三元组，由内核统一解析并映射到各对象的 state 字段。

```c
/* libdrm 调用示例 */
drmModeAtomicReqPtr req = drmModeAtomicAlloc();
drmModeAtomicAddProperty(req, plane_id, prop_fb_id,   fb_id);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_id, crtc_id);
drmModeAtomicAddProperty(req, plane_id, prop_src_w,   width << 16);
drmModeAtomicAddProperty(req, plane_id, prop_crtc_w,  width);

uint32_t flags = DRM_MODE_ATOMIC_ALLOW_MODESET;
drmModeAtomicCommit(fd, req, flags, NULL);
drmModeAtomicFree(req);
```

常用 flags：

| Flag | 含义 |
|---|---|
| `DRM_MODE_ATOMIC_TEST_ONLY` | 只做 check，不实际提交 |
| `DRM_MODE_ATOMIC_NONBLOCK` | 非阻塞提交（异步） |
| `DRM_MODE_ATOMIC_ALLOW_MODESET` | 允许 modeset（改分辨率/时序） |
| `DRM_MODE_PAGE_FLIP_EVENT` | 提交完成后发送 page-flip 事件 |

### 3.2 内核处理流程

ioctl 入口为 `drm_mode_atomic_ioctl()`，主要步骤如下：

```
drm_mode_atomic_ioctl()
  └── drm_atomic_helper_commit()
        ├── drm_atomic_helper_prepare_planes()   // 分配/pin framebuffer
        ├── drm_atomic_helper_check()            // 驱动 check 回调
        │     ├── drm_atomic_helper_check_modeset()
        │     └── drm_atomic_helper_check_planes()
        └── drm_atomic_helper_commit_tail()      // 实际硬件更新
              ├── drm_atomic_helper_commit_modeset_disables()
              ├── drm_atomic_helper_commit_planes()
              ├── drm_atomic_helper_commit_modeset_enables()
              └── drm_atomic_helper_wait_for_flip_done()
```

### 3.3 check 阶段

check 阶段不修改任何硬件寄存器，只做合法性验证：

```c
/* drivers/gpu/drm/drm_atomic_helper.c */
int drm_atomic_helper_check(struct drm_device *dev,
                             struct drm_atomic_state *state)
{
    int ret;

    ret = drm_atomic_helper_check_modeset(dev, state);
    if (ret)
        return ret;

    if (dev->mode_config.normalize_zpos) {
        ret = drm_atomic_normalize_zpos(dev, state);
        if (ret)
            return ret;
    }

    ret = drm_atomic_helper_check_planes(dev, state);
    if (ret)
        return ret;

    if (state->legacy_cursor_update)
        state->async_update = !drm_atomic_helper_async_check(dev, state);

    drm_self_refresh_helper_alter_state(state);

    return ret;
}
```

驱动在 `atomic_check` 回调中对私有约束（带宽、格式、缩放比例等）做额外校验，返回非零值即触发回滚。

### 3.4 commit 阶段

commit 阶段负责将新 state 写入硬件，分为三步：

1. **disable** 不再使用的编码器/CRTC
2. **update planes**（调用 `plane->helper_private->atomic_update`）
3. **enable** 新启用的编码器/CRTC

```c
void drm_atomic_helper_commit_planes(struct drm_device *dev,
                                     struct drm_atomic_state *old_state,
                                     uint32_t flags)
{
    struct drm_crtc *crtc;
    struct drm_crtc_state *old_crtc_state, *new_crtc_state;
    struct drm_plane *plane;
    struct drm_plane_state *old_plane_state, *new_plane_state;
    int i;

    /* 调用每个 CRTC 的 begin 回调 */
    for_each_oldnew_crtc_in_state(old_state, crtc,
                                  old_crtc_state, new_crtc_state, i) {
        funcs = crtc->helper_private;
        if (funcs->atomic_begin)
            funcs->atomic_begin(crtc, old_state);
    }

    /* 遍历所有 plane，调用 atomic_update 或 atomic_disable */
    for_each_oldnew_plane_in_state(old_state, plane,
                                   old_plane_state, new_plane_state, i) {
        funcs = plane->helper_private;
        if (drm_atomic_plane_disabling(old_plane_state, new_plane_state)) {
            funcs->atomic_disable(plane, old_state);
        } else {
            funcs->atomic_update(plane, old_state);
        }
    }

    /* 调用每个 CRTC 的 flush 回调，触发硬件生效 */
    for_each_oldnew_crtc_in_state(old_state, crtc,
                                  old_crtc_state, new_crtc_state, i) {
        funcs = crtc->helper_private;
        if (funcs->atomic_flush)
            funcs->atomic_flush(crtc, old_state);
    }
}
```

## 4. 驱动实现要点

### 4.1 atomic_check 回调

驱动需要在 `drm_plane_helper_funcs.atomic_check` 或 `drm_crtc_helper_funcs.atomic_check` 中检查私有约束：

```c
static int my_plane_atomic_check(struct drm_plane *plane,
                                  struct drm_atomic_state *state)
{
    struct drm_plane_state *new_state =
        drm_atomic_get_new_plane_state(state, plane);

    /* 示例：检查格式支持 */
    if (new_state->fb) {
        switch (new_state->fb->format->format) {
        case DRM_FORMAT_XRGB8888:
        case DRM_FORMAT_ARGB8888:
        case DRM_FORMAT_NV12:
            break;
        default:
            DRM_DEBUG_ATOMIC("unsupported format %p4cc\n",
                             &new_state->fb->format->format);
            return -EINVAL;
        }
    }

    /* 示例：检查缩放比例上限（最大 8x 缩放）*/
    if (new_state->crtc_w > 0 && new_state->src_w > 0) {
        u32 src_w = new_state->src_w >> 16;
        if (src_w / new_state->crtc_w > 8) {
            DRM_DEBUG_ATOMIC("downscale ratio too large\n");
            return -EINVAL;
        }
    }

    return 0;
}
```

### 4.2 atomic_commit_tail 回调

驱动可重写 `drm_mode_config_helper_funcs.atomic_commit_tail` 来控制 commit 顺序，例如在 VBlank 期间刷新寄存器：

```c
static void my_atomic_commit_tail(struct drm_atomic_state *old_state)
{
    struct drm_device *dev = old_state->dev;

    /* 先关掉旧的 encoder/CRTC */
    drm_atomic_helper_commit_modeset_disables(dev, old_state);

    /* 更新所有 plane */
    drm_atomic_helper_commit_planes(dev, old_state,
                                    DRM_PLANE_COMMIT_ACTIVE_ONLY);

    /* 开启新的 encoder/CRTC */
    drm_atomic_helper_commit_modeset_enables(dev, old_state);

    /* 等待下一个 VBlank，确保帧完整显示 */
    drm_atomic_helper_wait_for_flip_done(dev, old_state);

    /* 释放旧 framebuffer */
    drm_atomic_helper_cleanup_planes(dev, old_state);
}

static const struct drm_mode_config_helper_funcs my_mode_config_helpers = {
    .atomic_commit_tail = my_atomic_commit_tail,
};
```

## 5. 异步提交与 commit_work

当用户空间传入 `DRM_MODE_ATOMIC_NONBLOCK` 时，实际硬件操作会推迟到工作队列中执行：

```c
/* drm_atomic_helper.c */
int drm_atomic_helper_commit(struct drm_device *dev,
                              struct drm_atomic_state *state,
                              bool nonblock)
{
    /* ... check ... */

    if (nonblock)
        return drm_atomic_helper_setup_commit(state, nonblock);

    /* 同步路径：直接调用 commit_tail */
    drm_atomic_helper_commit_tail(state);
    return 0;
}
```

异步路径通过 `drm_crtc_commit` 对象和 completion 机制保证顺序：

```c
struct drm_crtc_commit {
    struct drm_crtc *crtc;

    struct kref ref;

    /* flip_done：plane 更新完成（可以释放旧 fb）*/
    struct completion flip_done;

    /* hw_done：硬件配置完成（可以开始下一次 commit）*/
    struct completion hw_done;

    /* cleanup_done：cleanup_planes 完成（可以销毁 state）*/
    struct completion cleanup_done;

    struct list_head commit_entry;
    struct drm_pending_vblank_event *event;
    bool abort_completion;
};
```

多个连续的非阻塞 commit 之间通过等待前一个 `hw_done` 来串行化硬件写入，避免乱序。

## 6. 与旧版 legacy 接口的对比

| 维度 | Legacy KMS | Atomic Modeset |
|---|---|---|
| 更新粒度 | 单对象（CRTC 或 Plane） | 事务（多对象同时更新） |
| check/commit 分离 | 否（隐式，易出错） | 是（显式两阶段） |
| 失败回滚 | 需驱动自行处理 | 框架保证 state 回滚 |
| 异步支持 | 仅 page-flip | 全部操作均可异步 |
| 驱动复杂度 | 较低 | 较高，但可复用 helper |
| 多屏同步 | 困难 | 原生支持（同一 state） |

目前新驱动均应优先实现 atomic 接口；legacy 接口由 DRM 核心通过兼容层自动适配旧版用户空间程序。

## 7. 常见问题排查

**1. atomic_check 返回 -EINVAL 但日志无明显报错**

启用 DRM 调试日志后重现：

```bash
echo 0x3f > /sys/module/drm/parameters/debug
dmesg | grep -i atomic
```

**2. 页面撕裂（tearing）仍然存在**

检查驱动的 `atomic_flush` 回调是否正确等待 VBlank，确认 hardware double buffering（寄存器 shadow）逻辑是否在 VBlank 中断中触发 latch：

```c
static irqreturn_t my_vblank_irq_handler(int irq, void *data)
{
    struct my_drm_private *priv = data;

    /* 触发硬件寄存器影子寄存器生效 */
    writel(BIT(0), priv->regs + REG_UPDATE_LATCH);

    drm_crtc_handle_vblank(&priv->crtc);
    return IRQ_HANDLED;
}
```

**3. 非阻塞 commit 之后旧 framebuffer 被提前释放**

确认驱动调用了 `drm_atomic_helper_cleanup_planes()`，且在 `flip_done` completion 之后再执行清理：

```c
/* 正确做法：等待 flip_done 再 cleanup */
drm_atomic_helper_wait_for_flip_done(dev, old_state);
drm_atomic_helper_cleanup_planes(dev, old_state);
```

---
感谢阅读！
