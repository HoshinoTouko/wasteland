---
title: 基于 MQTT 将 OpenWRT 接入 Home Assistant
date: 2025-03-29 20:40:44
categories:
- [Network, Router]
tags:
- HomeAssistant
- Router
- Openwrt
- Homelab
- Xiaomi
---

# 前言

惊闻米家官方接入了 Home Assistant（HA），我也是连忙重新捡起被废弃了很久的 Home Assistant。由于我的 Homelab 经过了多次升级和架构调整，已经完成了存算完全分离的架构，与{% post_link raspi-as-router 之前的架构 %}完全不同。我也会在后期重新详细介绍现有的 Homelab 架构。

在本章中，我将着重介绍我如何通过 MQTT 和 luci-app-statistics 工具将 OpenWRT 的统计信息悉数同步至 HomeAssistant，做到如下的 Infrastructure 看板。

{% asset_img 0.dashboard.jpg Home Assistant 上的 OpenWRT 预览 %}

<!--more-->

## Home Assistant 安装和米家接入

由于 Homelab 架构的大部分程序运行于 Docker。因此，HA 的部署完全基于 Docker。这种架构保证了我运行的所有应用程序部署足够简单且极易维护。

启动一个 HA 实例只需要以下一个 `docker compose` 文件

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
```

推荐将 `/config` 文件夹持久化保存且对外暴露，保证后期维护方便。

[米家集成](https://github.com/XiaoMi/ha_xiaomi_home)的接入我选择通过 The Home Assistant Community Store (HACS)。这是一个社区维护的类似 HA 应用商店的东西。Docker 安装的 HA 需要进入 Docker Container 的 bash 进行手动安装。

```bash
docker exec -it '<name of the container running homeassistant>' bash
wget -O - https://get.hacs.xyz | bash -
```

安装完成后重启 HA，HACS 即生效。

在 HACS 中搜索 `Xiaomi`，安装插件

{% asset_img 1.hacs-xiaomi-install.jpg HACS 安装米家官方集成 %}

重启 HA。在 Setting - Devices 中添加 Xiaomi 插件，一路下一步，登录自己的手机账号，处理好验证接口的回调跳转，几乎所有米家设备都能无缝接入。

## 接入 OpenWRT 数据

HA 设计之初是作为家庭设备的管理中枢，统一管理所有 IoT 设备。路由器在某种意义上也是一个 IoT 设备，我认为一定有人造过相关的轮子，允许 HA 接收 OpenWRT 的上报数据。在理论上，想要实现这个效果，有两种方法：（1）HA 主动轮询 OpenWRT，采集数据；（2）OpenWRT 主动定时上报数据给 HA。经过我的搜索，数据较为详细、自定义程度更高的方案，是由 OpenWRT 主动上报数据给一个 Message Queuing Telemetry Transport（MQTT）服务器，HA 再主动从 MQTT 服务器上获取数据。

### MQTT 服务器安装

[参考](https://www.home-assistant.io/installation/#advanced-installation-methods)，如果是 HA OS 安装方法或 HA Supervised 安装方法，能够直接从 Addon 中安装 MQTT 服务器。由于我使用的是 Container 版本的 HA，需要额外安装 MQTT 服务器，并保持 HA - MQTT 服务器 - OpenWRT 的网络联通。这在我使用的 Docker 部署方案上并不是什么难题。

MQTT 服务器的 Docker compose 文件样例

```yaml
version: "3.7"
services:
  # mqtt5 eclipse-mosquitto
  mqtt5:
    image: eclipse-mosquitto
    container_name: mqtt5
    ports:
      - 1883:1883
      - 9001:9001
    volumes:
      - ./config:/mosquitto/config:rw
      - ./data:/mosquitto/data:rw
      - ./log:/mosquitto/log:rw
    restart: unless-stopped
