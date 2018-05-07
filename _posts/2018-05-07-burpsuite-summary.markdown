---
layout:     post
title:      "Burp Suite 学习总结"
subtitle:   ""
date:       2018-05-07 22:00 +0800
author:     "gsfish"
header-img: "img/post-bg-hacker.jpg"
tags:
    - 渗透
---

最近在重新学习使用 Burp Suite，此物确实是 Web 安全测试的一大利器。总结了一些较有用设置选项，可用于 SSL 抓包等。

# Burp Suite 工作流程

![01.png](/img/burpsuite-summary/01.png)

# Proxy

参考官方文档 [Burp Proxy Options](https://portswigger.net/burp/help/proxy_options)

## Proxy Listeners

### Binding

IP 和端口绑定设置，绑定 IP 地址分仅本地回路、所有接口、指定地址三种模式。

### Request handling

![02.png](/img/burpsuite-summary/02.png)

`Redict to host`：

将 `request` 转发到指定主机。若该主机与 Host 不同可添加一条 `Match and Replace` 配置。

`Force use of SSL`：

强制使用 SSL。适用于开启了 HTST 的站点（在 `Response Modification` 中对 HTTPS 进行了降级，或使用了 ssltrip）。

`Support invisiable proxying`：

启用透明代理，详情参考官方文档 [Burp Proxy: Invisible Proxying](https://portswigger.net/burp/help/proxy_options_invisible)。适用于 non-proxy-aware 客户端（通过改本地 `hosts` 文件，或 DLL 劫持等方式进行代理）。

默认从请求的 `Host` 字段中识别目标主机并转发，若使用了改 `hosts` 的方式进行代理，则需在 `Project options - Connections - Hostname Resolution` 中添加该域名的正确记录；若无法识别 `Host` 字段，则需在 `Listioners` 中设置上述的 `Redict to host`。

对于 SSL 流量，若客户端会对服务端进行验证，则会使用该 hostname 生成证书；若不进行认证，则使用自签名证书。若设置了 `Redict to host`，则可在该 `Listener` 设置对应 hostname 的 CA-signed 证书；若代理的请求中含有多个目标域名，则可设置多个虚拟网卡与 `Listener` 并分别生成证书。

### Certificate

![03.png](/img/burpsuite-summary/03.png)

`self-signed certificate`：
CA 的根证书、使用 `openssl` 生成的证书为自签名证书，客户端不信任（内置 CA 除外）。

`CA-signed certificate`：

由 CA 颁发的证书为签名证书。

`CA-signed certificate with specific hostname`：

适用于使用了 `invisiable proxying` 或 `Redict to host` 的 `Listener`。

`custom certificate`：

官网有使用 `openssl` 生成该证书教程 [Creating a Custom CA Certificate](https://portswigger.net/burp/help/proxy_options#listeners_creatingcert)

### SSL Pass Through

对于列表中所设置的 `域名` / `IP` / `端口` 范围内的 SSL 流量不进行拦截。适用于 SSL 报错影响正常连接、无法消除的场景，例如移动端 APP 产生的请求。

`automatically add entries on client SSL negotiation failure`：

若 SSL 连接建立失败，则自动将该目标加入到列表中（下次忽略）。


# 其他

更多的设置选项在 [这篇文档](https://www.gitbook.com/book/t0data/burpsuite) 中有很详细的记载。不过部分内容感觉是机翻的，阅读起来有些问题，还是建议参考官方文档。


---

> 参考资料：  
> [Burp Suite 实战指南](https://www.gitbook.com/book/t0data/burpsuite)  
> [Burp Suite Documentation - Contents](https://portswigger.net/burp/help/contents)  
