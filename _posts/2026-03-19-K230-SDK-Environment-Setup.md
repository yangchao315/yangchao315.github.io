---
layout: post
title: "K230 SDK环境搭建"
date:   2026-03-19
tags:
    - K230
comments: true
author: yangchao
---

<!-- more -->
- [K230 SDK环境搭建](#k230-sdk环境搭建)
  - [前提条件](#前提条件)
  - [配置SDK环境](#配置sdk环境)
    - [更新国内源](#更新国内源)
    - [使用apt安装软件包](#使用apt安装软件包)
    - [修改pip为清华源](#修改pip为清华源)
    - [使用pip安装软件](#使用pip安装软件)
    - [安装微软软件包](#安装微软软件包)
    - [安装磁盘镜像工具](#安装磁盘镜像工具)
    - [清理缓存](#清理缓存)
    - [设置系统默认语言和字符编码](#设置系统默认语言和字符编码)
    - [创建工具链路径](#创建工具链路径)
  - [下载SDK](#下载sdk)
  - [编译SDK](#编译sdk)
    - [进入SDK根目录](#进入sdk根目录)
    - [下载toolchain](#下载toolchain)
    - [挂载工具链目录](#挂载工具链目录)
    - [编译SDK](#编译sdk-1)
  - [镜像文件说明](#镜像文件说明)

# K230 SDK环境搭建

本文详细介绍在 Ubuntu 20.04 系统上搭建 K230 SDK 开发环境的完整步骤。

## 前提条件

- **操作系统**: Ubuntu 20.04
- **默认用户名**: ubuntu
- **默认密码**: ubuntu
- **虚拟机**: VMware Workstation

> 如果没有 Ubuntu 环境，可以使用 VMware 虚拟机。VMware 下载链接：https://www.vmware.com/products/desktop-hypervisor.html

## 配置SDK环境

### 更新国内源

1. 打开 Ubuntu 20.04 系统自带的 **Software & Update** 软件
2. 点击 **Download from** 的下拉列表，找到 **Other** 选项
3. 选择国内的镜像源（以阿里云为例）
4. 输入虚拟机密码（默认：`ubuntu`）
5. 点击 **Close** 关闭选项框
6. 点击 **Reload** 重新加载镜像源

### 使用apt安装软件包

更新 apt 软件源：
```bash
sudo apt update
```

安装软件包：
```bash
sudo apt-get install -y --fix-broken --fix-missing --no-install-recommends \
sudo vim wget curl git git-lfs openssh-client net-tools sed tzdata expect mtd-utils inetutils-ping locales \
sed make binutils build-essential diffutils gcc g++ bash patch gzip bzip2 perl tar cpio unzip rsync file bc findutils \
dosfstools mtools bison flex autoconf automake \
libc6-dev-i386 libncurses5:i386 libssl-dev \
python3 python3-pip python-is-python3 \
lib32z1 scons libncurses5-dev \
kmod fakeroot pigz tree doxygen gawk pkg-config libyaml-dev libconfuse2 libconfuse-dev cmake
```

### 修改pip为清华源

编辑 pip 的配置文件，设置全局的 pip 配置选项：
```bash
sudo vi /etc/pip.conf
```

修改内容为：
```ini
[global]
timeout = 60
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
extra-index-url = https://mirrors.aliyun.com/pypi/simple/ https://mirrors.cloud.tencent.com/pypi/simple
```

### 使用pip安装软件

```bash
python3 -m pip install -U pyyaml pycryptodome gmssl \
numpy==1.19.5 protobuf==3.17.3 Pillow \
onnx==1.9.0 onnx-simplifier==0.3.6 onnxoptimizer==0.2.6 onnxruntime==1.8.0 cmake
```

### 安装微软软件包

1. 下载 Microsoft 软件包：
```bash
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
```

2. 安装软件包：
```bash
sudo dpkg -i packages-microsoft-prod.deb && rm packages-microsoft-prod.deb
```

3. 更新软件源：
```bash
sudo apt-get update
```

4. 安装 .NET 和 ICU 开发库：
```bash
sudo apt-get install -y dotnet-runtime-7.0 libicu-dev
```

### 安装磁盘镜像工具

1. 创建临时文件夹：
```bash
mkdir tmp
```

2. 下载磁盘镜像工具安装包：
```bash
wget https://github.com/pengutronix/genimage/releases/download/v16/genimage-16.tar.xz -O ./tmp/genimage-16.tar.xz
```

3. 解压安装：
```bash
cd tmp/
tar -xf genimage-16.tar.xz
cd genimage-16
./configure && make -j
sudo make install && cd ../../
```

### 清理缓存

```bash
sudo rm -rf /var/lib/apt/lists/*
```

### 设置系统默认语言和字符编码

```bash
sudo localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
```

### 创建工具链路径

```bash
sudo mkdir -p /opt/toolchain/
```

## 下载SDK

```bash
git clone https://e.coding.net/weidongshan/dshanpi-canmv/k230_sdk.git
```

## 编译SDK

### 进入SDK根目录

```bash
cd k230_sdk
```

### 下载toolchain

```bash
source tools/get_download_url.sh && make prepare_sourcecode
```

### 挂载工具链目录

```bash
sudo mount --bind $(pwd)/toolchain /opt/toolchain
```

### 编译SDK

```bash
make CONF=k230_canmv_dongshanpi_defconfig
```

> **注意**: SDK 不支持多进程编译，不要增加类似 `-j32` 多进程编译参数。

编译完成后，在 `output/xx_defconfig/images` 目录下可以看到编译输出产物。

## 镜像文件说明

`images` 目录下镜像文件说明如下：

| 镜像文件 | 说明 |
|---------|------|
| `sysimage-sdcard.img` | SD 和 eMMC 的非安全启动镜像 |
| `sysimage-sdcard.img.gz` | SD 和 eMMC 的非安全启动镜像压缩包，烧录时需要先解压缩 |
| `sysimage-sdcard_aes.img.gz` | SD 和 eMMC 的 AES 安全启动镜像压缩包，烧录时需要先解压缩 |
| `sysimage-sdcard_sm.img.gz` | SD 和 eMMC 的 SM 安全启动镜像压缩包，烧录时需要先解压缩 |

> 安全镜像默认不会产生，如果需要安全镜像请参考相关文档使能安全镜像。

- **大核系统**的编译产物放在 `images/big-core` 目录下
- **小核系统**的编译产物放在 `images/little-core` 目录下

---

感谢阅读！