```

配置密码用于 HA 和 OpenWRT 读写。

```bash
docker exec -it mqtt5 sh
mosquitto_passwd -c /mosquitto/config/pwfile mqtt_openwrt # 配置 OpenWRT 访问的用户名和密码
mosquitto_passwd -c /mosquitto/config/pwfile mqtt_ha # 配置 HA 访问的用户名和密码
```

### OpenWRT 配置

#### 软件包配置

OpenWRT 上需要安装以下依赖，用于识别系统的状态和上报消息，包括

- luci-app-statistics 
- collectd-mod-mqtt
- mosquitto-client-ssl
- collectd-mod-thermal
- collectd-mod-uptime
- collectd-mod-dhcpleases
- collectd-mod-ping
- collectd-mod-conntrack
- collectd-mod-iwinfo

直接使用 `apk install` 安装即可。

*小插曲*

我的 OpenWRT 还是老版本，并不支持基于 `apt` 的包管理。现有的 `luci snapshots` 镜像又基本都已经迁移到 apk 的版本了，导致我只能要么完全升级路由器，要么寻找一个仍然保持对 `luci snapshots` 提供支持的镜像。好消息是，我找到了。

```
src/gz openwrt_base https://mirror.math.princeton.edu/pub/openwrt/snapshots/packages/x86_64/base
src/gz openwrt_core  https://mirror.math.princeton.edu/pub/openwrt/snapshots/packages/x86_64/packages
src/gz openwrt_packages https://mirror.math.princeton.edu/pub/openwrt/snapshots/packages/x86_64/packages
src/gz openwrt_routing https://mirror.math.princeton.edu/pub/openwrt/snapshots/packages/x86_64/routing
src/gz openwrt_telephony https://mirror.math.princeton.edu/pub/openwrt/snapshots/packages/x86_64/telephony
```

该镜像依然保持了对 `luci snapshots` 的 `opkg` 安装方法支持，最后更新时间 2023年12月。谢谢美国人。

#### MQTT 连接

首先确认是否由图形界面。如果有图形界面的话，直接在图形界面配置。

{% asset_img 5.MQTT-config.jpg OpenWRT 上的 MQTT 配置入口 %}

如果没有图形界面，则需要进入 OpenWRT 的命令行，创建或进入 `/etc/collectd/conf.d` 文件夹，创建文件 `mqtt.conf`，写入以下配置文件。

```conf
LoadPlugin mqtt
<Plugin "mqtt">
  <Publish "OpenWRT">
    Host "MQTT_IP"
    Port "1883"
    User "mqtt_openwrt"
    Password "设置一个密码"
    ClientId "OpenWRT"
    Prefix "collectd"
    Retain true
  </Publish>
</Plugin>
```

设置的密码需要保持和 MQTT 服务器一致。

配置完成后，执行

```bash
service collectd restart
```

### 配置 HA

在 HA 中，添加 MQTT 插件，并配置 IP、端口、用户名密码等。本文使用的用户名为 `mqtt_ha`。

添加完成并确认能够正常连接后，进入 HA 的配置文件 `configuration.yaml`，添加

```yaml
mqtt: !include openwrt.yaml
```

在同目录下新建一个 `openwrt.yaml`，[参考这里](https://github.com/lukdwo/OpenWRT-collectd-MQTT-HA/blob/main/openwrt.yaml) 写入配置文件。可以手动修改下面部分的信息，用于区分设备。

```yaml
    device:
            identifiers: WRT7800
            name: Router Netgear R7800
            model: R7800
            manufacturer: Netgear
```

设置完成后同样重启 HA，根据系统报错微调 `openwrt.yaml`。如果没有问题，HA 中将出现一大堆包含统计信息的实体。

## 数据美化思路

善于利用 HA 中的各种 Helper，能够实现数据的美化展示。例如连接数的数据，这里提供一个思路。

{% asset_img 2.connections.jpg 连接数数据展示 %}

想要实现上述效果，首先需要在 HA 的 yaml 中增加信息展示。以下为一个示例。

```yaml
  - name: Connections Percentage
    state_topic: collectd/OpenWRT/conntrack/percent-used
    unit_of_measurement: '%'
    value_template: "{{ value.split(':')[1].split('\0')[0] | int }}"
    unique_id: ap_connections_percentage
    state_class: measurement
    icon: 'mdi:percent'
    device:
      identifiers: OpenWRT
      name: OpenWRT
      model: OpenWRT
      manufacturer: Touko
  
  - name: Max Connections
    state_topic: collectd/OpenWRT/conntrack/conntrack-max
    unit_of_measurement: connections
    value_template: "{{ value.split(':')[1].split('\0')[0] | int }}"
    unique_id: max_ap_connections
    state_class: measurement
    icon: 'mdi:connection'
    device:
      identifiers: OpenWRT
      name: OpenWRT
      model: OpenWRT
      manufacturer: Touko
```

然后需要创建一个 Helpers 用于展示 `当前连接数 / 总连接数`，类型为 Template。State template 分别为

```yaml
{{ states('sensor.connections') }} / {{ states('sensor.max_connections') }}
```

{% asset_img 3.uptime.jpg Uptime 文本展示 %}

Uptime 同理。创建一个 Template，填入以下模板

```yaml
{% set uptime = now() - states('sensor.gnsh_main_gateway_uptime') | as_datetime %}
{% set hours = (uptime.total_seconds() // 3600) | int %}
{% set minutes = ((uptime.total_seconds() % 3600) // 60) | int %}
{% set seconds = (uptime.total_seconds() % 60) | int %}

{{ hours }}小时 {{ minutes }}分钟 {{ seconds }}秒
```

最后再手动配置 Dashboard，调整各种配色，就能实现想要的效果。


# References

- https://www.home-assistant.io/installation/linux
- https://hacs.xyz/
- https://www.home-assistant.io/integrations/
- https://mosquitto.org/documentation/authentication-methods/
- https://github.com/XiaoMi/ha_xiaomi_home
- https://github.com/lukdwo/OpenWRT-collectd-MQTT-HA

