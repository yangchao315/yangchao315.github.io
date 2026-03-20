---
layout: post
title: "Radxa Cubie A7z VO"
date:   2026-1-15
tags:
    - Cubie A7z
    - A733
comments: true
author: yangchao
---

<!-- more -->
- [Cubie A7z VE](#cubie-a7z-ve)
- [Cubie A7z VO](#cubie-a7z-vo)
  - [TCON\_LCD](#tcon_lcd)
  - [TCON\_TV](#tcon_tv)
  - [eDP1.4](#edp14)
    - [Link Training](#link-training)
  - [DSI](#dsi)
- [Sunxi DRM](#sunxi-drm)

## Cubie A7z VE

- VE 
Video Encoding consists of the video encoding unit(VE) and JPEG encoder(JPEG). The VE supports H.264 and H.265 encoding,JPEG supports JPEG/MJPEG encoding.

## Cubie A7z VO
![A733 VO Block](https://yangchao315.github.io/images/A733-VO-Block.png)

### TCON_LCD

TimingController_LCD(TCON_LCD),TCON_LCD0 and TCON_LCD1

![A733 VO TCON Block](https://yangchao315.github.io/images/A733-VO-TCON-Block.png)

### TCON_TV
TCON_TVisabridgemodulebetweenDEandHDMI_TX/EDP_TXtogeneratevideotiming.

- TCON_TV0controlsHDMIinterface
- TCON_TV1controlseDPinterface
![A733 VO TCON TV Block](https://yangchao315.github.io/images/A733-VO-TCON-TV-Block.png)

### eDP1.4
The embedded Display Port (eDP) is a standard protocol in the digital display field. It is
completelycompatiblewithDPandconsistsofmainlink,auxiliarychannel,andhotplugging.

![A733 VO eDP Block](https://yangchao315.github.io/images/A733-VO-TCON-eDP-Block.png)

- Supports1-lane,2-lane,or4-lanetransmission,upto5.4Gbps/lane.
- Videoformats:RGB,YCbCr4:4:4,YCbCr4:2:2,andYCbCr4:2:0
- Colordepth:6-bit,8-bit,and10-bitperchannel
- SupportsHDCP1.4andHDCP2.3

#### Link Training
![A733 VO eDP Block](https://yangchao315.github.io/images/A733-VO-TCON-eDP-Block.png)


### DSI
DSI controller which is compliance with MIPI DSI specification V1.02 and a
D-PHYmodulewhichiscompliancewithMIPIDPHYspecificationV1.0.

![A733 VO DSI Block](https://yangchao315.github.io/images/A733-VO-TCON-DSI-Block.png)

- Supports up to 4 lanes
- OnlyDSI0supportsDSCfunction,supportingupto2000x1200@120fps
- Supportsnon-burstmodewithsyncpulse/syncevent,burstmode
- Supportscontinuouslaneclockmodeandnon-continuouslaneclockmode


**A733 Soc应用场景**


## Sunxi DRM
![A733 VO Block](https://yangchao315.github.io/images/A733-VO-Block.png)

Sunxi DRM 驱动由 Linux kernel DRM 框架（kernel/drivers/gpu/drm/）以及 bsp 仓库下 drivers/drm/ 目录下的全志平台适配驱动组成，通过 DRM 驱动实现了对全志显示硬件的管理控制。

DE 在 drm 中被抽象为 crtc，DE 具有的硬件图层合成能力，每个 blending pipe channel 被抽象为
一个 plane，各种硬件输出接口则被抽象为 connector，这些硬件的操作方法均通过 DRM 的标准 的 API 提供.

---

感谢阅读！
