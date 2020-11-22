---
title: Start with Hexo
date: 2020-03-14 00:18:51
categories: 
- [Web, Frontend]
- Blog
- Talk
tags: 
- Hexo
- Github
- Github Pages
- Web
---
# 一切的结束，一切的开始

我又搬博客了。在咕了两年零两个月之后，我终于继续找地方写我想写的东西了。  
此时应该放个烟花庆祝一下。如果你的城市不能放烟花，请打开一首打上花火。  

我把以前的博客封存了起来。在搬家时，我顺便翻了翻它。我发现每次搬家，都会再次发觉曾经的我的幼稚。当然人是要磨练的，在不断否定自己、改变自己后，才能在成长的路上艰难地向前挪动一步。

但我也会惊讶，惊讶于曾经渺小的自己爆发出来的强大的能量，和童言无忌口无遮拦的那些存粹的话。要是让我现在去修改以前的文章，可能要大段大段的砍掉让我现在念起来会面红耳赤的内容。我最终决定还是将他们悉数保留，也算是对我的青春的一种纪念。

## 鞭尸

有的文章实在值得我自己翻出来狠狠的抽自己不算厚脸皮，感受这种尴尬气氛下微妙的喉咙干燥感。以下文章全部摘自旧博客 `日常` 分类，并亲自附上吐槽。

