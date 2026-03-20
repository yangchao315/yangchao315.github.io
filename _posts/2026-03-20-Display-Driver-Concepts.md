# 显示软件工程师 JD 深度分析 — 详细知识点说明

## 目录

1. [Linux 内核架构与基础](#1-linux-内核架构与基础)
2. [DRM/KMS 显示驱动框架](#2-drmkms-显示驱动框架)
3. [fbdev 帧缓冲驱动](#3-fbdev-帧缓冲驱动)
4. [GEM / dma-buf 显存管理](#4-gem--dma-buf-显存管理)
5. [DSI / MIPI-DSI 显示接口协议](#5-dsi--mipi-dsi-显示接口协议)
6. [HDMI / DP / eDP / LVDS 显示接口](#6-hdmi--dp--edp--lvds-显示接口)
7. [VBlank 垂直同步机制](#7-vblank-垂直同步机制)
8. [中断处理与同步机制](#8-中断处理与同步机制)
9. [Android SurfaceFlinger 框架](#9-android-surfaceflinger-框架)
10. [Android HWC (Hardware Composer)](#10-android-hwc-hardware-composer)
11. [Gralloc 内存分配机制](#11-gralloc-内存分配机制)
12. [Wayland / Weston 显示框架](#12-wayland--weston-显示框架)
13. [X11 显示框架](#13-x11-显示框架)
14. [GPU / 图形渲染管线](#14-gpu--图形渲染管线)
15. [AFBC 帧缓冲压缩](#15-afbc-帧缓冲压缩)
16. [XR / VR 显示技术](#16-xr--vr-显示技术)
17. [芯片 Bring-up 与仿真平台](#17-芯片-bring-up-与仿真平台)
18. [性能测试与优化方法](#18-性能测试与优化方法)
19. [原理图与寄存器规格书](#19-原理图与寄存器规格书)
20. [Git 与代码规范](#20-git-与代码规范)
21. [多屏异显 / 分区更新 / 低功耗](#21-多屏异显--分区更新--低功耗)
22. [DRM 主线社区贡献](#22-drm-主线社区贡献)
23. [LTP-DDT 测试框架](#23-ltp-ddt-测试框架)

---

## 1. Linux 内核架构与基础

### 考察点
- **JD1**: "熟悉Linux内核架构，具备2年以上Linux DRM/KMS显示驱动开发经验"
- **JD2**: "熟悉Linux内核，掌握内核锁、中断使用、同步机制、内存申请、驱动调试手段等内核基本概念"

### 涉及知识
| 知识点 | 说明 | 深度要求 |
|---|---|---|
| 内核模块编写 | `module_init/exit`, GPL许可 | 理解 |
| 设备模型 | `platform_driver`, `i2c_driver`, `pci_driver` | 理解 |
| 内存管理 | `kmalloc/vmalloc`, `ioremap`, `dma_alloc_*` | 掌握 |
| 中断处理 | `request_irq`, `tasklet`, `workqueue`, 顶半部/底半部 | 掌握 |
| 同步原语 | `spinlock`, `mutex`, `rw_semaphore`, `rcu` | 掌握 |
| Proc/Sysfs 接口 | 驱动调试信息导出 | 理解 |

### 学习路径

#### 入门阶段
- **书籍**: 《Linux Device Drivers, 3rd Edition》(LDD3) — 第 1-10 章
- **书籍**: 《Linux Kernel Development》— Robert Love，第 4-8 章
- **在线**: Linux Kernel Documentation — [kernel.org/doc/Documentation/]

#### 进阶阶段
- **源码**: 研读内核 `drivers/gpu/drm/` 下的 DRM 驱动实现
- **书籍**: 《Understanding the Linux Kernel》— 深入内存管理子系统
- **实践**: 在 QEMU/virt 平台上编写简单的 platform 驱动

#### 资源推荐
- 内核文档: https://www.kernel.org/doc/html/latest/
- DRM 子系统文档: https://www.kernel.org/doc/html/latest/gpu/drm.html
- Linux Device Drivers 3rd Edition: https://lwn.net/Articles/driver-porting/

---

## 2. DRM/KMS 显示驱动框架

### 考察点
- **JD1**: "参与DRM/KMS框架下的DPU驱动开发与主线合入，支持主流Linux内核版本"
- **JD2**: "熟悉fbdev和DRM等主流显示框架，并有项目落地的经验；熟悉DisplayProcessUnit工作机制，掌握内核显卡驱动的设计与实现、验证方法"

### 涉及知识
DRM (Direct Rendering Manager) 是现代 Linux 显示驱动的核心框架，其核心组件：

| 组件 | 英文全称 | 功能 |
|---|---|---|
| **CRTC** | Cathode Ray Tube Controller | 扫描输出控制器，生成时序信号 |
| **Encoder** | Encoder | 将像素数据编码为显示信号 |
| **Connector** | Connector | 物理接口 (HDMI/DP/DSI等) |
| **Plane** | Plane | 画面层，支持叠加/混合 |
| **FB** | Framebuffer | 显存对象描述 |
| **Crtc** | — | `drm_crtc_*` 驱动注册 |
| **Encoder** | — | `drm_encoder_*` 驱动注册 |
| **Connector** | — | `drm_connector_*` 驱动注册 |
| **Encoder Helpers** | — | DRM Encoder 辅助函数 |
| **Bridge** | — | 级联信号转换 (如 DSI→HDMI) |
| **Panel** | — | 显示面板 (背光、EDID读取) |
| **Framebuffer** | — | `drm_framebuffer_*` |
| **Mode Setting** | — | `drm_mode_*` 模式设置 |

**DRM 驱动开发核心流程**:
```c
// 1. 分配 & 注册 DRM device
drm_dev_alloc() / drm_dev_register()

// 2. 初始化 CRTC
drm_crtc_init_with_planes()

// 3. 初始化 Encoder
drm_encoder_init()

// 4. 初始化 Connector
drm_connector_init() / drm_connector_attach_encoder()

// 5. 初始化 fbdev (可选, legacy fallback)
drm_fbdev_generic_setup()

// 6. 实现 atomic API 或 legacy set_config
drm_atomic_helper_check() / drm_mode_set_config_internal()
```

**DRM 四大对象关系**:
```
Plane → (attached to) → CRTC → (drives) → Encoder → (converts to) → Connector
                                                    ↓
                                              Framebuffer (FB)
```

### 学习路径

#### 阶段1: 框架概览
- **内核文档**: DRM overview — `Documentation/gpu/drm-uapi.rst`
- **内核文档**: DRM KMS overview — `Documentation/gpu/drm-kms.rst`
- **文章**: 《Linux DRM/Modesetting By Example》— David Herrmann
  - https://dvdhrm.github.io/xmore/docs/drm-howto/

#### 阶段2: 源码研读
- 参考驱动: `drivers/gpu/drm/vc4/` (树莓派, 最简单完整的 DRM 驱动)
- 参考驱动: `drivers/gpu/drm/udl/` (USB 显示)
- 参考驱动: `drivers/gpu/drm/etnaviv/` (嵌入式 GPU)
- 学习顺序: vc4_crtc.c → vc4_encoder.c → vc4_connector.c → vc4_kms.c

#### 阶段3: 理解 Atomic API
- `drm_atomic_helper_check()`
- `drm_atomic_commit()`
- `drm_pending_event_*` (VSync 事件)
- Properties system (brightness, alpha, zpos...)

#### 资源推荐
- DRM Wiki: https://wiki.freedesktop.org/norean/DRI/
- 内核 DRM 文档: https://www.kernel.org/doc/html/latest/gpu/drm.html
- DRM 设计文档: https://01.org/linuxgraphics/gfx-docs/drm/


---

## 3. fbdev 帧缓冲驱动

### 考察点
- **JD1**: "熟悉帧缓冲、显存管理（fbdev/drm/gem/dma-buf）、VBlank、中断处理等机制"
- **JD2**: "熟悉fbdev和DRM等主流显示框架"

### 涉及知识
fbdev (Framebuffer Device) 是 DRM 出现前的传统 Linux 显示接口：

| 概念 | 说明 |
|---|---|
| `/dev/fb0` | 帧缓冲设备节点 |
| `struct fb_info` | 帧缓冲核心结构 |
| `fb_ops` | 底层硬件操作 (set_par, pan_display, blank...) |
| `vm_area_struct` | mmap 显存映射 |
| IOCTL | `FBIOGET_VSCREENINFO`, `FBIOPUT_VSCREENINFO`, `FBIOBLANK` 等 |

**与现代显示系统的关系**:
```
用户空间 (显示APP)
    ↓ mmap / ioctl
/dev/fb0 (fbdev层, legacy)
    ↓ 抽象
DRM/KMS 驱动 (现代层)
    ↓ 寄存器操作
显示硬件 (DPU/CRTC)
```

**理解要点**: fbdev 在现代系统中通常通过 DRM 的 `drm_fbdev_generic_setup()` 自动生成，作为兼容性 fallback 存在。

### 学习路径
- 内核文档: `Documentation/fb/`
- 《Linux Device Drivers》— Framebuffer 章节
- 研读: `drivers/video/fbdev/core/fbmem.c` — fbdev 核心实现


---

## 4. GEM / dma-buf 显存管理

### 考察点
- **JD1**: "熟悉帧缓冲、显存管理（fbdev/drm/gem/dma-buf）"
- **JD2**: "熟悉DisplayProcessUnit工作机制，掌握内核显卡驱动的设计与实现"

### 涉及知识

#### GEM (Graphics Execution Manager)
DRM 框架中的显存管理子系统：

| 功能 | API |
|---|---|
| 分配显存对象 | `drm_gem_object_alloc()` |
| mmap 映射 | `drm_gem_mmap()` |
| 同步管理 | `drm_syncobj_*` |
| Prime 共享 | `drm_gem_prime_*` |

#### dma-buf
跨设备/跨驱动的缓冲区共享机制：

```c
// 导出端 (GPU驱动)
struct dma_buf *dma_buf_export_dynamic(...);
int dma_buf_attach(struct dma_buf *, struct device *);
struct sg_table *dma_buf_map_attachment(...);

// 导入端 (显示驱动)
struct dma_buf *dma_buf_get(int fd);
int dma_buf_begin_cpu_access(struct dma_buf *, enum dma_data_direction);
void *dma_buf_vmap(struct dma_buf *);
```

**使用场景**:
1. GPU 渲染 → 显示控制器读取（零拷贝）
2. Camera → ISP → 显示管线
3. 多屏共享同一帧缓冲区

### 学习路径
- 内核文档: `Documentation/driver-api/dma-buf.rst`
- 内核文档: `Documentation/gpu/drm-gem.rst`
- 源码: `drivers/dma-buf/` 和 `drivers/gpu/drm/drm_gem*.c`


---

## 5. DSI / MIPI-DSI 显示接口协议

### 考察点
- **JD1**: "调试与优化双屏、4K、MIPI-DSI、eDP、HDMI、LVDS等多种输出场景"
- **JD2**: "负责DisplayProcessUnit，及相关显示接口DSI，HDMI，DP的驱动架构设计、研发、性能优化"

### 涉及知识

#### MIPI-DSI 协议栈

| 层级 | 说明 |
|---|---|
| **物理层 (DSI PHY)** | D-PHY / C-PHY / M-PHY 电气特性 |
| **协议层** | 数据包格式 (Short/Long Packet)、Data Type |
| **命令模式 (Video Mode)** | 同步事件、像素数据流传输 |
| **视频模式 (Command Mode)** | DCS (Display Command Set) 寄存器配置 |

**DSI 数据包格式**:
```
┌──────┬────────┬──────┬──────┬─────────┐
│ Data ID │ Word Count │ ECC  │ Payload │ Checksum │
└──────┴────────┴──────┴──────┴─────────┘
```

**关键概念**:
- **Lane**: 1/2/4 lane DSI, 带宽计算
- **D-PHY**: 时钟 Lane + 数据 Lane, HS/LP 模式切换
- **DCS Commands**: `set_display_on`, `set_pixel_format`, `set_address_mode` 等
- **Video Mode**: 实时像素流, 需要 HS 模式
- **Command Mode**: 可发送命令, 支持 LP 模式省电

#### DSI 驱动架构 (Linux)
```
drm_panel (显示面板)
    ↓
mipi_dsi_device / mipi_dsi_host
    ↓
dsi_host_ops (发送数据包)
    ↓
DSI PHY driver (dsi_phy_*.c)
    ↓
硬件寄存器
```

**关键驱动组件**:
- `drivers/gpu/drm/panel/panel-*.c` — 面板驱动
- `drivers/gpu/drm/bridge/synopsys/dsi->*` — DSI bridge
- `drivers/gpu/drm/bridge/analogix/analogix*dsi*.c`

### 学习路径
- MIPI Alliance 官方规范 (MIPI DSI-2): https://www.mipi.org/specifications/display
- 内核文档: `Documentation/gpu/panel-recipe.rst`
- 研读: `drivers/gpu/drm/panel/panel-samsung-s6d16d0.c` 等典型面板驱动
- 理解 DSI 调试: 逻辑分析仪 / MIPI 协议分析仪抓包


---

## 6. HDMI / DP / eDP / LVDS 显示接口

### 考察点
- **JD1**: "HDMI、eDP、LVDS等多种输出场景"（作为调试场景提及）
- **JD2**: "负责DisplayProcessUnit，及相关显示接口DSI，HDMI，DP的驱动架构设计"

### 涉及知识

#### HDMI
| 知识点 | 说明 |
|---|---|
| TMDS 编码 | 最小化传输差分信号 |
| HDCP | 内容保护 |
| EDID | 显示器能力读取 (I2C DDC) |
| CEC | 消费电子控制 |
| HDMI Audio | 音频传输 |
| HDMI 版本 | 1.4 / 2.0 / 2.1 带宽差异 |

#### DisplayPort (DP)
| 知识点 | 说明 |
|---|---|
| AUX 通道 | 辅助通道, EDID/HPD/HDCP |
| Main Link | 高速数据传输 (1/2/4 lane) |
| MSA | Main Stream Attribute |
| DPCD | DisplayPort Configuration Data |
| 版本差异 | DP 1.4 / 2.0 / 2.1 |

#### eDP
- 嵌入式 DisplayPort, 笔记本内部屏幕接口
- 相比 DP 更简化，无外部连接器
- 涉及 Panel Self Refresh (PSR) 节能

#### LVDS
- 低压差分信号, 传统接口
- 低功耗、低 EMI
- 逐渐被 eDP/MIPI-DSI 替代

#### DRM 中的接口实现
```c
// HDMI Connector
drm_connector_init()
drm_hdmi_connector_set_funcs()

// DP Connector
drm_dp_aux_register()

// eDP (通常走 DP AUX)
drm_panel_attach()
```

### 学习路径
- HDMI spec: https://www.hdmi.org/spec/
- DisplayPort spec: https://vesa.org/standards-developments/displayport/
- 内核驱动: `drivers/gpu/drm/bridge/synopsys/`
- 内核驱动: `drivers/gpu/drm/bridge/parade-ps8622.c` 等


---

## 7. VBlank 垂直同步机制

### 考察点
- **JD1**: "熟悉帧缓冲、显存管理（fbdev/drm/gem/dma-buf）、VBlank、中断处理等机制"

### 涉及知识

#### VBlank (Vertical Blank)
CRT/LCD 显示器的扫描机制，扫描完一帧后到下一帧开始前的空档期：

| 概念 | 说明 |
|---|---|
| **VBlank 中断** | 每帧结束后触发, 用于页面翻转 |
| **drm_vblank_*()** | DRM VBlank API |
| **双缓冲 vs 三缓冲** | 缓冲区切换策略 |
| **Page Flip** | `drm_mode_page_flip()` / atomic commit |

#### DRM VBlank API
```c
// 获取 VBlank 时间戳
drm_wait_vblank_reply();

// 启用/禁用 VBlank
drm_vblank_off();
drm_vblank_on();

// 原子提交时的页面翻转
drm_atomic_helper_page_flip();
```

**VSync 流程**:
```
用户请求页面翻转
    ↓
等待 VBlank 中断
    ↓
交换缓冲区 (ping-pong)
    ↓
硬件扫描输出新帧
```

### 学习路径
- 内核文档: `Documentation/gpu/drm-kms.rst` (VBlank 部分)
- DRM 原子模式: `drivers/gpu/drm/drm_atomic.c`


---

## 8. 中断处理与同步机制

### 考察点
- **JD2**: "熟悉Linux内核，掌握内核锁、中断使用、同步机制、内存申请、驱动调试手段等内核基本概念"

### 涉及知识

#### 中断处理
| 机制 | 适用场景 | 上下文限制 |
|---|---|---|
| **Hard IRQ** | 硬件中断, 紧急处理 | 原子上下文, 不能睡眠 |
| **tasklet** | 软中断, 延迟处理 | 原子上下文 |
| **workqueue** | 延迟处理, 可睡眠 | 进程上下文 |
| ** threaded IRQ** | 中断线程化 | 可睡眠 |
| ** IRQ bottom half** | 中断下半部 | 取决于实现 |

#### 同步原语
| 原语 | 类型 | 用途 |
|---|---|---|
| `spinlock_t` | 自旋锁 | 原子上下文 |
| `struct mutex` | 互斥锁 | 进程上下文 |
| `rw_semaphore` | 读写信号量 | 读多写少 |
| `completion` | 完成量 | 等待事件 |
| `atomic_t` | 原子变量 | 计数器 |
| `rcu` | 读拷贝更新 | 免锁读 |

### 学习路径
- 《Linux Kernel Development》— 中断与异常章节
- 《Linux Device Drivers》— 第 9 章 (中断处理)
- 内核文档: `Documentation/kernel-hacking/`

---

## 9. Android SurfaceFlinger 框架

### 考察点
- **JD1** (加分项): "有Android平台下显示系统开发经验"
- **JD2**: "熟悉Android显示框架Surfacflinger和HWC"

### 涉及知识

#### SurfaceFlinger 架构
Android 的显示服务器，负责合成所有应用的 Surface：

```
┌──────────────────────────────────────────────────┐
│              应用进程 (APP)                        │
│   Canvas / OpenGL ES / Vulkan 渲染到 Surface      │
└──────────────────────────────────────────────────┘
                    ↓ 匿名共享内存 (ashmem)
┌──────────────────────────────────────────────────┐
│              SurfaceFlinger (System Server)      │
│                                                   │
│   1. HWC (Hardware Composer) 获取合成策略          │
│   2. 使用 GLES 或 HWC 合成所有 Surface            │
│   3. 双重缓冲: App 写 Surface / SF 读 Surface    │
│   4. 最终合成结果 → Framebuffer                    │
│   5. 发送 VSync 信号同步                          │
└──────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────┐
│              HAL 层 (gralloc, hwcomposer)        │
└──────────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────┐
│              Kernel DRM / fbdev                  │
└──────────────────────────────────────────────────┘
```

**SurfaceFlinger 核心组件**:

| 组件 | 功能 |
|---|---|
| **BufferQueue** | 生产者(APP)/消费者(SF) 之间的缓冲区队列 |
| **Surface** | 应用的绘图表面 |
| **Layer** | SF 中的合成层 |
| **DisplayDevice** | 显示设备抽象 |
| **HWComposer** | 硬件合成器接口 |
| **RenderEngine** | GLES 合成引擎 |

### 学习路径
- AOSP 源码: `frameworks/native/services/surfaceflinger/`
- Android HAL: `hardware/interfaces/graphics/composer/`
- 文章: Android Display System 深度系列 — 罗升阳
- 书籍: 《Android BSP 开发与维护》


---

## 10. Android HWC (Hardware Composer)

### 考察点
- **JD1** (加分项): "有HWC相关开发经验"
- **JD2**: "负责AndroidOS下HWC实现和优化"

### 涉及知识

#### HWC HAL 版本
| 版本 | Android 版本 | 特性 |
|---|---|---|
| HWC1.0 | Android 4.x | 传统接口 |
| HWC2.0 | Android 8+ | 标准化接口, 支持多种显示 |
| HWC3.0 | Android 14+ | AIDL 化 |

#### HWC2 API 核心概念
```cpp
// 核心数据类型
struct layers_t;         // 合成层列表
struct display_t;        // 显示设备
struct composer_t;       // HWC 作曲家

// 核心流程
prepare()     // 获取合成策略: CLIENT/DEVICE
set()         // 执行合成
present()     // 提交到显示硬件
getCapabilities() // 查询硬件能力
```

**合成类型**:
| 类型 | 说明 | 硬件负担 |
|---|---|---|
| **CLIENT** | SF 使用 GLES 合成 | GPU |
| **DEVICE** | HWC 直接合成 | 显示控制器 |
| **SOLID_COLOR** | 纯色层 | 无需缓冲区 |

**HWC vs SF 分工**:
```
理想情况: HWC 尽量多做合成 (省电/省GPU)
极限情况: 全 GLES 合成 (SF_GEOMETRY_CHANGED)
```

### 学习路径
- AOSP HAL: `hardware/interfaces/graphics/composer/` (V2/V3)
- 旧版 HWC1: `frameworks/native/services/surfaceflinger/HWC2.cpp`
- 典型实现: `vendor/qcom/display-hal/` 或 `vendor/xxx/display-hal/`


---

## 11. Gralloc 内存分配机制

### 考察点
- **JD1** (加分项): "有Gralloc相关开发经验"

### 涉及知识

#### Gralloc HAL
Android 图形内存分配器，负责分配用于渲染和显示的缓冲区：

```
APP 请求分配 → Gralloc HAL → DMA-Buf 导出 → 共享给 SF/GPU/显示控制器
```

**gralloc 接口**:
```cpp
int gralloc_alloc(size_t size, int usage, buffer_handle_t* handle);
int gralloc_free(buffer_handle_t handle);

// 按 GRALLOC_USAGE_* 标志分配:
// - GRALLOC_USAGE_HW_FB       // 显示控制器使用
// - GRALLOC_USAGE_HW_RENDER   // GPU 渲染使用
// - GRALLOC_USAGE_HW_COMPOSER // HWC 使用
// - GRALLOC_USAGE_PROTECTED   // 受保护内容
```

**Gralloc 与 DRM GEM**:
- gralloc 后端通常调用 `dma-buf` 或 `GEM` API
- 分配结果通过 `ANativeWindowBuffer::handle` (fd) 传递

### 学习路径
- AOSP: `hardware/libhardware/modules/gralloc/` (旧版)
- AOSP: `hardware/interfaces/graphics/allocator/` (新版 AIDL)
- 内核: `drivers/dma-buf/`, `drivers/gpu/drm/drm_gem*.c`


---

## 12. Wayland / Weston 显示框架

### 考察点
- **JD1** (加分项): "有Wayland/Weston经验"
- **JD2**: "Linux Desktop下Wayland和x11的实现和优化"

### 涉及知识

#### Wayland 架构
现代 Linux 桌面显示协议:

```
┌─────────────────────────────────────────────────┐
│                   客户端 (Client)                 │
│  EGL / Vulkan → wl_surface → Wayland Protocol  │
└─────────────────────────────────────────────────┘
                     ↓ Unix Socket
┌─────────────────────────────────────────────────┐
│              Weston (Compositor)                │
│                                                   │
│  接收 Client 的 wl_surface → 合成 → DRM/KMS      │
│  wl_shell / xdg-shell / IVI-shell              │
│  支持: GBM, EGL, libinput                       │
└─────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────┐
│           DRM / KMS (Linux Kernel)              │
└─────────────────────────────────────────────────┘
```

**核心协议**:
| 协议 | 说明 |
|---|---|
| `wl_surface` | 窗口/图层 |
| `wl_buffer` | 图形缓冲区 |
| `wl_compositor` | 合成器 |
| `wl_output` | 显示输出 |
| `xdg-shell` | 桌面窗口管理接口 |

**Weston vs SurfaceFlinger 对比**:

| 维度 | Weston | SurfaceFlinger |
|---|---|---|
| 平台 | Linux Desktop | Android |
| 协议 | Wayland | Binder IPC |
| 硬件加速 | GBM + EGL | EGL + HWC |
| 输入处理 | libinput | InputDispatcher |

### 学习路径
- Weston 源码: https://gitlab.freedesktop.org/wayland/weston
- Wayland 协议: https://wayland.freedesktop.org/
- GBM: `drivers/gpu/drm/gem-display` 下的 GBM 子系统
- 书籍: 《Wayland Development》— Packt Publishing


---

## 13. X11 显示框架

### 考察点
- **JD2**: "Linux Desktop下Wayland和x11的实现和优化"

### 涉及知识

#### X11 架构 (传统)
```
┌─────────────────────────────────────────────────┐
│        X11 Client (Qt/Gtk 应用)                 │
│        libX11 → X Protocol                      │
└─────────────────────────────────────────────────┘
                    ↓ TCP/Unix Socket
┌─────────────────────────────────────────────────┐
│            X Server (Xorg)                       │
│                                                   │
│  窗口管理, 合成, 输入处理                         │
│  驱动: modesetting (DRM) / fbdev / intel/nvidia│
└─────────────────────────────────────────────────┘
```

**X11 与 DRM 集成**:
- Xorg modesetting 驱动: `xfree86/modesetting/`
- 使用 KMS 进行模式设置
- 使用 GBM 分配缓冲区

### 学习路径
- X.Org Wiki: https://wiki.x.org/
- 《X Window System》— 架构理解


---

## 14. GPU / 图形渲染管线

### 考察点
- **JD1** (加分项): "有GPU相关开发经验"

### 涉及知识

#### GPU 渲染管线
```
应用 → 顶点着色 → 图元装配 → 光栅化 → 片段着色 → 测试与混合 → 帧缓冲
```

#### DRM 中的 GPU 协同
- `drm/gem` — 显存对象管理 (GPU 与显示共享)
- `drm/scheduler` — GPU 命令调度
- `drm/gpu_scheduler` — 多引擎调度

#### 常见 GPU 驱动
| 驱动 | GPU | 架构 |
|---|---|---|
| `i915` | Intel Gen GPU | DRM + GEM |
| `amdgpu` | AMD GPU | DRM + GEM + KFD |
| `panfrost` | ARM Mali Bifrost | DRM + panfrost |
| `vc4` / `v3d` | Broadcom VC4/V3D | DRM + V3D |

### 学习路径
- 《Real-Time Rendering》— 图形学基础
- DRM GPU 调度: `drivers/gpu/drm/scheduler/`


---

## 15. AFBC 帧缓冲压缩

### 考察点
- **JD1** (加分项): "有AFBC（ARMFrame Buffer Compression）相关开发经验"

### 涉及知识

#### AFBC 简介
ARM Frame Buffer Compression — ARM 设计的帧缓冲压缩格式，用于减少显存带宽：

| 特性 | 说明 |
|---|---|
| 无损压缩 | 压缩/解压缩零误差 |
| 带宽节省 | 可达 50% 带宽降低 |
| 分块压缩 | 16x16 或 32x8 块 |
| 色彩格式 | 支持 YUV420 / RGBA 等 |
| AFBC 1.2 | 支持 Tiled 模式 |

**显示控制器支持**:
- 需要硬件支持 (如 ARM Mali-DP / DPU)
- 驱动中需要在 plane format list 中声明 AFBC 格式
- `DRM_FORMAT_ARM_AFBC` 等格式定义

### 学习路径
- ARM 官方文档: AFBC Specification
- 内核: `include/uapi/drm/drm_fourcc.h` (AFBC 格式定义)
- 驱动参考: ARM Mali-DP 驱动中 AFBC 相关代码


---

## 16. XR / VR 显示技术

### 考察点
- **JD2**: "熟悉XR相关显示技术实现和优化，例如frontbuffer rendering，ATW，Multiview rendering等"

### 涉及知识

#### XR 显示核心技术

| 技术 | 全称 | 说明 |
|---|---|---|
| **ATW** | Asynchronous TimeWarp | 头盔移动时画面翘曲补偿 |
| **ASW** | Asynchronous SpaceWarp | 空间翘曲, 降低 GPU 负担 |
| **Front Buffer Rendering** | — | 直接渲染到前缓冲区, 避免撕裂 |
| **Multiview Rendering** | — | 单次渲染生成左右眼画面 (OVR_multiview) |
| **Single Pass Stereo** | — | 单遍立体渲染 |
| **显示畸变校正** | — | 桶形畸变/枕形畸变 |
| **低持久性** | Low Persistence | 减少运动模糊 |

#### Android XR (ARCore) 显示管线
```
┌─────────────────────────────────────┐
│       SurfaceFlinger (双显示器)      │
│   分别为左眼/右眼创建 Surface        │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│        HWC (VR Composer HAL)         │
│   TimeWarp 应用, 畸变校正            │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│        显示控制器 (OVR芯片)           │
│   双屏输出 / 时序同步                │
└─────────────────────────────────────┘
```

### 学习路径
- Oculus 官方文档: https://developer.oculus.com/documentation/
- Vulkan MultiView: `VK_EXT_multiview` 扩展
- Android VR SDK: `androidx.core.hardware.display`


---

## 17. 芯片 Bring-up 与仿真平台

### 考察点
- **JD2**: "具有FPGA、ZEBU、Veloce等仿真平台使用经验和芯片bringup经验者优先"

### 涉及知识

#### 芯片 Bring-up 流程
```
规格定义 → RTL设计 → 仿真验证 → 硅片 → Bring-up
                                              ↓
                                    Bootloader → 内核 → 显示驱动
                                    (Tianocore / U-Boot / LK)
```

#### 仿真平台

| 平台 | 厂商 | 说明 |
|---|---|---|
| **FPGA** | Xilinx / Intel | 原型验证, 最真实 |
| **ZEBU** | Synopsys | 虚拟平台, 速度慢但灵活 |
| **Veloce** | Siemens EDA | 仿真加速器 |
| **QEMU** | 开源 | 虚拟化 ARM/RISC-V |
| **J7200** | Synopsys | 下一代虚拟平台 |

#### 显示子系统 Bring-up 步骤
1. **时钟配置** — 显示时钟 PLL 配置
2. **PHY 初始化** — DSI/HDMI/DP PHY 训练
3. **DPU 初始化** — 扫描输出配置
4. **寄存器配置** — 面板参数、时序参数
5. **EDID 读取** — 显示器能力探测
6. **Framebuffer 分配** — GEM 对象创建
7. **模式设置** — DRM 模式协商
8. **双缓冲** — ping-pong 机制
9. **性能验证** — 带宽、帧率测试

### 学习路径
- 《硬件系统设计》— 芯片 Bring-up 参考
- U-Boot 显示: `drivers/video/drm/`
- Arm SoC Boot: https://u-boot.readthedocs.io/


---

## 18. 性能测试与优化方法

### 考察点
- **JD2**: "熟练掌握Linux下常见的性能测试、剖析工具及优化方法"

### 涉及知识

#### 性能测试工具

| 工具 | 用途 |
|---|---|
| **perf** | CPU profiling, `perf top`, `perf record` |
| **ftrace** | 内核函数追踪, 延迟分析 |
| **systrace** | Android 系统追踪 |
| **Simpleperf** | Android 原生 profiling |
| **drm_debug** | DRM 调试 (DRM_ATOMIC, DRM_KMS) |
| **kms++, DRM tests** | 显示驱动测试 |
| **weston-simple-egl** | Wayland 性能基准 |

#### 关键性能指标

| 指标 | 工具/方法 |
|---|---|
| 帧率 (FPS) | `/dri/0/crtc*/output_framerate` |
| 帧时间 | `drm_vblank_get()` 时间戳 |
| 显存带宽 | IP 寄存器或 perf stat |
| 显示延迟 | 传感器 + 高速摄像头 |
| 功耗 | `/sys/class/graphics/` |

#### 优化方向
- 带宽优化: AFBC, 压缩格式, tiling
- 延迟优化: 双缓冲 vs 三缓冲, 减少合成层数
- 功耗优化: Panel Self Refresh (PSR), DCRO
- CPU 优化: 中断合并, DMA 传输

### 学习路径
- perf: https://www.brendangregg.com/perf.html
- ftrace: https://www.kernel.org/doc/html/latest/trace/ftrace.html
- systrace: https://developer.android.com/topic/performance/tracing


---

## 19. 原理图与寄存器规格书

### 考察点
- **JD1**: "能看懂原理图和寄存器规格书，能与硬件团队协同调试"

### 涉及知识

#### 原理图阅读
- 显示接口信号定义 (DSI lane, HDMI differential pair)
- 电源/时钟连接
- SoC Pin MUX 配置
- I2C/SPI 控制总线

#### 寄存器规格书阅读
| 技能 | 说明 |
|---|---|
| DPU registers | CRTC, Plane, Overlay, Mixer 配置寄存器 |
| DSI registers | Lane配置, Packet格式, PHY控制 |
| PHY registers | Tx/Rx 电气特性, 时钟数据恢复 |
| Interrupt registers | 中断状态/屏蔽/清除 |

#### 寄存器操作实践
```c
// 映射外设寄存器
void __iomem *base = ioremap(res->start, resource_size(res));

// 读写寄存器
writel(value, base + OFFSET);
u32 val = readl(base + OFFSET);

// 寄存器位操作
u32 val = readl(base + OFFSET);
val |= BIT(16);
writel(val, base + OFFSET);
```

### 学习路径
- SoC 官方 TRM (Technical Reference Manual)
- 芯片参考原理图 (Renesas/Qualcomm/MTK/NXP 等)


---

## 20. Git 与代码规范

### 考察点
- **JD1**: "熟悉Git，具备良好的代码风格与文档能力"

### 涉及知识

| 技能 | 说明 |
|---|---|
| Git 基础 | add, commit, push, pull, merge, rebase |
| 分支管理 | feature branch, develop, master |
| Code Review | PR/MR, 评审流程 |
| 提交规范 | `feat:`, `fix:`, `docs:` 等 commit message |
| 子模块管理 | git submodule |
| Patch 制作 | `git format-patch`, `git send-email` |

### 学习路径
- Pro Git 书籍: https://git-scm.com/book/zh/v2
- Linux 内核提交规范: `Documentation/process/submitting-patches.rst`


---

## 21. 多屏异显 / 分区更新 / 低功耗

### 考察点
- **JD1** (加分项): "有复杂场景调试经验（如多屏异显、旋转缩放、分区更新、低功耗等）"

### 涉及知识

#### 多屏异显 (Multi-Display)
```
┌─────────────────────────────┐
│   显示控制器 (DPU)           │
│                             │
│  ┌─────────┐ ┌─────────┐  │
│  │ Plane 0 │ │ Plane 1 │  │  ← 多个图层
│  │ Display A│ │Display B│ │
│  └────┬────┘ └────┬────┘  │
│       ↓           ↓        │
│    CRTC 0      CRTC 1      │  ← 多个 CRTC
│       ↓           ↓        │
│   Encoder     Encoder      │
│      ↓           ↓         │
│  Connector   Connector     │
└─────────────────────────────┘
```

**DRM 多屏实现**:
```c
// 为每个显示器创建独立的 CRTC/Encoder/Connector
// 通过 drmModeSetCrtc() 配置各显示器
// plane 可以 attach 到不同的 CRTC
```

#### 分区更新 (Partial Update)
- 只更新屏幕变化的区域, 节省带宽
- 面板支持: e-ink, AMOLED 常用
- DRM atomic commit 中的 damage tracking

#### 低功耗优化
| 技术 | 说明 |
|---|---|
| **PSR** (Panel Self Refresh) | 静止画面时面板自刷新, 关闭 DPU |
| **DRCS** (Dynamic Refresh Rate) | 动态刷新率 |
| **DCRO** (DRM Clock Resource Optimization) | 动态时钟门控 |
| **LVDC** | 低压差分时钟门控 |


---

## 22. DRM 主线社区贡献

### 考察点
- **JD1** (加分项): "有主线社区patch提交经验"

### 涉及知识

#### Linux DRM 主线开发流程
1. **邮件列表订阅**: dri-devel@lists.freedesktop.org
2. **代码规范**: Linux 内核编码风格
3. **测试**: `igt` (Intel Graphics Tools) 测试套件
4. **提交**: `git format-patch` + `git send-email`
5. **Review**: Patchwork 跟踪 (https://patchwork.freedesktop.org/)
6. **Bot 检测**: `checkpatch.pl`, `syzkaller` fuzz

#### 知名 DRM 开发者
- Dave Airlie (Red Hat) — DRM maintainer
- Daniel Vetter (Intel) — DRM, Tungsten
- Rob Clark (Qualcomm) — Freedreno (Adreno GPU)

### 学习路径
- Linux 内核贡献指南: https://www.kernel.org/doc/html/latest/process/
- DRM 子系统指南: `Documentation/gpu/`


---

## 23. LTP-DDT 测试框架

### 考察点
- **JD2**: "熟悉和掌握LTP-DDT的测试case的设计、修改、实现和调试"

### 涉及知识

#### LTP (Linux Test Project)
- 系统级功能测试
- 覆盖内核系统调用、内存、网络、文件系统

#### DDT (Driver Development Test)
- 驱动级测试
- 针对特定驱动 (如显示驱动) 的测试

#### IGT (Intel Graphics Tools)
显示驱动测试的事实标准:

```
igt@gem_@          — GEM/显存测试
igt@kms_@          — KMS/显示功能测试
igt@i915_@         — Intel GPU 特定测试
```

**常用 IGT 命令**:
```bash
igt-runner list                    # 列出所有测试
igt-runner run kms_pipe_crc_basic  # 运行多屏 CRC 测试
igt@kms_plane@plane-panning       # 平面平移测试
igt@kms_flip@basic-plain-flip      # 页面翻转测试
```

### 学习路径
- IGT 源码: https://gitlab.freedesktop.org/drm/igt-gpu-tools
- LTP: https://github.com/linux-test-project/ltp


---

## 附录: 两家 JD 对比

| 维度 | 无锡诚恒微电子 | 此芯科技 |
|---|---|---|
| **技术栈** | 以 DRM/KMS 内核驱动为主 | 覆盖 Android + Linux 双栈 |
| **深度** | 强调 DRM 主线合入、AFBC 加分 | 强调 SoC 集成、性能优化 |
| **加分项** | DRM 主线贡献、XR/VR | 芯片 Bring-up、XR/VR |
| **系统视野** | 中等 | 更广 (SoC 层面) |
| **城市** | 无锡 | 苏州 |
| **薪资** | 15-30K·15薪 | 18-25K·15薪 |
