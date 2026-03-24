---
layout: post
title: "ARMv8 系统启动流程深度解析：BL0~BL33、TEE、SMMU 与异常等级"
date: 2026-03-23
tags: [Amlogic]
comments: true
author: yangchao
---

ARMv8 平台的启动流程涉及多个引导阶段（BL0~BL33）、安全世界（TrustZone/TEE）、DDR 初始化、MMU/SMMU 配置、中断控制器初始化以及最终跳转到 Linux Kernel。本文从硬件上电复位出发，逐阶段剖析每个 BL 阶段的功能、异常等级切换、TEE 启动机制，并给出 Mermaid 流程图辅助理解。

<!-- more -->

- [一、ARMv8 异常等级与安全状态](#一armv8-异常等级与安全状态)
  - [1.1 Exception Level 划分](#11-exception-level-划分)
  - [1.2 安全状态与 TrustZone](#12-安全状态与-trustzone)
- [二、整体启动流程总览](#二整体启动流程总览)
- [三、BL0：片上 ROM 启动](#三bl0片上-rom-启动)
  - [3.1 硬件复位与 CPU 状态](#31-硬件复位与-cpu-状态)
  - [3.2 BL0 完成的工作](#32-bl0-完成的工作)
- [四、BL1：AP Trusted ROM / 第一阶段 Bootloader](#四bl1ap-trusted-rom--第一阶段-bootloader)
  - [4.1 运行环境](#41-运行环境)
  - [4.2 主要功能](#42-主要功能)
- [五、BL2：Trusted Boot Firmware](#五bl2trusted-boot-firmware)
  - [5.1 DDR 初始化](#51-ddr-初始化)
  - [5.2 安全外设初始化](#52-安全外设初始化)
  - [5.3 加载后续镜像](#53-加载后续镜像)
- [六、BL31：EL3 Runtime Firmware（ATF）](#六bl31el3-runtime-firmwareatf)
  - [6.1 EL3 永驻服务](#61-el3-永驻服务)
  - [6.2 GIC 中断初始化](#62-gic-中断初始化)
  - [6.3 PSCI 电源管理](#63-psci-电源管理)
  - [6.4 SMC 调用处理](#64-smc-调用处理)
- [七、BL32：Secure EL1 TEE OS](#七bl32secure-el1-tee-os)
  - [7.1 TEE 启动流程](#71-tee-启动流程)
  - [7.2 SMMU 安全配置](#72-smmu-安全配置)
  - [7.3 安全内存隔离](#73-安全内存隔离)
- [八、BL33：Non-Secure Bootloader（U-Boot / UEFI）](#八bl33non-secure-bootloaderu-boot--uefi)
  - [8.1 MMU 初始化](#81-mmu-初始化)
  - [8.2 外设驱动初始化](#82-外设驱动初始化)
  - [8.3 Kernel Image 加载与跳转](#83-kernel-image-加载与跳转)
- [九、Linux Kernel 早期启动](#九linux-kernel-早期启动)
  - [9.1 head.S 汇编阶段](#91-heads-汇编阶段)
  - [9.2 MMU 与页表建立](#92-mmu-与页表建立)
  - [9.3 中断控制器与 GIC 初始化](#93-中断控制器与-gic-初始化)
- [十、虚拟化：Hypervisor 与 Guest OS](#十虚拟化hypervisor-与-guest-os)
  - [10.1 EL2 Hypervisor 启动](#101-el2-hypervisor-启动)
  - [10.2 SMMU 虚拟化（Stage-2 Translation）](#102-smmu-虚拟化stage-2-translation)
- [十一、完整启动时序图](#十一完整启动时序图)

---

## 一、ARMv8 异常等级与安全状态

### 1.1 Exception Level 划分

ARMv8-A 定义了 4 个异常等级（Exception Level, EL），权限从低到高：

| 等级 | 典型软件 | 特权 |
|------|----------|------|
| EL0 | 用户态应用 / TEE TA | 最低 |
| EL1 | Linux Kernel / TEE OS (S-EL1) | 内核级 |
| EL2 | Hypervisor (KVM/Xen) | 虚拟化 |
| EL3 | Secure Monitor (ATF BL31) | 最高，控制安全/非安全切换 |

```mermaid
graph LR
    EL0["EL0\nApp / TA"] -->|异常上升| EL1["EL1\nKernel / TEE OS"]
    EL1 -->|异常上升| EL2["EL2\nHypervisor"]
    EL2 -->|SMC/异常| EL3["EL3\nSecure Monitor\n(ATF BL31)"]
    EL3 -->|切换SCR_EL3.NS| SW["Secure World\nS-EL1 TEE"]
    EL3 -->|切换SCR_EL3.NS=1| NW["Non-Secure World\nEL1/EL2/EL0"]
    style EL3 fill:#c0392b,color:#fff
    style SW fill:#8e44ad,color:#fff
    style NW fill:#2980b9,color:#fff
```

### 1.2 安全状态与 TrustZone

ARM TrustZone 将处理器划分为**安全世界（Secure World）**和**普通世界（Normal World）**。`SCR_EL3.NS` 位控制当前世界：
- `NS=0`：安全世界，访问所有内存/外设
- `NS=1`：普通世界，被 TrustZone 保护区域不可访问

EL3（Secure Monitor）是唯一能够在两个世界之间切换的等级，通过 `SMC`（Secure Monitor Call）指令触发。

---

## 二、整体启动流程总览

```mermaid
flowchart TD
    RESET([上电复位\nCPU Reset Vector]) --> BL0
    BL0["BL0\n片上 ROM\nEL3 Secure\n- CPU 最小初始化\n- 验证 BL1/BL2\n- 加载到 SRAM"] --> BL1
    BL1["BL1\nAP Trusted ROM\nEL3 Secure\n- 时钟/PLL 初始化\n- Flash/eMMC 驱动\n- 加载 BL2"] --> BL2
    BL2["BL2\nTrusted Boot Firmware\nEL1 Secure\n- DDR 初始化训练\n- TZASC/TZC400 配置\n- 加载 BL31/BL32/BL33"] --> BL31
    BL2 --> BL32
    BL2 --> BL33_img["BL33 镜像\n加载到 DRAM"]
    BL31["BL31\nEL3 Runtime\n- GIC 初始化\n- PSCI\n- SMC 处理\n- 永驻 EL3"] --> BL32
    BL32["BL32\nTEE OS (OP-TEE)\nS-EL1\n- Secure Heap\n- SMMU 安全配置\n- TA 管理"] --> BL33
    BL33["BL33\nU-Boot / UEFI\nEL2/EL1 Non-Secure\n- MMU 开启\n- 外设驱动\n- 加载 Kernel\n- 设备树传递"] --> KERNEL
    KERNEL["Linux Kernel\nEL1 Non-Secure\n- head.S 汇编\n- 页表重建\n- GIC 驱动\n- SMMU 驱动\n- 子系统初始化"] --> USERSPACE
    USERSPACE([用户空间\nEL0])

    style RESET fill:#27ae60,color:#fff
    style BL0 fill:#e74c3c,color:#fff
    style BL1 fill:#e67e22,color:#fff
    style BL2 fill:#f39c12,color:#fff
    style BL31 fill:#c0392b,color:#fff
    style BL32 fill:#8e44ad,color:#fff
    style BL33 fill:#2980b9,color:#fff
    style KERNEL fill:#16a085,color:#fff
    style USERSPACE fill:#27ae60,color:#fff
```

---

## 三、BL0：片上 ROM 启动

### 3.1 硬件复位与 CPU 状态

CPU 上电或复位后，PC 跳转到固定的复位向量（Reset Vector），通常为片上 ROM 起始地址（如 `0x00000000` 或 `0xFFFF0000`）。此时处于：
- **EL3**，**安全世界**
- AArch64 或 AArch32（由 `RES1` 引脚/熔丝决定）
- 寄存器处于 UNKNOWN 状态，需要软件初始化

### 3.2 BL0 完成的工作

```mermaid
flowchart LR
    A[CPU 复位 EL3] --> B[栈指针 SP_EL3 初始化]
    B --> C[CPU 内部寄存器清零\nSCTLR_EL3 关闭 MMU/Cache]
    C --> D[安全配置寄存器\nSCR_EL3 初始化]
    D --> E[片上 SRAM 可用性检测]
    E --> F[存储接口初始化\neMMC/SPI NOR BootROM驱动]
    F --> G[镜像签名验证\nHASH/RSA 公钥]
    G --> H{验证通过?}
    H -- Yes --> I[加载 BL1/BL2 到 SRAM]
    H -- No --> J[进入恢复模式\nUSB/UART下载]
    I --> K[跳转 BL1/BL2\nATF Entry Point]
```

关键寄存器初始化：
```asm
/* 关闭 MMU，禁用 I/D Cache，进入已知状态 */
msr     SCTLR_EL3, xzr
isb

/* 配置安全状态：EL3 为 AArch64，允许访问安全内存 */
mov     x0, #(SCR_RES1_BITS | SCR_RW_BIT)
msr     SCR_EL3, x0

/* 设置异常向量表 */
adr     x0, bl0_vector_table
msr     VBAR_EL3, x0
isb
```

---

## 四、BL1：AP Trusted ROM / 第一阶段 Bootloader

### 4.1 运行环境

- **异常等级**：EL3，安全世界
- **运行介质**：片上 SRAM（DDR 尚未初始化）
- **入口**：ATF 标准入口 `bl1_entrypoint`

### 4.2 主要功能

| 功能 | 说明 |
|------|------|
| CPU 架构初始化 | 开启指令缓存（I-Cache），设置 CPACR/CPTR |
| PLL/时钟初始化 | 配置主 PLL，提升 CPU 频率至工作频率 |
| UART 初始化 | 早期调试串口输出 |
| 存储控制器初始化 | eMMC/UFS/SPI NOR 控制器基本配置 |
| BL2 镜像加载 | 从存储介质加载 BL2 到 SRAM/IRAM |
| 镜像认证 | 使用 TBB（Trusted Board Boot）验证 BL2 签名 |
| 跳转 BL2 | 通过 `bl1_run_next_image()` 跳转到 BL2 EL1 |

BL1 → BL2 的异常等级切换（EL3 → EL1 Secure）：
```c
/* ATF bl1_main.c 简化逻辑 */
void bl1_main(void)
{
    /* 加载 BL2 镜像描述 */
    image_info_t bl2_image = load_image(BL2_ID);

    /* 构造 BL2 入口参数：在 EL1 Secure 运行 */
    entry_point_info_t ep = {
        .pc    = bl2_image.image_base,
        .spsr  = SPSR_64(MODE_EL1, MODE_SP_ELX, DISABLE_ALL_EXCEPTIONS),
        /* NS bit = 0 → Secure EL1 */
    };

    /* ERET 到 EL1S */
    bl1_run_next_image(&ep);
}
```

---

## 五、BL2：Trusted Boot Firmware

BL2 在 **Secure EL1** 运行，是启动链中功能最复杂的阶段，核心职责是初始化 DDR 并加载后续所有镜像。

### 5.1 DDR 初始化

DDR 初始化是 BL2 最关键的硬件工作，分为以下步骤：

```mermaid
flowchart TD
    A[BL2 开始 DDR 初始化] --> B[DDR PHY 复位\n配置 PLL/时钟]
    B --> C[ZQ 校准\nODT/驱动强度]
    C --> D[发送 MRS 命令\n配置 MR0-MR6]
    D --> E[Write Leveling\n时序对齐]
    E --> F[Read/Write DQ Training\nDLL 锁相]
    F --> G[内存颗粒 BIST 测试]
    G --> H{测试通过?}
    H -- Yes --> I[DDR 控制器 DRAM Map 配置\n地址映射/交织]
    H -- No --> J[尝试降速/换配置\n失败则 Panic]
    I --> K[ECC 初始化\n全内存清零]
    K --> L[DDR 可用\n迁移运行环境到 DRAM]
```

### 5.2 安全外设初始化

BL2 配置 TrustZone 内存/外设保护控制器，防止普通世界访问安全资源：

```c
/* TZC-400 配置示例（保护安全内存区域）*/
tzc400_init(TZC_BASE);
/* 区域0：默认拒绝所有访问 */
tzc400_configure_region0(TZC_REGION_S_NONE, 0);
/* 区域1：安全 TEE 内存，仅 Secure 可访问 */
tzc400_configure_region(1,
    TEE_DRAM_BASE, TEE_DRAM_BASE + TEE_DRAM_SIZE - 1,
    TZC_REGION_S_RDWR,   /* Secure RW */
    0);                   /* Non-Secure 拒绝 */
/* 区域2：共享内存，Normal World 只读 */
tzc400_configure_region(2,
    SHARED_MEM_BASE, SHARED_MEM_BASE + SHARED_MEM_SIZE - 1,
    TZC_REGION_S_RDWR,
    TZC_REGION_ACCESS_RDONLY(0));
tzc400_set_action(TZC_ACTION_ERR); /* 违规触发总线错误 */
```

### 5.3 加载后续镜像

BL2 依次加载 BL31（EL3 Runtime）、BL32（TEE OS）、BL33（U-Boot）并验证签名，然后通过 `bl2_run_next_image` 跳转到 BL31：

```mermaid
sequenceDiagram
    participant BL2
    participant Storage as eMMC/Flash
    participant DRAM

    BL2->>Storage: 读取 FIP 镜像包
    Storage-->>BL2: FIP (BL31+BL32+BL33+证书)
    BL2->>BL2: 验证 FIP 签名链 (TBB)
    BL2->>DRAM: 加载 BL31 → BL31_BASE
    BL2->>DRAM: 加载 BL32 → BL32_BASE (Secure DRAM)
    BL2->>DRAM: 加载 BL33 → BL33_BASE (NS DRAM)
    BL2->>BL2: 构造 BL31 入口参数\n(bl31_params: BL32/BL33 ep_info)
    BL2->>BL2: ERET → EL3 跳转 BL31
```

---

## 六、BL31：EL3 Runtime Firmware（ATF）

BL31 是**永驻 EL3** 的运行时固件，负责提供 EL3 级安全服务。系统运行期间，BL31 始终驻留在安全内存中，通过 `SMC` 接口对外提供服务。

### 6.1 EL3 永驻服务

```mermaid
graph TB
    subgraph EL3["EL3 BL31 (永驻)"]
        PSCI["PSCI\n电源管理服务\nCPU_ON/OFF\nSYSTEM_SUSPEND"]
        SMC_HANDLER["SMC 分发器\nSMC Handler"]
        SDEI["SDEI\n软件授权异常接口"]
        TZ_MGMT["TrustZone 世界切换\n安全/普通世界管理"]
        GIC_EL3["GIC EL3 部分配置\nFIQ 路由到 EL3"]
    end
    NORMAL["Normal World\nEL1/EL0"] -->|SMC 指令| SMC_HANDLER
    SECURE["Secure World\nS-EL1 TEE"] -->|SMC 指令| SMC_HANDLER
    SMC_HANDLER --> PSCI
    SMC_HANDLER --> SDEI
    SMC_HANDLER --> TZ_MGMT
    TZ_MGMT -->|ERET NS=1| NORMAL
    TZ_MGMT -->|ERET NS=0| SECURE
```

### 6.2 GIC 中断初始化

ARMv8 系统通常使用 GICv3/v4 中断控制器。BL31 完成 Distributor 和 Re-Distributor 的基础初始化：

```c
/* GICv3 初始化 (gicv3_driver_init) */
void gicv3_distif_init(void)
{
    /* 禁用 Distributor */
    gicd_write_ctlr(gicd_base, 0);

    /* 配置所有 SPI 为不触发（默认禁用）*/
    for (i = MIN_SPI_ID; i <= MAX_SPI_ID; i += 32)
        gicd_write_icenabler(gicd_base, i, 0xFFFFFFFF);

    /* 设置 SPI 默认优先级 */
    for (i = MIN_SPI_ID; i <= MAX_SPI_ID; i += 4)
        gicd_write_ipriorityr(gicd_base, i, GIC_HIGHEST_NS_PRIORITY);

    /* 启用 Affinity Routing (ARE_S/ARE_NS) */
    gicd_write_ctlr(gicd_base,
        CTLR_ENABLE_G0_BIT | CTLR_ENABLE_G1S_BIT |
        CTLR_ENABLE_G1NS_BIT | CTLR_ARE_S_BIT | CTLR_ARE_NS_BIT);
}
```

GIC 中断分组策略：

| 中断组 | 目标 | 触发信号 | 用途 |
|--------|------|----------|------|
| Group 0 | EL3 | FIQ | 安全中断，EL3 直接处理 |
| Group 1 Secure | S-EL1 TEE | FIQ | TEE 专用中断（TrustZone 外设）|
| Group 1 Non-Secure | EL1/EL2 | IRQ | Linux 普通中断 |

### 6.3 PSCI 电源管理

PSCI（Power State Coordination Interface）是 Linux 通过 SMC 调用 BL31 实现 CPU 热插拔、系统休眠的标准接口：

```mermaid
sequenceDiagram
    participant Linux as Linux Kernel EL1
    participant BL31 as BL31 EL3
    participant HW as 电源管理硬件

    Linux->>BL31: SMC(PSCI_CPU_OFF)
    BL31->>BL31: 保存 CPU 上下文
    BL31->>HW: 关闭 CPU 电源域
    Note over BL31: CPU 进入 OFF 状态

    Linux->>BL31: SMC(PSCI_CPU_ON, mpidr, entry)
    BL31->>HW: 上电 CPU 电源域
    BL31->>Linux: ERET 到指定 entry 地址
```

### 6.4 SMC 调用处理

```c
/* BL31 SMC 处理入口（简化）*/
uintptr_t smc_handler(uint32_t smc_fid, u_register_t x1,
                      u_register_t x2, u_register_t x3,
                      u_register_t x4, void *cookie,
                      void *handle, u_register_t flags)
{
    uint32_t owning_entity = GET_SMC_OEN(smc_fid);

    switch (owning_entity) {
    case OEN_ARM_START ... OEN_ARM_END:
        /* ARM 标准服务：PSCI, SDEI, etc. */
        return arm_svc_smc_handler(smc_fid, ...);
    case OEN_SIP_START ... OEN_SIP_END:
        /* 厂商 SiP 服务（如 Amlogic 私有服务）*/
        return sip_svc_smc_handler(smc_fid, ...);
    case OEN_TOS_START ... OEN_TOS_END:
        /* 路由到 TEE OS (BL32) */
        return optee_smc_handler(smc_fid, ...);
    }
}
```

---

## 七、BL32：Secure EL1 TEE OS

BL32 通常是 **OP-TEE**（Open Portable Trusted Execution Environment），运行于 **Secure EL1**，提供安全存储、密钥管理、DRM、指纹等可信服务。

### 7.1 TEE 启动流程

```mermaid
flowchart TD
    BL31_LAUNCH["BL31 启动 BL32\nERET → S-EL1"] --> OPTEE_RESET
    OPTEE_RESET["OP-TEE head.S\n- 设置 SP_EL1\n- 初始化异常向量\n- 关闭 MMU"] --> PLAT_INIT
    PLAT_INIT["平台初始化\n- Secure UART\n- 平台特定外设"] --> MEM_INIT
    MEM_INIT["安全内存初始化\n- Secure Heap 建立\n- 内存池划分"] --> MMU_INIT
    MMU_INIT["MMU 初始化\n- 安全页表建立\n- TTBR0_EL1 设置\n- 开启 MMU"] --> THREAD_INIT
    THREAD_INIT["线程系统初始化\n- Boot thread\n- IRQ stack"] --> TA_FRAMEWORK
    TA_FRAMEWORK["TA 框架初始化\n- 安全存储 rpmb\n- 密码库 libtomcrypt"] --> NOTIFY_BL31
    NOTIFY_BL31["通知 BL31\n- SMC OPTEE_CALL_RETURN_FROM_RPC\n- TEE OS 初始化完毕"] --> WAIT_NS
    WAIT_NS["等待 Normal World 调用\n通过 SMC 进入 TEE"]
```

### 7.2 SMMU 安全配置

SMMU（System Memory Management Unit）是外设 DMA 的 MMU，BL2/BL32 需配置其安全属性，防止外设 DMA 访问安全内存：

```mermaid
graph TB
    subgraph SMMUv3["SMMUv3 架构"]
        direction TB
        NS_STREAM["Non-Secure 设备流\n(NS StreamID)"]
        S_STREAM["Secure 设备流\n(S StreamID)"]
        STB["Stream Table\n(Linear/2-Level)"]
        STE_NS["STE: Stage-1 (NS)\nIOVA → IPA\n+ Stage-2\nIPA → PA"]
        STE_S["STE: S-Stage-1\nIOVA → Secure PA\n只能访问安全内存"]
        CMDQ["Command Queue\nINVALIDATE/SYNC"]
        EVENTQ["Event Queue\n违规上报"]
    end

    NS_STREAM --> STB --> STE_NS
    S_STREAM --> STB --> STE_S
    STE_NS -->|违规| EVENTQ
    CMDQ -->|管理| STB

    BL32["BL32 配置 Secure STE\n绑定安全外设 StreamID"] --> S_STREAM
    BL33["BL33/Kernel 配置 NS STE\n建立 IOVA 页表"] --> NS_STREAM
```

### 7.3 安全内存隔离

```
物理内存布局示例（基于 TZC-400 保护）：

0x00000000 ┌─────────────────────────────┐
           │  BL31 (EL3 Runtime Code)    │ ← Secure Only
0x04000000 ├─────────────────────────────┤
           │  OP-TEE OS (BL32)           │ ← Secure Only
           │  + Secure Heap              │
0x06000000 ├─────────────────────────────┤
           │  TEE Shared Memory          │ ← NS Read / S RW
0x06100000 ├─────────────────────────────┤
           │  Normal World DRAM          │ ← NS RW (主内存)
           │  (Linux Kernel, 用户空间)   │
0x80000000 └─────────────────────────────┘
```

---

## 八、BL33：Non-Secure Bootloader（U-Boot / UEFI）

BL31 启动完 BL32（TEE）后，通过 ERET 切换到普通世界，跳转到 BL33（通常是 U-Boot）。BL33 运行于 **EL2 或 EL1，普通世界**。

### 8.1 MMU 初始化

U-Boot 在 `board_init_f()` 阶段开启 MMU（用于 Cache 加速和内存保护）：

```c
/* U-Boot arch/arm/cpu/armv8/cache.c 简化 */
void enable_caches(void)
{
    /* 建立平坦映射页表（identity map）*/
    setup_pgtables();   /* TTBR0_EL1/EL2 指向页表基地址 */

    /* 设置 TCR_EL2：4KB 粒度，48-bit VA */
    write_tcr_el2(TCR_EL2_PS_BITS_48 | TCR_EL2_TG0_4K |
                  TCR_EL2_IRGN0_WBWA | TCR_EL2_ORGN0_WBWA |
                  TCR_EL2_T0SZ(16));

    /* MAIR：内存属性配置（Device/Normal/NC）*/
    write_mair_el2(MAIR_ATTR_DEVICE_nGnRE    << (0 * 8) |
                   MAIR_ATTR_NORMAL_WB_NTR_RA_WA << (1 * 8));

    isb();
    /* 开启 MMU + D-Cache + I-Cache */
    set_sctlr(get_sctlr() | CR_M | CR_C | CR_I);
    isb();
}
```

### 8.2 外设驱动初始化

U-Boot 的外设初始化顺序（`initcall_run_list`）：

```mermaid
flowchart LR
    A[board_init_f] --> B[arch_cpu_init\n基础 CPU 配置]
    B --> C[timer_init\n系统定时器]
    C --> D[env_init\n环境变量]
    D --> E[serial_init\nUART 串口]
    E --> F[board_init_r\n完整初始化]
    F --> G[mmc_init\neMMC/SD 驱动]
    F --> H[usb_init\nUSB 驱动]
    F --> I[net_init\n以太网]
    F --> J[pci_init\nPCIe]
    G & H & I & J --> K[启动命令解析\nbootcmd 执行]
    K --> L[加载 Kernel + DTB\nfatload/ext4load]
    L --> M[bootm/booti 命令\n跳转 Kernel]
```

### 8.3 Kernel Image 加载与跳转

ARM64 Linux Kernel（Image/Image.gz）加载到 DRAM 后，U-Boot 通过 `booti` 命令跳转：

```bash
# U-Boot 典型启动命令
setenv bootargs "root=/dev/mmcblk0p2 rootwait console=ttyAML0,115200"
fatload mmc 0:1 ${kernel_addr_r} Image        # 加载 Kernel
fatload mmc 0:1 ${fdt_addr_r}    device.dtb   # 加载设备树
booti ${kernel_addr_r} - ${fdt_addr_r}        # 跳转
```

`booti` 实际调用流程（最终跳转到 Kernel entry）：

```mermaid
sequenceDiagram
    participant UBoot as U-Boot
    participant ATF as BL31 EL3
    participant Kernel as Linux Kernel

    UBoot->>UBoot: 校验 ARM64 Image magic (0x644d5241)
    UBoot->>UBoot: 关闭 MMU/Cache (cleanup_before_linux)
    UBoot->>UBoot: 设置 x0=fdt地址, x1/x2/x3=0
    Note over UBoot: SPSR 配置目标为 EL2/EL1
    UBoot->>ATF: SMC(PSCI_SYSTEM_RESET) [可选]\n或直接 br x4 跳转 Kernel entry
    ATF->>ATF: ERET → EL2/EL1 (Non-Secure)
    ATF->>Kernel: 跳转到 kernel_entry(fdt, 0, 0, 0)
    Note over Kernel: x0 = 设备树 DTB 物理地址
```

---

## 九、Linux Kernel 早期启动

### 9.1 head.S 汇编阶段

Linux Kernel 入口 `arch/arm64/kernel/head.S` 完成 MMU 开启前的准备工作：

```mermaid
flowchart TD
    ENTRY["_head (kernel entry)\nx0=DTB 地址"] --> PRESERVE_DTB["保存 x0 (DTB 地址) 到 x21"]
    PRESERVE_DTB --> EL_CHECK["el2_setup:\n判断当前异常等级\n(EL2 还是 EL1)"]
    EL_CHECK -->|EL2| HYP_STUB["安装 HYP stub\nHVC trap 处理\n降级到 EL1"]
    EL_CHECK -->|EL1| NEXT
    HYP_STUB --> NEXT
    NEXT["set_cpu_boot_mode_flag\n保存启动模式"] --> PRESERVE_ARGS["__preserve_boot_args\n保存 x0-x3"]
    PRESERVE_ARGS --> KASLR["__relocate_kernel\nKASLR 地址随机化 (可选)"]
    KASLR --> PGTABLE["__create_page_tables\n建立初始页表\nidmap_pg_dir + init_pg_dir"]
    PGTABLE --> MMU_ON["__primary_switch\n开启 MMU\nTTBR0 = idmap, TTBR1 = init"]
    MMU_ON --> START_KERNEL["__primary_switched\n→ start_kernel()"]
```

### 9.2 MMU 与页表建立

ARM64 Linux 使用 4 级页表（4KB 页，48-bit VA），内核地址空间从 `0xFFFF000000000000` 开始：

```
ARM64 虚拟地址空间划分 (48-bit VA, 4KB)：

0x0000000000000000 - 0x0000FFFFFFFFFFFF  用户空间 (TTBR0_EL1)
0xFFFF000000000000 - 0xFFFFFFFFFFFFFFFF  内核空间 (TTBR1_EL1)
                                          ↑ 包含：线性映射、vmalloc、vmemmap
```

### 9.3 中断控制器与 GIC 初始化

Linux 在 `start_kernel()` → `init_IRQ()` → DT 解析 GIC 节点后初始化 GICv3 驱动：

```c
/* drivers/irqchip/irq-gic-v3.c 关键初始化 */
static int gic_init_bases(...)
{
    /* 初始化 Distributor */
    gic_dist_init(gic_data);

    /* 每个 CPU 初始化 Redistributor + CPU Interface */
    gic_cpu_init(gic_data);        /* 设置 SRE, PMR, BPR */
    gic_smp_init();                /* 注册 IPI 中断 */

    /* 设置中断亲和性、优先级、触发方式 */
    set_handle_irq(gic_handle_irq);
}
```

GIC 中断处理流程：

```mermaid
flowchart LR
    IRQ_SRC["外设触发 IRQ\n(SPI/PPI/SGI)"] --> GIC_DIST["GIC Distributor\n仲裁 + 路由"]
    GIC_DIST --> GIC_REDIST["Redistributor\n（目标 CPU）"]
    GIC_REDIST --> CPU_IF["CPU Interface\nICC_IAR1_EL1 读取"]
    CPU_IF --> KERNEL_IRQ["Linux IRQ 向量\nel1_irq / el0_irq"]
    KERNEL_IRQ --> HANDLE["handle_arch_irq()\n→ 设备驱动 handler"]
    HANDLE --> EOI["ICC_EOIR1_EL1\n中断结束 EOI"]
```

---

## 十、虚拟化：Hypervisor 与 Guest OS

### 10.1 EL2 Hypervisor 启动

当系统支持虚拟化时（如运行 KVM），Linux 可作为 Hypervisor 运行于 EL2，或通过 KVM 模块在 EL1 宿主机上管理 EL1 Guest：

```mermaid
flowchart TD
    BL33 -->|"SPSR EL2"| KVM_HOST["Linux Host\nEL2 (VHE) 或 EL1"]
    KVM_HOST --> KVM_MOD["KVM 模块加载\n__kvm_hyp_init\n安装 EL2 向量表"]
    KVM_MOD --> VCPU["创建 VCPU\nkvm_vcpu_create()"]
    VCPU --> GUEST_RUN["kvm_arch_vcpu_ioctl_run()\nhyp_entry"]
    GUEST_RUN --> GUEST["Guest OS EL1\n(VM 内核)"]
    GUEST -->|"HVC/物理中断"| KVM_EXIT["VM Exit\nhandle_exit()"]
    KVM_EXIT --> EMULATE["MMIO 模拟\n中断注入\n设备透传"]
    EMULATE --> GUEST_RUN

    style KVM_HOST fill:#2980b9,color:#fff
    style GUEST fill:#27ae60,color:#fff
    style KVM_MOD fill:#e67e22,color:#fff
```

### 10.2 SMMU 虚拟化（Stage-2 Translation）

SMMU 支持两阶段地址翻译，配合 Hypervisor 实现外设直通（VFIO/IOMMU）：

```mermaid
graph LR
    DEV["外设 DMA\n(Guest StreamID)"]
    S1["Stage-1 翻译\nGuest IOVA → IPA\n(Guest OS 配置)"]
    S2["Stage-2 翻译\nIPA → PA\n(Hypervisor 配置)"]
    MEM["物理内存"]

    DEV -->|IOVA| S1 -->|IPA| S2 -->|PA| MEM

    SMMU_NS["SMMU NS 寄存器\nNSMAP:允许 Guest OS\n更新 Stage-1 页表"] --> S1
    SMMU_HYP["Hypervisor\n控制 Stage-2 页表\n限制 Guest 访问范围"] --> S2
```

---

## 十一、完整启动时序图

```mermaid
sequenceDiagram
    participant ROM as BL0 ROM (EL3-S)
    participant BL1 as BL1 (EL3-S)
    participant BL2 as BL2 (EL1-S)
    participant BL31 as BL31 EL3 Runtime
    participant TEE as BL32 OP-TEE (S-EL1)
    participant UBoot as BL33 U-Boot (EL2-NS)
    participant Kernel as Linux Kernel (EL1-NS)

    ROM->>ROM: CPU 复位，SRAM 初始化
    ROM->>BL1: 验证并跳转 BL1 (EL3)
    BL1->>BL1: PLL/时钟初始化，eMMC 驱动
    BL1->>BL2: 加载+验证 BL2，ERET→EL1-S
    BL2->>BL2: DDR 初始化 (PHY训练/ECC)
    BL2->>BL2: TZC-400 内存保护配置
    BL2->>BL31: 加载 BL31 镜像到 EL3
    BL2->>TEE: 加载 BL32 镜像到 Secure DRAM
    BL2->>UBoot: 加载 BL33 镜像到 NS DRAM
    BL2->>BL31: ERET→EL3 执行 BL31

    BL31->>BL31: GIC 初始化 (Distributor/Redistributor)
    BL31->>BL31: PSCI 框架注册
    BL31->>TEE: ERET→S-EL1 启动 OP-TEE
    TEE->>TEE: 安全内存/堆初始化
    TEE->>TEE: SMMU 安全流配置
    TEE->>TEE: TA 框架初始化
    TEE->>BL31: SMC(OPTEE_RETURN_FROM_RPC) 通知完成
    BL31->>UBoot: ERET→EL2 NS 执行 U-Boot

    UBoot->>UBoot: MMU 初始化 (页表+Cache 开启)
    UBoot->>UBoot: eMMC/USB/网络驱动
    UBoot->>UBoot: 从存储加载 Image + DTB
    UBoot->>UBoot: 关闭 MMU/Cache (cleanup_before_linux)
    UBoot->>Kernel: br kernel_entry (x0=dtb)

    Kernel->>Kernel: head.S: 异常等级检测/KASLR
    Kernel->>Kernel: 建立初始页表 (idmap+init)
    Kernel->>Kernel: 开启 MMU (TTBR0/TTBR1)
    Kernel->>Kernel: start_kernel(): GIC 驱动注册
    Kernel->>Kernel: SMMU 驱动初始化
    Kernel->>Kernel: 设备驱动、文件系统挂载
    Kernel->>Kernel: init 进程启动 (EL0)
```

---

感谢阅读！
