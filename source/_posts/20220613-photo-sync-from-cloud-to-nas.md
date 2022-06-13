---
title: 保持云相册和 NAS 同步（iCloud & Google Photos）
date: 2022-06-13 11:53:39
tags:
- iCloud
- Google Photo
- NAS
---

# 前言

相册这种东西，堆着堆着就几万张了。粗略看了一下统计，我存放在 iCloud 上的相册中包含了 45k 张照片，在 Google Photos 上的相册有 30k 左右的照片。这些照片的成分也很复杂，包括日常拍照、屏幕截图和一堆纸片人图。因此，我需要一种方式管理这些照片。

我的初步想法是，将照片同步至本地 NAS，使用自建工具进行管理。我本地使用的是群晖。即使是群晖这种较为成熟的 NAS 系统，也只有和 Google Drive 进行同步的工具，无法和 iCloud 进行同步。甚至在 Google Drive 同步工具中，也无法和相册进行同步。好消息是很多个人开发者开发了用于同步相册的工具。本文将记录这些工具的部署流程和使用方法。

# 相册同步工具

## Google Photos

我搜索到两个主流的、已经封装好 Docker 镜像的 Google Photos 同步工具，分别是 [https://github.com/JakeWharton/docker-gphotos-sync](https://github.com/JakeWharton/docker-gphotos-sync) 和 [https://github.com/gilesknap/gphotos-sync](https://github.com/gilesknap/gphotos-sync)。其中，前者需要使用 `chromium-browser` 登陆谷歌账号进行验证，后者需要通过 GCP 创建 OAuth API 进行账户验证。我本人更喜欢后者的方式，因此选择后者作为本地使用的同步工具。

### 部署步骤

后者的配置方式在 [文档](https://gilesknap.github.io/gphotos-sync/main/index.html) 中有详细介绍，此处做个总结。

<!--more-->

1. 在 GCP 中创建项目：[https://console.cloud.google.com/cloud-resource-manager](https://console.cloud.google.com/cloud-resource-manager)。
2. 给项目 Enable Photo Library API：[https://console.cloud.google.com/apis/library/photoslibrary.googleapis.com](https://console.cloud.google.com/apis/library/photoslibrary.googleapis.com)。
3. 前往 `汉堡菜单-API-Credentials` 创建OAuth 2.0 Client IDs，Type 为 Desktop：[https://console.cloud.google.com/apis/credentials](https://console.cloud.google.com/apis/credentials)。中途会要求创建 `OAuth consent screen`，随便写，创建完成后将其 `Publish`。
4. 创建完成后，页面会允许下载一个包含了验证信息的 json 文件，修改文件名为 `client_secret.json`，放置在宿主机对应的 `/path/to/config/` 文件夹下。
5. [Docker 命令方法] 确认照片文件夹和 config 文件夹路径，编写初次运行的 Docker 命令，用于验证 Google App。`docker run --rm -v /path/to/config:/config -v /path/to/storage:/storage -p 18080:18080 -it ghcr.io/gilesknap/gphotos-sync /storage --port 18080 --skip-files --skip-albums --skip-index`。考虑到 8080 端口常被占用，更换了不常用端口。
6. [Docker compose 方法] 准备 `docker-compose.yml` 文件，参考下图。执行 `docker compose up` 。
7. 将控制台输出的 `https://accounts.google.com/o/oauth2/auth?xxxxx` 复制到浏览器，进行登录操作。回调会默认回调至 http://localhost:18080/ ，此时需要将 localhost 修改为 NAS 的 IP 或者域名，手动触发验证回调。浏览器显示 `The authentication flow has completed. You may close this window.` 时，说明验证完成。此时在对应 `/storage` 的文件夹下会生成 `.gphotos.token` 文件，用于保存用户验证信息。
8. 修改 `docker-compose.yml` 或 docker 执行脚本，然后定时执行命令即可。

### 参考 Docker Compose 文件

```yaml
version: '3.3'
services:
  gphotos-sync:
    container_name: gphotos-sync
    environment:
      - PYTHONUNBUFFERED=1
    volumes:
      - ./config/:/config:rw
      - /storage:/storage:rw
    image: ghcr.io/gilesknap/gphotos-sync
    networks:
      - gphotos
    ports:
      - 18080:18080
    command: /storage
    # For OAuth authentication.
    # command: /storage --port 18080 --skip-files --skip-albums --skip-index

networks:
  gphotos:
```

## iCloud Photos

iCloud 照片的下载我选择了 [iCloud Photos Downloader](https://github.com/icloud-photos-downloader/icloud_photos_downloader)。实际场景下我选择了第三方打包的 Docker：[https://github.com/boredazfcuk/docker-icloudpd](https://github.com/boredazfcuk/docker-icloudpd)。相比于 Google Photos，iCloud Photos Downloader 的配置更为简单。直接编写 `docker-compose.yml` 后执行 `docker compose up`，配置密码和 2FA。之后执行 `docker compose up -d`，程序会以 `synchronisation_interval` 设定的间隔定期更新照片库。

```yaml
version: '3.3'
services:
  icloudpd:
    container_name: iCloudPD-boredazfcuk
    networks:
      - icloudpd
    restart: always
    environment:
      - user=USER_NAME
      - user_id=USER_ID
      - group=USER_GROUP
      - group_id=GROUP_ID
      - apple_id=APPLE_ID
      - authentication_type=2FA
      - notification_type=Telegram
      - telegram_token=TELEGRAM_TOKEN
      - telegram_chat_id=CHAT_ID
      - folder_structure={:%Y}
      - auto_delete=True
      - notification_days=14
      - synchronisation_interval=43200
      - TZ=Asia/Hong_Kong
    volumes:
      - /path/to/config:/config
      - /path/to/photos:/home/USERNAME/iCloud
    image: boredazfcuk/icloudpd

networks:
  icloudpd:
```

配置完这些自动下载工具后，下一步就是管理照片。我使用了 `PhotoPrism`，一个 `self-hosted` 的、带 AI 分类功能的图片管理工具。相关配置就是后话了。

# References

- https://github.com/gilesknap/gphotos-sync
- https://gilesknap.github.io/gphotos-sync/main/index.html
