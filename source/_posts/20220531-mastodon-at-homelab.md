---
title: 在 Homelab 中部署运行 Mastodon 并接入广域网
date: 2022-05-31 21:57:40
categories:
tags:
- Mastodon
- WebApp
- Homelab
---

# 前言

心心念念 [Mastodon](https://github.com/mastodon/mastodon) 将近两年了，昨天终于打起精神，尝试自己部署一个。作为一名十年微博用户，我对社交网络的需求已经逐渐减少，但仍希望有一个干净的、能够自由交流的工具。因此，我决定部署属于自己的 Mastodon 实例。

## 部署前准备

据我所知，Mastodon 实例对机器性能的要求较高，特别是对内存的需求。考虑这些，决定在 Homelab 部署 Mastodon 实例。但问题依然存在，为了维护双向关系，Mastodon 要求在互联网公开，因此我使用 [CloudFlare ZeroTrust Tunnel](https://www.cloudflare.com/products/zero-trust/zero-trust-network-access/) 将其公开至互联网。

机器我选用了长期吃灰的 昂达 H310SD3-ITX 作为平台，CPU 选用 Intel Pentium G4500T 2c/4t，内存为两条 8G DDR3 AMD 专用条。即使该平台比较老旧，但仍足够胜任一般的 Web 服务器需求。

## 关于这篇文章

Mastodon 的官方文档对使用纯 Docker 环境部署并不友好。因此，这篇文章主要对 Mastodon 使用 Docker Compose 部署做记录。当然也包括如何安全地将内网服务器暴露至公网。

# Mastodon 部署流程

经过摸索后，在集群内使用 Docker compose 起一个 Mastodon 实例仅需两个配置文件+少量部署操作。`docker-compose.yml` 如下，由官方 Github 仓库中获取并微调。

由于该配置文件直接上了 `production` 环境，会强制 301 到 HTTPS。在测试环境中需要 Web 服务器反代并做好 SSL 配置。我在本地使用 `traefik` 作为 Web 服务器。

<!--more-->

## .env.production 准备

使用官方提供的 `.env.production` （[https://github.com/mastodon/mastodon/blob/main/.env.production.sample](https://github.com/mastodon/mastodon/blob/main/.env.production.sample)） 文件作为模板修改。其中，`LOCAL_DOMAIN` 作为用户名后半段显示的域名，`WEB_DOMAIN` 作为 Mastodon 实例实际运行的域名。在测试部署时，可以使用 `ALTERNATE_DOMAINS` 允许其他临时域名。

举例说明，假设我希望用户名显示为 `@user@example.com`，而站点部署在 `mast.example.com` ，那我就设置 `LOCAL_DOMAIN=example.com`，`WEB_DOMAIN=example.com` 即可。如有该需求，需要在 example.com 所属的 Web 服务器中部署对应跳转规则。此处样例为 Nginx。

```conf
location /.well-known/webfinger {
  return 301 https://mast.example.com$request_uri;
}
```

其次，因为使用 docker compose 部署，并将所有容器连接至 `internal_network` ，因此无需将数据库的端口映射至宿主机。因此，在 `.env.production` 中可以这样配置：

```env
REDIS_HOST=redis
DB_HOST=db
ES_HOST=es
```

然后，默认的配置文件是将 s3 开启，es 功能关闭。其中 s3 用于存放静态文件，包括图片、头像、emoji、视频文件等，es 功能则用于全文搜索。由于我并不希望有额外的 s3 开销，寄希望于 CloudFlare CDN，因此我将 s3 关闭，并开启了 es 功能。这些功能的开关都在 .env.production 文件中显式标明。

在配置 `SECRET_KEY_BASE`、`OTP_SECRET`、`VAPID_PRIVATE_KEY` 和 `VAPID_PUBLIC_KEY` 时，需要启动一个临时容器用于密文生成。如 `docker run --rm -it tootsuite/mastodon:v3.5.3 bash` 后执行 `./bin/rails secret` 或 `./bin/rails mastodon:webpush:generate_vapid_key` 。注：此处如果使用 .end.production 内的 `rake` 命令会报错。

## Docker Compose 准备

下面的 docker compose 配置文件经过少量修改。截至发文，`mastodon:v3.5.3` 为最新 release 版本，因此我固定写入该版本。在 Mastodon Github 仓库中，默认会重新编译本地版本，在此我也注释了重新编译的部分代码。

同时，为了后期迁移方便，我对文件结构进行了调整，全部数据放在当前目录的 `data/` 文件夹下。部署中需要对文件夹权限进行微调。

```yaml
version: '3'
services:
  db:
    restart: always
    image: postgres:14-alpine
    shm_size: 256mb
    networks:
      - internal_network
    healthcheck:
      test: ['CMD', 'pg_isready', '-U', 'postgres']
    volumes:
      - ./data/postgres14:/var/lib/postgresql/data
    environment:
      - 'POSTGRES_HOST_AUTH_METHOD=trust'

  redis:
    restart: always
    image: redis:6-alpine
    networks:
      - internal_network
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
    volumes:
      - ./data/redis:/data

  es:
    restart: always
    image: docker.elastic.co/elasticsearch/elasticsearch-oss:7.10.2
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "cluster.name=es-mastodon"
      - "discovery.type=single-node"
      - "bootstrap.memory_lock=true"
    networks:
       - internal_network
    healthcheck:
       test: ["CMD-SHELL", "curl --silent --fail localhost:9200/_cluster/health || exit 1"]
    volumes:
       - ./data/elasticsearch:/usr/share/elasticsearch/data
    ulimits:
      memlock:
        soft: -1
        hard: -1

  web:
    # build: .
    image: tootsuite/mastodon:v3.5.3
    restart: always
    env_file: .env.production
    command: bash -c "rm -f /mastodon/tmp/pids/server.pid; bundle exec rails s -p 3000"
    networks:
      - external_network
      - internal_network
    healthcheck:
      # prettier-ignore
      test: ['CMD-SHELL', 'wget -q --spider --proxy=off localhost:3000/health || exit 1']
    # ports:
    #   - 3000:3000
    depends_on:
      - db
      - redis
      - es
    volumes:
      - ./data/public/system:/mastodon/public/system

  streaming:
    # build: .
    image: tootsuite/mastodon:v3.5.3
    restart: always
    env_file: .env.production
    command: node ./streaming
    networks:
      - external_network
      - internal_network
      - traefik
    healthcheck:
      # prettier-ignore
      test: ['CMD-SHELL', 'wget -q --spider --proxy=off localhost:4000/api/v1/streaming/health || exit 1']
    # ports:
    #   - 4000:4000
    depends_on:
      - db
      - redis

  sidekiq:
    # build: .
    image: tootsuite/mastodon:v3.5.3
    restart: always
    env_file: .env.production
    command: bundle exec sidekiq
    depends_on:
      - db
      - redis
    networks:
      - external_network
      - internal_network
    volumes:
      - ./data/public/system:/mastodon/public/system
    healthcheck:
      test: ['CMD-SHELL', "ps aux | grep '[s]idekiq\ 6' || false"]
  
  Cloudflare Zero Trust Tunnel
  cftunnel:
    image: cloudflare/cloudflared:2022.5.3
    restart: always
    command: tunnel --no-autoupdate run --token YOUR_CF_TUNNEL_TOKEN
    networks:
      - internal_network
    depends_on:
      - web


networks:
  external_network:
  internal_network:
    internal: true
```

## 环境部署

我将部署环境的过程分为以下几步

- 初始化数据库
- migrate 数据库
- 创建用户
- 赋予用户 admin 权限
- 设置存储位置的权限

首先初始化一个临时的 postgres 容器

```bash
docker compose run --rm db
```

然后开一个新终端，进入

```bash
docker exec -it mastodon-db-1 psql -U postgres
```

执行创建数据库用户和数据库命令

```bash
CREATE USER mastodon;
CREATE DATABASE mastodon_production owner=mastodon;
```

然后结束临时 postgres 容器，在当前文件夹继续执行初始化数据库命令

```bash
docker compose run --rm web rails db:migrate
```

至此，就可以执行 `docker compose up` 启动容器，并访问主页创建用户。在创建用户完成后，进入 web 实例，将新创建的用户设为管理员

```bash
docker exec -it mastodon-web-1 bash
# THEN
RAILS_ENV=production bin/tootctl accounts modify USERNAME --role admin
```

同时记得给文件夹设置权限。

```bash
# Storage
sudo chown -R 991:991 ./data/public/system
# ES
sudo chown 1000:1000 ./data/elasticsearch
```

## 配置 CloudFlare ZeroTrust Tunnel

在 docker compose 配置文件中，我还增加了 CloudFlare 的 Tunnel 容器，用于映射至外网。首先进入 CloudFlare Zero Trust Dashboard https://dash.teams.cloudflare.com/ ，从左侧 Access - Tunnels 入口进入，新建一个 Tunnel，并将 token 拷贝至 docker compose 配置文件。

当本地的 Tunnel Docker 容器连接上后，在页面配置对应转发域名。

| id | Hostname         | Path             | Service               |
|----|------------------|------------------|-----------------------|
| 1  | mast.example.com | api/v1/streaming | http://streaming:4000 |
| 2  | mast.example.com | *                | http://web:3000       |

streaming 容器在 `api/v1/streaming` 提供了一个 websocket 服务，一定注意其对应的路径优先级更高，CF 此处的优先级配置并不是最长前缀匹配。

至此，实例已经可以正确运行并经过公网访问。

# References

- .env.production 文件编写 https://docs.joinmastodon.org/admin/config/#basic
- CLI 控制命令 https://docs.joinmastodon.org/admin/tootctl/#cache
- https://github.com/mastodon/mastodon/issues/3676
- https://stackoverflow.com/questions/55279515/elasticsearchexception-failed-to-bind-service-error
- https://pullopen.github.io/%E5%9F%BA%E7%A1%80%E6%90%AD%E5%BB%BA/2020/10/19/Mastodon-on-Docker.html
- https://docs.joinmastodon.org/
- https://github.com/mastodon/mastodon/issues/7612
