---
title: 我的第一台黑苹果主机
date: 2020-11-21 21:25:54
categories:
- [Hackintosh]
tags:
- Hackintosh
---

# 前言

近期，拼拼凑凑从各类垃圾中组装出了一台轻便的 ITX 主机。主板是真白丢给我的，在他那不能工作的 `华擎 H110M-ITX` ；CPU 是我之前测试 QL3X 时点亮用的 `Pentium G4500T` 6 代低压 U；电源是 tb 买的韩国牌子 `全汉代工 450W SFX` ；机箱是 `酷鱼 T40` ；内存是 `影驰 镭 RGB 灯条 3200MHz 8G×2`；硬盘是以前捡的 `昂达 480G SSD`，显卡是亮机卡 `NVIDIA Quadro K600`。这么一套东西装起来，我就想，能不能搞个黑苹果玩玩。于是，查了一段时间的资料，准备动手。

> 本文仅记录点亮黑苹果的过程。其他驱动的修改预计会在后续记录。

## 配置价格

装机之前，大致估算一下目前这套机器的价格。散热我用的酷冷十几块的（AMD 原装）散热铝块，就不计入总价了。

|               |                               |      |
|:-------------:|:-----------------------------:|:----:|
|      主板     |        ASRock H110M-ITX       |  150 |
|      CPU      |         Pentium G4500T        |  230 |
|      内存     | 影驰 镭 RGB 灯条 3200MHz 8G×2 |  400 |
|      电源     |       全汉代工 450W SFX       |  130 |
|      硬盘     |         昂达 480G SSD         |  240 |
| 机箱 & 延长线 |            酷鱼 T40           |  140 |
|      显卡     |       NVIDIA Quadro K600      |  140 |
|      总价     |                               | 1430 |

其中内存可以不用这么好的，CPU 可以再上好一些的，显卡可以换矿卡（等矿难）。整机价格偏高，不过既然都是捡来的垃圾，那就无所谓性价比了。

## 上机图片

酷鱼 T40 的设计十分紧凑，附带扶手，单手就能轻松提起。机器显卡为倒装，很有意思。由于是非模组电源，走线较为凌乱。

<!--more-->

{% asset_img 1.case_graphic_card.JPG 机箱 显卡面 %}

{% asset_img 2.case_mb.JPG 机箱 主板面 %}

{% asset_img 3.case_view.JPG 机箱 全览 %}

# 安装准备

安装黑苹果首先需要准备一块大号U盘（至少32G）。我用了一块从 Surface 上拆下来的 `PM961` + `ACASIS NVMe 硬盘盒` 作为安装介质。不能使用 SD 卡。一开始我使用的是 SD 卡 + 读卡器，结果一直无法通过 Etcher 写入 dmg 文件后的镜像校验。

{% asset_img 4.nvme.JPG NVMe 和硬盘盒 %}

其次，需要准备 安装介质写入工具。我使用的是 `balenaEtcher` 作为写入工具。

同时最重要的，带引导的黑苹果镜像。我主要从

- 黑苹果星球 https://heipg.cn/
- 黑果小兵 https://blog.daliansky.net/

上下载镜像。

这类镜像一般由几个分区组成。OpenCore / Clover 引导分区，白苹果分区，WePE 分区（用于应急）。需要安装黑苹果的电脑由 OpenCore 引导启动（Clover 底层也是 OpenCore 了），模拟 Mac 相关硬件，最后启动白苹果系统。

## 镜像烧录

此处以从 heipg.cn 上下载的最新版本的 `Catalina` 为例。下载镜像，并使用 `balenaEtcher` 烧录进 U 盘。

{% asset_img 6.etcher.png balenaEtcher 烧录 %}

确认 Etcher 烧录没有报错后，弹出 U 盘并重新插入，准备修改 EFI。

## 微调 EFI

此类打包好的镜像文件一般集成了现有大多数常用驱动程序。我们首先要做的是点亮系统，再考虑其他硬件的运行情况。最开始安装黑苹果的过程中，我没有考虑 CPU 的影响，结果安装触发了 `PANIC`。经各种资料查阅，发现是由于我使用的奔腾 CPU 并不被苹果官方支持，需要微调 EFI，让 OC / Clover 欺骗苹果系统。

EFI 分区在 Windows 下不默认挂载。我们需要使用 `diskpart` 工具将其挂载至硬盘驱动器目录中。打开**带管理员权限的** `PowerShell (Admin)`，执行

```powershell
diskpart
```

进入 diskpart 后，

```powershell
DISKPART> list disk

  Disk ###  Status         Size     Free     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  Disk 0    Online          447 GB      0 B        *
  Disk 1    Online          931 GB  1024 KB        *
  Disk 2    Online          953 GB      0 B        *
  Disk 4    Online          238 GB   225 GB        *

```

