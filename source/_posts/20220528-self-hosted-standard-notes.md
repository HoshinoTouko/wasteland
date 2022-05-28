---
title: Standard Notes 部署小结和使用体验
date: 2022-05-28 01:14:36
categories:
- [WebApp]
tags:
- Note
- WebApp
---

# 前言

因为有私有记录笔记和维护版本的需求，我进行了一系列的调研，并最终选中了 [Standard Notes](https://standardnotes.com/) 和 [Outline](https://www.getoutline.com/) 两款产品。我先尝试上手部署了 Standard Notes 并试用，前后时间约一天。在本文中，我对其部署和使用做一个简单的记录。

近年来各类文档编辑工具类目繁多且各有特色。我最初接触的在线文档是石墨，后来随着需求和环境的变化，以及在线办公的普及，还曾使用了钉钉、企业微信等以中国大陆互联网企业为开发主体的文档工具，和 Notion 等更加国际化的文档。从使用的便捷程度和完善程度上说，我个人倾向于使用国际化的文档工具。

但是，我使用在线文档在大多数情况下是为了和协作者进行信息共享。因此在更多的时间里，我会使用例如 Typora 这类的本地 Markdown 编辑工具，加上 Syncthing 进行多端同步以保证数据可靠。随着 Typora 的收费和 Beta 版本的完全禁用，我决定寻找一个更安全、方便的文档编辑和同步方式。

# 部署

在部署部分，我着重记录官网文档中提及甚少的部分，如一些功能的不可用、插件的部署。

<!--more-->

## 基本流程

大致的部署流程都可以从[官网](https://standardnotes.com/)或[官方文档](https://docs.standardnotes.com/self-hosting/getting-started)处找到。一般来说，直接 clone 官方的 `main` 分支仓库 `git clone --single-branch --branch main https://github.com/standardnotes/standalone.git` 并做好配置，即可直接通过 `docker compose` 启动所需的所有容器。

在 clone 完后，执行 `./server.sh setup`，程序会自动生成所需的 `.env`、`docker/auth.env` 和 `docker/api-gateway.env`，需要进入 `.env` 修改 `AUTH_JWT_SECRET`，同时进入 `docker/auth.env` 修改 `JWT_SECRET`、`LEGACY_JWT_SECRET``PSEUDO_KEY_PARAMS_KEY` 和 `ENCRYPTION_SERVER_KEY`。以上这些参数都可以通过 `openssl rand -hex 32` 生成。

需要执行程序时，直接执行 `docker compose up -d` 或者使用 `./server.sh start`。之后在 host 上做好反向代理，客户端就能直接连接。

整体的部署流程是平滑且容易上手的。Standard Notes 的开发团队对开源的支持非常友好，且不断在提升 self-hosted 的部署体验。

## 开启订阅

部署完成后的服务端是允许任意用户注册的，但是注册完成后，用户无法使用所有的高级功能，需要我们手动将对应用户升级为订阅账户以支持更多功能。官网的文档参考 [Subscriptions on your Standard Notes Standalone Server](https://docs.standardnotes.com/self-hosting/subscriptions) 。总结来说，需要在先前 clone 的 git 文件夹下执行以下命令。

```bash
docker compose exec db sh -c 'MYSQL_PWD=$MYSQL_ROOT_PASSWORD mysql $MYSQL_DATABASE'
```

然后在 MySQL Container 的 Shell 中执行，

```sql
INSERT INTO user_roles (role_uuid , user_uuid) VALUES ( ( select uuid from roles where name="PRO_USER" order by version desc limit 1 ) ,( select uuid from users where email="<EMAIL@ADDR>" )  ) ON DUPLICATE KEY UPDATE role_uuid = VALUES(`role_uuid`);

insert into user_subscriptions set uuid = UUID() , plan_name="PRO_PLAN" , ends_at = 8640000000000000, created_at = 0 , updated_at = 0,user_uuid= (select uuid from users where email="<EMAIL@ADDR>") , subscription_id=1 , subscription_type='regular';
```

## 扩展部署

官方提供的扩展介绍在 [https://docs.standardnotes.com/extensions/intro](https://docs.standardnotes.com/extensions/intro)，部署文档在 [Extensions Local Setup](https://docs.standardnotes.com/extensions/local-setup) 。为了应对 self-hosted 的部署，我构建了一个 gh-page 用于存储扩展所属的静态网页 [SNExts](https://snexts.github.io/)。这些扩展有的能够统计字数，有的能够支持更好的 Markdown + 富文本 编辑体验，取舍主要看个人喜好。

## 软件架构

经过部署和使用，我对该软件的整体架构做一个简单总结。在服务端，Standard Notes 的开发团队将软件拆分为多个微服务，以多个 Docker 部署，包括同步服务器、验证（Auth）服务器、数据库等。每个 Standard Notes 的 Client 连接到服务端时，都可以注册属于自己的用户信息（账号，密码）。账号和密码留存在服务端，客户端并不做任何账号密码的存储，因此，抛去笔记撰写客户端的外壳，本质上 Standard Notes 是一个一对多的、带加密的中心化文件同步工具。

## 客户端

Standard Notes 提供了跨平台的客户端，其中 Web 端和各桌面端均使用 Web 构建，甚至可以直接访问 https://app.standardnotes.com/ 以使用网页编辑器，这是非常方便的。移动端的编辑器目前较为简陋，但也在官方市场都有上架（Google Play & App Store）。

# 体验

## 使用体验

我试用了大约半天的时间，该软件的 UI 简洁且扁平，比较能够满足我的需求。现在的大多数流行的文本编辑软件都能做到这一点。 Standard Notes 将原本 "Folders" 插件整合至编辑器，支持以多级 Tag 形式给 Notes 打标签，对于 Note 管理来说较为方便。

Standard Notes 使用 Workspace 的概念，在每个不同的 Workspace 中，用户能够登陆不同站点的不同账号，也能在同一个 Workspace 中使用二次密钥以使用更私有的笔记存储空间。用户的笔记是以加密后的纯文本 JSON 文件存储的，在本地存储空间中，Standard Notes 贴心地提供了一个用于离线解密的网页。用户可以通过输入密码恢复加密的、打包好的备份笔记文件。**安全性有待商榷。**当然，他们还提供了二次加密的 `Private Workspace` 功能 [What are private workspaces?
](https://standardnotes.com/help/80)，但跨客户端支持存在一定问题。

## 缺陷

软件仍存在一定问题，特别是在 self-hosted 的场景下。Standard Notes 在 `Feb.10 2022` 前使用 [FileSafe](https://github.com/standardnotes/filesafe-relay) 插件用于图片和文件的插入，但是后续取消了该支持 [Deprecation Notice: FileSafe](https://blog.standardnotes.com/32429/deprecation-notice-filesafe)，转而使用 [Standard Notes Files Service](https://github.com/standardnotes/files)。但是该修改暂时还没有同步到 self-hosted 的开源项目中 \[Cite Github issue: [Unable to start Upload Session](https://github.com/standardnotes/standalone/issues/69)\]，但会尽快推进。这意味着目前，Standard Notes 的图片插入只能通过图床完成。

另外，扩展（Extension）的导入并不友好，这导致我需要使用一个额外的 Web 服务器以支持更多的扩展功能。虽然目前 gh-pages 可以解决大部分的插件导入问题，但仍比直接存入用户账户来的更为繁琐。希望能够优化。

# References

- https://www.blackvoid.club/standard-notes-docker-self-hosted-alternative/
- https://bitsly.org/posts/standard-notes/

# Acknowledgements

- Thank Icemic & Peter for assistance.
