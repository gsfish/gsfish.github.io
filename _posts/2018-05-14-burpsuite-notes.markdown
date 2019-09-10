---
layout:     post
title:      "Burp Suite 学习总结"
subtitle:   "此物实乃安全测试一大利器"
date:       2018-05-14 15:00 +0800
author:     "gsfish"
header-img: "img/post-bg-hacker.jpg"
tags:
    - Web安全
    - 渗透
---


> 最近在重新学习使用 Burp Suite，此物确实乃 Web 安全测试的一大利器。总结了一些较有用设置选项，可用于 SSL 抓包等。

# 0x00 Burp Suite 工作流程

![01.png](/img/burpsuite-summary/01.png)

# 0x01 设置选项

## Proxy 标签页

参考官方文档 [Burp Proxy Options](https://portswigger.net/burp/help/proxy_options)

### Proxy Listeners

#### 1 Binding

IP 和端口绑定设置，绑定 IP 地址分仅本地回路、所有接口、指定地址三种模式。

#### 2 Request handling

![02.png](/img/burpsuite-summary/02.png)

`Redict to host`：

将 `request` 转发到指定主机。若该主机与 Host 不同可添加一条 `Match and Replace` 配置。

`Force use of SSL`：

强制使用 SSL。适用于开启了 HTST 的站点（在 `Response Modification` 中对 HTTPS 进行了降级，或使用了 ssltrip）。

`Support invisiable proxying`：

启用透明代理，详情参考官方文档 [Burp Proxy: Invisible Proxying](https://portswigger.net/burp/help/proxy_options_invisible)。适用于 non-proxy-aware 客户端（通过改本地 `hosts` 文件，或 DLL 劫持等方式进行的代理）。

默认从请求的 `Host` 字段中识别目标主机并转发，若使用了改 `hosts` 的方式进行代理，则需在 `Project options - Connections - Hostname Resolution` 中添加该域名的正确记录；若无法识别 `Host` 字段，则需在 `Listioners` 中设置上述的 `Redict to host`。

对于 SSL 流量，若客户端会对服务端进行验证，则会使用该 hostname 生成证书；若不进行认证，则使用自签名证书。若设置了 `Redict to host`，则可在该 `Listener` 设置对应 hostname 的 CA-signed 证书；若代理的请求中含有多个目标域名，则可设置多个虚拟网卡与 `Listener` 并分别生成证书。

#### 3 Certificate

![03.png](/img/burpsuite-summary/03.png)

`self-signed certificate`：
CA 的根证书、使用 `openssl` 生成的证书为自签名证书，客户端不信任（内置 CA 除外）。

`CA-signed certificate`：

由 CA 颁发的证书为签名证书。

`CA-signed certificate with specific hostname`：

适用于使用了 `invisiable proxying` 或 `Redict to host` 的 `Listener`。

`custom certificate`：

官网有使用 `openssl` 生成该证书教程 [Creating a Custom CA Certificate](https://portswigger.net/burp/help/proxy_options#listeners_creatingcert)

#### 4 SSL Pass Through

对于列表中所设置的 `域名` / `IP` / `端口` 范围内的 SSL 流量不进行拦截。适用于 SSL 报错影响正常连接、无法消除的场景，例如移动端 APP 产生的请求。

`automatically add entries on client SSL negotiation failure`：

若 SSL 连接建立失败，则自动将该目标加入到列表中（下次忽略）。

## Intruder 标签页

Intruder 在原始请求数据的基础上，通过修改各种请求参数，以获取不同的请求应答。每一次请求中，Intruder 通常会携带一个或多个 Payload，在不同的位置进行攻击重放，通过应答数据的比对分析来获得需要的特征数据。

### Positions

#### 1 Attack type

`Sniper`：

它使用一组 Payload 集合，依次替换 Payload 位置上（一次攻击只能使用一个 Payload 位置）被 `§` 标志的文本（而没有被 `§` 标志的文本将不受影响），对服务器端进行请求，通常用于测试请求参数是否存在漏洞。

`Battering ram`：

它使用单一的 Payload 集合，依次替换 Payload 位置上被 `§` 标志的文本（而没有被 `§` 标志的文本将不受影响），对服务器端进行请求，与 `Sniper` 模式的区别在于，如果有多个参数且都为 Payload 位置标志时，使用的 Payload 值是相同的，而 `Sniper` 模式只能使用一个 Payload 位置标志。

`Pitchfork`：

它可以使用多组 Payload 集合，在每一个不同的 Payload 标志位置上（最多 20 个），遍历所有的 Payload。举例来说，如果有两个 Payload 标志位置，第一个 Payload 值为 `A` 和 `B`，第二个 Payload 值为 `C` 和 `D`，则发起攻击时，将共发起两次攻击，第一次使用的 Payload 分别为 `A` 和`C`，第二次使用的 Payload 分别为 `B` 和`D`。

`Cluster bomb`：

它可以使用多组 Payload 集合，在每一个不同的 Payload 标志位置上（最多 20 个），依次遍历所有的 Payload。它与 `Pitchfork` 模式的主要区别在于，所发送请求中的 Payload 组合为各不同位置 Payload 值的笛卡尔乘积。举例来说，如果有两个 Payload 标志位置，第一个 Payload 值为 `A` 和 `B`，第二个 Payload 值为 `C` 和 `D`，则发起攻击时，将共发起四次攻击，第一次使用的 Payload 分别为 `A` 和 `C`，第二次使用的 Payload 分别为 `A` 和 `D`，第三次使用的 Payload 分别为 `B` 和 `C`，第四次使用的 Payload 分别为 `B` 和 `D`。

### Payloads

#### 1 Payloads type

![04.png](/img/burpsuite-summary/04.png)

`Simple list`：

最简单的 Payload 类型，通过配置一个字符串列表作为 Payload，也可以手工添加字符串列表或从文件加载字符串列表。在 `Add from list...` 中可以添加内置的 Payload 列表，包括 XSS、SQL注入、数字、大小写字母、用户名、密码和目录名等等。

`Runtime file`：

指定文件，作为相对应 Payload 位置上的 Payload 列表。

`Custom iterator`：

一共可指定 8 个 position，每个 position 可以指定 `Simple list` 类型的 Payload，然后所有指定过的 position 进行笛卡尔积，生成最终的 Payload 列表。

`Character substitution`：

对预定义的字符串进行替换后生成新的 Payload。

`Case modification`：

对预定义的字符串，按照大小写规则，进行替换。

`Recursive grep`：

适用于所发送的 Payload 需要提供部分服务端响应的内容（如 CSRF token 等）的场景。对响应内容的提取规则可在 `Options` -> `Grep - Extract` 中设置。

`Illegal Unicode`：

用于基于原始字符串，并由多种规则生成不合法的 Unicode Payloads 绕过基于字符匹配的防御机制。

`Character blocks`：

用于缓冲区溢出或边界测试。将根据设置的长度、增长步长产生不同大小的字符块 Payload。

`Number`、`Dates`：

顾名思义，生成数字、日期形式的 Payload。

`Brute forcer`：

使用预定字符，基于长度设置生成其全排列作为 Payload。

`Null payloads`：

不产生任何 Payload，`Intruder` 将重放指定次数请求。可用于维持 Session 或对具有竞争条件漏洞的应用进行攻击。

`Character frobber`：

依次修改指定字符串在每个字符位置的值，每次都是在原字符上递增一个该字符的 ASCII 码。可用于测试请求令牌中的不同字符对应用功能的影响。

`Bit flipper`：

对预设的 Payload 原始值，按照比特位，依次进行修改。

`Username generator`：

用于基于指定规则，自动生成用户名和 Email 帐号。

`ECB block shuffler`：

用于测试基于 ECB 加密模式的请求数据，通过改变分组数据的位置方式来验证应用程序是否易受到攻击。

`Extension-generated`：

基于 Burp 插件来生成 Payload，因此使用前必须安装配置 Burp 插件。在插件中注册一个Intruder payload 生成器，供此处调用。

`Copy other payload`：

将其他位置的参数复制到当前 Payload 位置上，作为新的 Payload 值，通常适用于多个参数的请求消息中。

## 其他

更多的设置选项在 [这篇文档](https://www.gitbook.com/book/t0data/burpsuite) 中有很详细的记载。不过部分内容感觉是机翻的，阅读起来有些问题，还是建议参考官方文档。


# 参考文献

1. [t0data. Burp Suite 实战指南[EB/OL]. https://www.gitbook.com/book/t0data/burpsuite](https://www.gitbook.com/book/t0data/burpsuite)
2. [Burp Suite Documentation[EB/OL]. https://portswigger.net/burp/help/contents](https://portswigger.net/burp/help/contents)