- [我与开发的故事](https://savepoint.touko.moe/blog/whispers-my-way-to-developer)
  我没想到当时的我竟有勇气将 Link 的内容设置为 *my-way-to-developer*，因为现在的我越发没有自信称自己为开发者。我觉得我自己就是个写代码造软件赚点外快的。可能这就是小孩子的勇气吧。
- [我和typecho的第一次(](https://savepoint.touko.moe/blog/modify-typecho)
  一篇中规中矩的初级技术文章，伴随着一个自言自语式的尴尬的开场。当时的我竟然可以突破重重看不懂的代码，找到问题的核心所在，还能依葫芦画瓢写一些 "it works" 的 Apache 配置文件，实在难得。这样一个聪明的小孩，竟然没有在 SE 的道路上继续走下去，实在是可惜了。
- [再見，我的競賽生涯](https://savepoint.touko.moe/blog/deep-in-physics-and-maths)
  就这点数学物理知识还是不要说自己是打竞赛的好。我感觉以前学竞赛学的那点皮毛，现在我把自己按在椅子前学两天，也能把题目做的头头是道。
  哇，竟然还爱上杭州，半年前我提着行李箱顶着烈日在没地铁的浙大西溪校区边上走着的时候肯定无比后悔曾经说过的话。不过，只身一人从杭州前往深圳、中途中转广州的经历算是很深的记忆了。虽然他们可能远不及现在的日子精彩，但作为那时的我，也是一个值得纪念的挑战。

## 关于旧博客的存档

旧博客我还是做好了存档，网址是 https://savepoint.touko.moe/ 。也不知道当时是为何停止维护了，明明刚开了两个项目连载，分别是 MIPS CPU 和 bot-framework，结果最后也没能继续做下去。

在考虑旧博客命名时，我和 POJO 讨论了如下几个选项。

- archive
- legacy
- deprecated
- milestone
- pyramid
- savepoint
- ruin

这下可好，已经把未来数次的博客搬迁取名事宜考虑好了，以后可以尽情搬迁新博客而不用担心二级域不够用。

<!-- more -->

## 关于新博客取名

至于为啥叫 wasteland，大概只是因为它很帅。当我在考虑新博客取名时，随口问了句 POJO 我该用啥名好。

POJO: "teenage wasteland" 。

虽然我没了解过相关的歌，但是觉得这个名字实在有意思。鉴于我已经到了奔三的年龄，teenage 还是不必了，那么就取名为 **wasteland** 好了。我会把我想说的，有用的没用的，技术的非技术的，碎碎念还是吐槽啥的，一股脑丢到这。如果几年之后，这里的文章有人能够全读下来，那说不定对我的了解比我自己还深。

{% asset_img image-20200314121336505.png Google Translator %}

是的，谷歌翻译很酷

# 技术相关

## Savepoint - 迁移旧博客

### 让 Typecho 运行在 PHP 7

旧博客我用的是运行于 *PHP* 的 *Typecho* ，甚至一度还是 0.9 版本，三年前的某天终于找了机会升级到了 1.0。我把整站数据库打包，PHP文件打包，装在了新的网站上。  
旧网站使用的是 *PHP 5.6 + Apache 2.4 + MySQL 5.6* 的组合，新网站已经全站切换至 *PHP 7.3 + NginX 1.17 + MySQL 5.6* 的组合。目前看来，*NginX* 用起来更顺手，并且资源消耗相比前一套系统要少一些。  

意料之内，在部署过程中遇上了数据库问题。更新 *MySQL* 账号密码后，页面仍然提示 `Database Server Error`，查看访问日志发现报错为 `PHP message: Adapter Typecho_Db_Adapter_Mysql is not available`。经排查，发现是由于 *Typecho* 默认使用的 *MySQL* 适配器 `Mysql` 在 *PHP 7* 环境下已经不再使用，只需要将 `config.inc.php` 文件进行如下修改，改为使用 `Pdo_Mysql` 进行数据库连接操作。

```php
// $db = new Typecho_db("Mysql","typecho_");
$db = new Typecho_db("Pdo_Mysql","typecho_");
```

还要对 *Typecho* 的伪静态进行迁移。由于前期我使用的是 *Apache*，现在使用 *NginX*，配置文件工作方式有一定变化。

```apache
<IfModule mod_rewrite.c>
RewriteEngine On

RewriteBase /
RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^(.*)$ /index.php/$1 [L]

RewriteCond %{HTTP_HOST} ^www.touko.moe 
RewriteRule (.*) http://touko.moe/$1 [R=301,L]

RewriteCond %{HTTPS} !^on$ 
RewriteRule (.*) https://%{SERVER_NAME}%{REQUEST_URI} [R=301] 

</IfModule>
```

上面是我曾经写下的 *Apache* 配置文件，主要做了三件事

- 将找不到的文件路径隐性转发给 `index.php` 处理请求 （伪静态）
- 将 `www.touko.moe` 301 重定向至 `touko.moe` 处理
- 将非 HTTPS 请求 301 重定向至 HTTPS（强制 HTTPS）

在 NginX 中，上述设置修改为如下形式

```nginx
if (!-e $request_filename) {
    rewrite ^(.*)$ /index.php$1 last;
}
if ($server_port !~ 443){
    rewrite ^(/.*)$ https://$host$1 permanent;
}
```

### SEO 相关

为了让曾经的链接可以正常访问，我做了一个小处理，让 *NginX* 妥善处理旧请求至旧博客存档点。由于只是域名发生了变化，只需要针对特定 URL Pattern 做 301 重定向即可。我检查了一下旧网站上文章出现的 URL ，找到了几个关键词，用以下规则完成转发

```nginx
rewrite ^/(category|blog|author|links|aboutme|archives|usr)(.*) http://savepoint.touko.moe/$1$2 permanent;
```

举个例子，当访问页面 https://touko.moe/blog/MonocycleCPU_MIPS 时，*NginX* 将匹配 `blog` 和 `/MonocycleCPU_MIPS` 作为参数 1 和参数 2，并 301 重定向至 `http://savepoint.touko.moe/{1}{2}` ，也就是 ``http://savepoint.touko.moe/{blog}{/MonocycleCPU_MIPS}` 这样就起到正常访问原网页的效果，并且对搜索引擎的影响也有一定的降低。

同时，因为我使用了 `Disqus` 评论系统，还需要在他们网站上进行站点转移。在 `Disqus` 设置页面的 `Configure Disqus for Your Site` 下，可以找到 `Changing domains? Learn how.` 在这里就可以设置一键转移了。

### Github Pages 反向代理

我偷懒使用 *Jekyll* 建立了我的 Under Construction 页面。因为需要转发旧博客的访问，希望使用我的 *NginX* 处理所有来自 https://touko.moe/ 的请求，而不是 CNAME。在设置 Github Pages 反向代理时，需要使用假的 header 欺骗 Github 服务器，避免 404 和被多次重定向。

```nginx
location / {
    proxy_pass https://hoshinotouko.github.io;
    proxy_redirect     off;
    proxy_set_header Host hoshinotouko.github.io;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header REMOTE-HOST $remote_addr;
}
```

其中，这句配置非常重要。

**`proxy_set_header Host hoshinotouko.github.io;`** 

这样一来，网站就能呈现如下组织方式。

```
touko.moe - Under construction (Will be a navi page in the future)
  |
  |- touko.moe/post/xxx -> savepoint.touko.moe/post/xxx
  |
  |- savepoint.touko.moe
  |
  |- wasteland.touko.moe
  |
  ...
```

## Wasteland - 新博客

我使用 *Hexo* 将网站部署在 *Github Pages* 上，解析为 https://wasteland.touko.moe/ 。

不得不说 *Hexo* 真是个方便的东西，日常使用只需要几个命令就能完成。

```bash
$ hexo clean // Delete all generated files
$ hexo g // Generate
$ hexo d // Deploy
$ hexo s // Local development server
```

我使用了爆款 [NexT 主题](https://theme-next.org/)，因为这个主题的确简单好看。我随意增加了几个插件，用于改善博客的开发和访问体验

- https://github.com/acwong00/hexo-addlink
- https://github.com/hexojs/hexo-deployer-git

### Automatically Deploy

可以通过在 *Hexo* 增加插件的方式，快速将静态页面部署至 *Github Pages* 。我使用了 *hexo-deployer-git* 插件。首先 `yarn add hexo-deployer-git` ，其次进行配置，在 `_config.yml` 中添加如下设置字段。

```yaml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: git@github.com:HoshinoTouko/wasteland.git
  branch: gh-pages
```

我把 *Github Pages* 的页面放在 *gh-pages branch* 下，*master branch* 用于存放网站的源代码。所以在上方配置文件中我这样写。但是要注意，使用插件进行更新，会触发 git 的 force push，尽量不要直接在生成的网页文件中进行修改。

#### Git SSH Key

因为我开了 Github 两步验证，每次部署都要收一下手机验证码，实在有点讨厌，所以我打算直接在本地创建一份 SSH Key 用于部署。我使用的是 Windows，Linux 和 Mac 的配置应该类似。

```bash
$ ssh-keygen -t rsa -C "your_email@mail.com"
$ eval $(ssh-agent -s)
$ ssh-add /c/Users/{USERNAME}/.ssh/id_rsa
$ clip < /c/Users/{USERNAME}/.ssh/id_rsa.pub
```

然后将剪贴板中的公钥上传至 `Github - Settings - SSH Keys` ，就可以直接通过 `hexo d` 部署网页了。

### CNAME of Github Pages

上文提到，`hexo d` 的更新会进行一次 force push，因此在 Github Setting 页面进行 CNAME 的设置会被下一次 force push 覆盖。为了解决这个问题，需要创建文件 `source/CNAME` 并设置为

```
wasteland.touko.moe
```

之后进行 `hexo g && hexo d` 操作时，CNAME 记录就会被自动保存。

### CloudFlare 面板访问问题

我将域名解析由 *DNSPod* 全部切换至 *CloudFlare* 。设置好 CF 的 HTTPS 访问后，我发现我的服务器面板（非 80 / 443 端口）无法访问了。经检查，是由于如下原因

> 需要**注意**的是，**中国境内**的 HTTP/HTTPS 流量节点**只支持 80 和 443 端口**。

倒也不用担心，只需要将其关闭，或者挂梯子访问就可以了。

# References

- https://www.typechodev.com/case/Adapter-Typecho_Db_Adapter_Mysql-is-not-available.html
- https://zhuanlan.zhihu.com/p/26625249
- https://www.jianshu.com/p/9317a927e844
- https://blog.csdn.net/JOYIST/article/details/90514991
- https://www.jianshu.com/p/8b25564d3ff3
- https://io-oi.me/tech/hexo-next-optimization/#%E9%83%A8%E7%BD%B2%E5%8D%9A%E5%AE%A2%E5%88%B0-github-pages
- https://isdaniel.github.io/hexo-blog-theme/
