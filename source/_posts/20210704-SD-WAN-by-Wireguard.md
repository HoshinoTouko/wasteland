---
title: 使用 WireGuard 多地组网及 WireGuard-UI 登录权限控制实践
date: 2021-07-04 23:33:00
categories:
- [Network， SD-WAN]
tags:
- NAT Traverse
- SD-WAN
- WireGuard
---

# 前言

近期在学校机房上架了数台新机器，算上之前文章 [优雅地管理内网集群](https://wasteland.touko.moe/blog/2020/04/intranet-cluster-management/) 提到的，已经有约三十台不同地区的机器了。解决机器的管理、远程登录、权限验证等问题迫在眉睫。正好从友人 nasbdh9 处了解到，Wireguard 可以比较好地实现多地组网的问题。我立刻上手研究。该文章总结了如何在实现了三地组网+VPN远程登录，并且有较好的用户管理方案。

# 前期准备

首先梳理使用 WireGuard 做异地组网的逻辑，确认网络拓扑、各地网关和各子网之间的关系。然后，确认需要使用到的软件和硬件，最后上手部署。

## 网络拓扑



# References

主要参考文献

- WireGuard https://www.wireguard.com/
- WireGuard - Openwrt https://openwrt.org/docs/guide-user/services/vpn/wireguard/start
- OpenWrt / LEDE 安装 WireGuard，建立 VPN 隧道 https://steemit.com/cn/@curl/openwrt-lede-wireguard-vpn
- WireGuard —— 多用户配置教程 https://doubibackup.com/0k260c3z.html
- Wireguard小记 http://blog.home999.cc/2020/Wireguard%E5%B0%8F%E8%AE%B0/
- 通过 Wireguard 构建 NAT to NAT VPN https://anyisalin.github.io/2018/11/21/fast-flexible-nat-to-nat-vpn-wireguard/
