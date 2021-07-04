---
title: 在 Ubuntu 上安装 CUDA
date: 2020-12-31 19:59:48
categories:
- [Linux]
- [Docker]
- [ML]
tags:
- NVIDIA
- CUDA
- Ubuntu
---

# 前言

老是忘记怎么在 Ubuntu 上安装 CUDA，每次有新机器都要在环境恶劣的互联网上寻找各种不靠谱又容易炸的教程。还是简单做个总结，关于如何安装炼丹人必备的依赖 `CUDA`。

## 一些链接

- CUDA GPU Capability https://developer.nvidia.com/cuda-gpus
- CUDA Zone https://developer.nvidia.com/CUDA-zone

# 安装流程

## 安装前准备

先看看机器的版本号啥的

```bash
sudo lsb_release -a

No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.2 LTS
Release:        18.04
Codename:       bionic
```

再看看显卡在不在

```bash
lspci | grep NVIDIA

18:00.0 3D controller: NVIDIA Corporation Device 1eb8 (rev a1)
af:00.0 3D controller: NVIDIA Corporation Device 1eb8 (rev a1)
```

再换个源。机器在深圳，用科大了

```bash
sudo sed -i 's/archive.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
```

## 安装

### Step 1：安装 `ubuntu-drivers` 并配置驱动

```bash
sudo apt update
sudo apt install ubuntu-drivers-common -y
```

运行

```bash
ubuntu-drivers devices

== /sys/devices/pci0000:17/0000:17:00.0/0000:18:00.0 ==
modalias : pci:balabalabala
vendor   : NVIDIA Corporation
driver   : nvidia-driver-440-server - distro non-free
driver   : nvidia-driver-450-server - distro non-free
driver   : nvidia-driver-455 - distro non-free recommended
driver   : nvidia-driver-450 - distro non-free
driver   : nvidia-driver-418-server - distro non-free
driver   : xserver-xorg-video-nouveau - distro free builtin
```

装那个 recommended 的

```bash
sudo apt install nvidia-driver-455
```

<!--more-->

### Step 2：安装 CUDA

去官网看看怎么下载 [https://developer.nvidia.com/cuda-downloads](https://developer.nvidia.com/cuda-downloads)

可惜太慢了，可以用 **阿里云** 的源 [https://developer.aliyun.com/mirror/nvidia-cuda](https://developer.aliyun.com/mirror/nvidia-cuda)

官网选择 `deb(network)`，得到安装命令

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/cuda-ubuntu1804.pin
sudo mv cuda-ubuntu1804.pin /etc/apt/preferences.d/cuda-repository-pin-600
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/7fa2af80.pub
sudo add-apt-repository "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/ /"
sudo apt-get update
sudo apt-get -y install cuda
```

替换下源到阿里云

```bash
wget https://mirrors.aliyun.com/nvidia-cuda/ubuntu1804/x86_64/cuda-ubuntu1804.pin
sudo mv cuda-ubuntu1804.pin /etc/apt/preferences.d/cuda-repository-pin-600
sudo apt-key adv --fetch-keys https://mirrors.aliyun.com/nvidia-cuda/ubuntu1804/x86_64/7fa2af80.pub
sudo add-apt-repository "deb https://mirrors.aliyun.com/nvidia-cuda/ubuntu1804/x86_64/ /"
sudo apt-get update
sudo apt-get -y install cuda
```

要是出现

```bash
cuda : Depends: cuda-11-2 (>= 11.2.0) but it is not going to be installed
```

删掉 NVIDIA 相关的程序重来

```bash
sudo apt clean
sudo apt update
sudo apt purge cuda
sudo apt purge nvidia-*
sudo apt autoremove
sudo apt install cuda
```

## Step 3：测试下

漫长的安装完成后，`reboot` ，运行 `nvidia-smi` 看看

```bash
nvidia-smi

Thu Dec 31 12:25:41 2020
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 460.27.04    Driver Version: 460.27.04    CUDA Version: 11.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla T4            Off  | 00000000:18:00.0 Off |                    0 |
| N/A   94C    P0    37W /  70W |      0MiB / 15109MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   1  Tesla T4            Off  | 00000000:AF:00.0 Off |                    0 |
| N/A   95C    P0    38W /  70W |      0MiB / 15109MiB |      5%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

# 妙啊

# 补充

感觉好像只要运行二、三步就行了
