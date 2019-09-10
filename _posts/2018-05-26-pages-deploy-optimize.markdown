---
layout:     post
title:      "博客部署优化实践"
subtitle:   "没有最快，只有更快"
date:       2018-05-26 17:00 +0800
author:     "gsfish"
header-img: "img/post-bg-07.jpg"
tags:
    - 运维
---


# 0x00 概述

本博客目前采用了两个国内外较为出名的代码托管平台提供的 Pages 服务，具体方案为：

* 境外线路 -> Github Pages
* 国内线路 -> Coding Pages

Github 和 Coding 都可以使用 Let's Encrypt 提供的 SSL/TLS 证书自动部署服务，且支持开启 HSTS。同时，对于当前方案的考虑是，一方面可以一定程度上地优化国内外用户的访问速度，另一方面还可以解决百度爬虫无法收录 Github Pages 的问题。


# 0x01 远程与本地配置

DNS 解析设置如下：

![01.png](/img/blog-deploy-optimize/01.png)

Github Pages 及 Coding Pages 的具体搭建过程此处略过，添加自定义域名并开启 HTTPS 即可。

Git 远程仓库配置如下，此处添加了 all 远程仓库，并分别设置了 Github 和 Coding 的仓库地址。如此以来只用 `git push` 就能一次性同步更新两个仓库了：

```
// 添加一个名为 all 的远程仓库，并设置仓库的原始 url
git remote add all git@github.com:gsfish/gsfish.github.io.git
// 为 all 添加两个 pushurl
git remote set-url --add --push all git@github.com:gsfish/gsfish.github.io.git
git remote set-url --add --push all git@git.coding.net:gsfish/gsfish.coding.me.git
// 将 git push 的上游设置为 all 的 master 分支
git push -u all master
```


# 0x02 Coding 重定向问题

使用之后发现了一个问题。以下为原本使用 Github Pages 时返回的相应，是一个 301 跳转，可直接跳转至博客页面：

```
HTTP/2 301 
server: GitHub.com
content-type: text/html
location: http://www.grassfish.net/
x-github-request-id: BAAC:6D17:24C5032:2834EAF:5B091C90
accept-ranges: bytes
date: Sat, 26 May 2018 09:18:25 GMT
via: 1.1 varnish
age: 2513
x-served-by: cache-lax8624-LAX
x-cache: HIT
x-cache-hits: 1
x-timer: S1527326306.892833,VS0,VE1
vary: Accept-Encoding
x-fastly-request-id: 6b5ccd78a634d30cdb150750115c7a77d32d0229
content-length: 178

<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

而以下为首次访问 Coding Pages 时返回的响应内容，是一个 302 跳转：

```
HTTP/2 302 
location: https://coding.net/pages/waiting?redirect=https%3A%2F%2Fwww.grassfish.net%2F
server: Coding Pages
set-cookie: splash=1; Domain=www.grassfish.net; Expires=Sun, 27 May 2018 08:54:27 GMT; HttpOnly
vary: Accept-Encoding
content-type: text/html; charset=utf-8
content-length: 99
date: Sat, 26 May 2018 08:54:27 GMT

<a href="https://coding.net/pages/waiting?redirect=https%3A%2F%2Fwww.grassfish.net%2F">Found</a>.
```

对于 301、302 跳转之间的区别可以参考[这篇博客](http://veryyoung.me/blog/2015/08/24/difference-between-301-and-302.html)，比较重要的一点是对于 301 跳转浏览器是可以缓存的。根据该 `location` 重定向的内容，Coding 会先放 5 秒钟的广告，然后才跳转到正常的博客页面。同时会设置一个 `splash=1` 的 Cookie 以记录当前用户的阅览状态。

毕竟是免费的东西，而自己又懒得维护一个独立的服务器，只能说有点失望咯。下一步考虑只将百度的线路解析至 Coding 用于爬虫收录，默认线路还是解析至 Github Pages。


# 0x03 最终方案

于是乎，现已成功更改为上述方案。本来想把 DNS 服务器改为 Cloudfare 的，顺便使用 CDN，不过 Cloudfare 的免费版并不支持国内外分线路解析（需要 Enterprise），因此作罢。


# 参考文献

1. [蒋海云. Coding.net上建立镜像解决Github Pages博客百度无法收录问题[EB/OL]. http://www.atjiang.com/coding.net-pages-as-github-pages-mirror-for-baidu/](http://www.atjiang.com/coding.net-pages-as-github-pages-mirror-for-baidu/)
2. [Pushing code to two remotes[EB/OL]. https://stackoverflow.com/questions/14290113/git-pushing-code-to-two-remote](https://stackoverflow.com/questions/14290113/git-pushing-code-to-two-remotes)
