---
title: 更换软路由：T620 Plus
date: 2021-10-23 23:28:10
categories:
- [Network, Router]
- [DIY]
tags:
- Router
- Openwrt
- Homelab
---

# 前言

拖延症是这样的，从抽屉里翻出了半年前买的、打算做软路由的 T620 Plus。这一拖就是半年，这次准备给它部署 Openwrt，替代目前使用的单口软路由。

T620 Plus 应该被不少人推荐过，包括 B 站 UP 主司波图 https://www.bilibili.com/video/BV1J54y1m7ar。这台机器的 CPU 性能超过 J1900，且带 AES 指令集，从性能上看绰绰有余，甚至可以安装一些虚拟化应用。

## 老家伙

之前我使用的软路由是 1037U + 定制亚克力外壳。1037U 作为软路由来说，性能已经足够，就算用来科学上网也能够跑到 500Mbps 左右。但我使用的定制外壳防尘性能较差，处理器散热器已经有一定的积灰，且有时候会莫名卡死。且当前软路由只有一个网口，我将其以单臂路由的形式下挂在连接到光猫的交换机上，链路重复利用，可能也会导致某些不可知的问题。因此，我将其更换为 T620 Plus。

{% asset_img old_1037u.jpg 老路由：1037U %}

# T620 Plus

我于半年前从闲鱼购入这台 T620 Plus，当时的价格为 RMB328。机器是普通小型桌面机器的大小，加装硬件部分全部采用免螺丝设计，非常好评。机器内部结构如下。

{% asset_img T620Plus内部1.jpg %}

翻开盖板后，可以看到内部结构，非常整洁。

<!--more-->

{% asset_img T620Plus内部2.jpg %}

机器附带一个 19V、90W 电源适配器，实际用不到这么高的功耗。

{% asset_img 电源适配器.jpg %}

## 加装网卡

机器本身自带一个网口，因此我又花 99 买了一张古老的 BCM5709 四口半高网卡。

{% asset_img NIC_BCM5709.jpg %}

安装网卡非常容易，注意卡扣要顶住 PCIe 挡板，避免网卡移动。

{% asset_img T620Plus加装网卡.jpg %}

安装后的网卡不好拔网线，具体可以从司波图的视频中看到。

## 性能测试

我在 Win 10 下测试了机器性能。

CPU-Z

{% asset_img CPU-Z.jpg %}

CPU-Z 跑分

{% asset_img CPU-Z_跑分.jpg %}

AIDA 64 内存和缓存测试
{% asset_img AIDA64_Cache_and_memory.jpg %}

AIDA 64 CPU + FPU 烤机测试，全程保持在最高 68 度，机器几乎没有噪音。烤机功耗 30W 左右，待机功耗 15-20W。

{% asset_img AIDA64_烤机.jpg %}

{% asset_img 功耗.png %}

# 总结

总体上看，这是一台功耗低、性能还不错、噪音小的小机器，加装网卡后非常适合作为软路由使用。如果有需要，甚至可以加装万兆网卡。唯一的缺点是体积，和专门设计用于软路由场景的工控机相比，T620 Plus 还是稍大。如果有软路由需求，该机器可以纳入考虑。