选择 U 盘对应的 Disk

```powershell
DISKPART> select disk 4

Disk 4 is now the selected disk.

DISKPART> list partition

  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    System             200 MB    20 KB
  Partition 2    Primary             12 GB   200 MB
  Partition 3    Primary            380 MB    12 GB
```

选择对应分区。EFI 分区一般为 `200MB`。最后进入分区并分配盘符

```powershell
DISKPART> select partition 1

Partition 1 is now the selected partition.

DISKPART> assign letter=H

DiskPart successfully assigned the drive letter or mount point.
```

运行后，EFI 分区可以在电脑中看到。修改 `/EFI/OC/config.plist` 

对于 Clover，修改 `KernelCPU` 附近的代码为

```xml
<key>KernelCpu</key>
<false/>
<key>FakeCPUID</key>
<string>0x0306A0</string>
```

对于 OpenCore，修改 `Kernel - Emulate` 附近的代码为

```xml
<key>Cpuid1Data</key>
<data>oAYDAAAAAAAAAAAAAAAAAA==</data>
<key>Cpuid1Mask</key>
<data>/////wAAAAAAAAAAAAAAAA==</data>
```

以欺骗苹果系统，让其认为该 CPU 是 `Ivy Bridge 的 i5 CPU`。

由于我使用的是 Windows 系统，需要手撸配置文件。由于 OpenCore 这项的表示方法是 id + mask，并且是 **Little Endian** 的。所以对于 Clover 中的 `0x0306A0` ，需要将其转换为 A0 06 03 00，掩码 11 11 11 11，补齐后面 12 个 `00` 后，使用工具 https://cryptii.com/pipes/base64-to-hex 转换为对应的 Base 64 编码。

太麻烦了，还是用 Clover 吧。

设置结束后，不要忘了在 `diskpart` 中执行

```powershell
remove letter=H
```

取消挂载，避免占用盘符。

## BIOS 设置

BIOS 设置可以参考 https://heipg.cn/tutorial/basic-install-hackintosh-walkthrough.html 。基本思路是

- 关闭 CFG Lock
- 关闭 VT-d
- 关闭核显，使用独显（在我这套硬件配置下）
- 关闭 Fast Boot
- 关闭 Secure Boot
- 关闭 CSM

# 实机安装

插电，开机。由于我原有一个 Windows 系统，所以先进入 PE 删除原有系统。开机连点对应按键进入 Boot 菜单（我是华擎所以是 F11），第一个分区是 OpenCore，第二个是 WePE。

这一步总是报错然后卡住。最后我用 heipg.cn 双引导 Catalina 中 Clover 引导，终于进入了系统。

## 执行安装程序

{% asset_img 5.macos_loading.JPG MacOS loading %}

{% asset_img 7.macos_install_navi.JPG MacOS 安装初始界面 %}

如果顺利，等苹果 logo 读条完成后，就可以进入 MacOS 安装界面了。

首先选择磁盘工具，抹除原有磁盘上的所有数据。

{% asset_img 8.macos_install_format.JPG 磁盘格式化界面 %}

之后选择安装 MacOS，一路下一步即可。保持引导 U 盘插在电脑上，等待几次重启后就安装完毕了。

{% asset_img 9.macos_install.JPG MacOS 安装界面 %}

## 建立本机引导

安装完成，进入系统后，安装 `Clover Configuration`，然后通过 `Clover Configuration` 挂载隐藏的 EFI 分区，将 U 盘上的 EFI 分区下 EFI 文件夹复制到机器的 EFI 分区中。注意区分好哪个是机器的 EFI 分区，哪个是 U 盘的。

至此，可以关机，拔掉 U 盘，重新启动等待机器自动进入 MacOS 系统了。

{% asset_img 10.rebuild_EFI.JPG EFI 分区拷贝 %}

{% asset_img 11.OS_info.jpg 系统信息 %}

# References

- https://blog.daliansky.net/Updated-Frequently-Asked-Questions-in-Sierra-or-high-sierra.html
- https://www.mfpud.com/topics/1031/
- https://ark.intel.com/content/www/us/en/ark/products/90725/intel-pentium-processor-g4500t-3m-cache-3-00-ghz.html
- https://heipg.cn/tutorial/basic-install-hackintosh-walkthrough.html
- https://www.insanelymac.com/forum/topic/343348-transition-from-clover-to-opencore-with-pentium-g4560/
- https://oc.skk.moe/7-kernel.html
- https://cryptii.com/pipes/base64-to-hex
- https://www.bilibili.com/read/cv7941995/
