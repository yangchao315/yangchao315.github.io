---
layout: post
title: "K230 SDK 编译流程分析：目录结构、Makefile 入口与 Buildroot 集成"
date:   2026-03-27
tags: [K230]
comments: true
author: yangchao
---

K230 是嘉楠科技基于玄铁 C908 双核的 RISC-V AI 视觉 SoC。本文分析 K230 SDK 的编译流程，覆盖目录布局、Makefile 入口、Buildroot 集成方式以及最终输出产物。

<!-- more -->

- [SDK 目录结构](#sdk-目录结构)
- [Makefile 入口分析](#makefile-入口分析)
- [Buildroot 集成方式](#buildroot-集成方式)
  - [BR2_EXTERNAL 机制](#br2_external-机制)
  - [自定义 package 示例](#自定义-package-示例)
- [编译输出产物](#编译输出产物)
- [常用编译命令](#常用编译命令)

---

## SDK 目录结构

K230 SDK 顶层目录布局如下：

```
k230_sdk/
├── Makefile                  # 顶层入口，核心编译驱动
├── configs/                  # defconfig（如 k230_evb_defconfig）
├── board/                    # 板级配置文件、设备树、启动脚本
│   └── k230/
│       ├── env.env           # U-Boot 环境变量
│       └── genimage.cfg      # 镜像打包配置
├── buildroot-ext/            # BR2_EXTERNAL 扩展目录
│   ├── Config.in             # 顶层 Kconfig 入口
│   ├── external.mk           # 顶层 make 入口
│   ├── package/              # 自定义 package（AI 库、驱动、应用）
│   └── configs/              # Buildroot defconfig
├── src/
│   ├── little/               # 小核（Linux + Buildroot）
│   │   └── buildroot/        # Buildroot 源码（submodule）
│   ├── big/                  # 大核（RT-Thread 或裸机）
│   │   └── rtsmart/          # RT-Smart 源码
│   └── uboot/                # U-Boot 源码
├── tools/                    # 镜像打包、烧录工具
└── output/                   # 编译产物目录
    ├── images/               # 最终镜像
    └── build/                # 中间产物
```

K230 采用双核异构架构：小核运行 Linux（Buildroot 构建），大核运行 RT-Smart（实时操作系统），两者通过共享内存 + mailbox 通信。

---

## Makefile 入口分析

顶层 `Makefile` 的核心逻辑如下：

```makefile
# 加载 defconfig，确定 ARCH、CROSS_COMPILE 等变量
include $(BR2_CONFIG)

# 小核编译：调用 Buildroot
little:
	$(MAKE) -C src/little/buildroot \
		BR2_EXTERNAL=$(SDK_ROOT)/buildroot-ext \
		O=$(OUTPUT)/little \
		$(LITTLE_DEFCONFIG)
	$(MAKE) -C src/little/buildroot \
		O=$(OUTPUT)/little

# 大核编译：调用 RT-Smart
big:
	$(MAKE) -C src/big/rtsmart \
		CROSS_COMPILE=$(RISCV_BIG_CROSS) \
		O=$(OUTPUT)/big

# U-Boot 编译
uboot:
	$(MAKE) -C src/uboot \
		CROSS_COMPILE=$(RISCV_CROSS) \
		k230_evb_defconfig
	$(MAKE) -C src/uboot CROSS_COMPILE=$(RISCV_CROSS)

# 打包镜像
image: little big uboot
	python3 tools/gen_image.py \
		--config board/k230/genimage.cfg \
		--output output/images/sysimage-sdcard.img
```

编译顺序：`uboot` → `big`（RT-Smart）→ `little`（Buildroot/Linux）→ `image`（打包）。

顶层 `make` 不加参数时默认执行 `all`，等价于完整编译并打包。可单独执行 `make little` 仅编译小核部分。

---

## Buildroot 集成方式

### BR2_EXTERNAL 机制

K230 SDK 通过 Buildroot 的 `BR2_EXTERNAL` 机制将 SDK 特有 package 和配置注入 Buildroot，而不修改 Buildroot 主线源码。

`BR2_EXTERNAL` 目录须包含以下固定文件：

```
buildroot-ext/
├── Config.in        # 必须：声明 source "$BR2_EXTERNAL_K230_PATH/package/.../Config.in"
├── external.mk      # 必须：include 所有自定义 package 的 .mk 文件
└── external.desc    # 必须：描述此 BR2_EXTERNAL 的 name 和 desc
```

`external.desc` 示例：

```
name: K230
desc: Canaan K230 SDK external tree
```

Buildroot 在调用时通过 `O=` 指定独立输出目录，源码树与产物分离，支持多配置并存。

### 自定义 package 示例

以 K230 AI 推理库 `k230_nncase_runtime` 为例，package 目录结构：

```
buildroot-ext/package/k230_nncase_runtime/
├── Config.in
└── k230_nncase_runtime.mk
```

`Config.in`：

```kconfig
config BR2_PACKAGE_K230_NNCASE_RUNTIME
	bool "k230_nncase_runtime"
	help
	  nncase runtime library for K230 KPU inference.
```

`k230_nncase_runtime.mk`：

```makefile
K230_NNCASE_RUNTIME_VERSION = 1.0.0
K230_NNCASE_RUNTIME_SITE = $(SDK_ROOT)/src/little/nncase
K230_NNCASE_RUNTIME_SITE_METHOD = local

define K230_NNCASE_RUNTIME_BUILD_CMDS
	$(MAKE) -C $(@D) CROSS_COMPILE=$(TARGET_CROSS)
endef

define K230_NNCASE_RUNTIME_INSTALL_TARGET_CMDS
	$(INSTALL) -D -m 0755 $(@D)/libnncase.so $(TARGET_DIR)/usr/lib/
endef

$(eval $(generic-package))
```

---

## 编译输出产物

完整编译后，`output/images/` 目录包含：

| 文件 | 说明 |
|------|------|
| `sysimage-sdcard.img` | SD 卡完整镜像（含分区表、U-Boot、内核、rootfs） |
| `u-boot.bin` | U-Boot 二进制 |
| `Image` / `Image.gz` | Linux 内核镜像 |
| `k230-evb.dtb` | 设备树 blob |
| `rootfs.ext4` | 小核 Linux rootfs（ext4） |
| `rtsmart.img` | 大核 RT-Smart 镜像 |
| `env.bin` | U-Boot 环境变量二进制 |

`sysimage-sdcard.img` 的分区布局（典型配置）：

```
Offset 0       : MBR 分区表
Partition 1    : U-Boot（raw，不格式化）
Partition 2    : U-Boot env
Partition 3    : Linux 内核 + DTB（FAT32）
Partition 4    : rootfs（ext4）
```

---

## 常用编译命令

```bash
# 完整编译（首次或 clean 后）
make CONF=k230_evb_defconfig

# 仅编译小核（Linux + Buildroot）
make little

# 仅重新打包镜像（不重编代码）
make image

# 单独重编某个 Buildroot package
make -C src/little/buildroot O=$(pwd)/output/little k230_nncase_runtime-rebuild

# 清理输出
make clean

# 烧录 SD 卡镜像（Linux 主机）
sudo dd if=output/images/sysimage-sdcard.img of=/dev/sdX bs=4M status=progress
```

---
感谢阅读！
