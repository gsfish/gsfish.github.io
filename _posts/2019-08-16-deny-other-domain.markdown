---
typora-root-url: ../img
layout:     post
title:      "防范恶意域名解析"
subtitle:   ""
date:       2019-08-16 18:00 +0800
author:     "gsfish"
header-img: "img/post-bg-07.jpg"
tags:
    - 运维安全
---


主要目的是通过配置 Web 服务器的方式，防范通过其他域名指向我们的服务器并访问的情况。

## 0x00 漏洞危害

1. 若该域名原本指向非法网站，重新指向我们的服务器并被工信部扫描到的话，会影响该服务器的域名备案
2. 会引发搜索引擎惩罚，连带 IP 受到牵连
3. 会导致流量被劫持到别的域名，从而遭到广告联盟的封杀


## 0x01 Nginx 配置

```
if ($host !~ ^(www.example.com)$ ) {
    return 403;
}
```


## 0x02 Apache 配置

```
<VirtualHost *:80>
    ServerName default
    <Location />
        Order Allow,Deny
        Deny from all
    </Location>
</VirtualHost>

<VirtualHost *:80>
    ServerName www.example.com
    DocumentRoot "/home/www"
</VirtualHost>
```


## 0x03 关于 Apache 配置的说明

之所以要为 Apache 的配置附加单独的说明，是因为感觉其配置命令没有 Nginx 那么直观，且存在一些隐式的内部机制，若没有阅读过官方文档的话容易踩坑。

关于 Apache 的配置可以引申出三个问题：

1. 为什么要使用两个 `VirtualHost`？
2. 为什么要将用于拦截的 `VirtualHost` 放在第一个位置？
3. 为什么第一个 `VirtualHost` 的 `ServerName` 为 `default`？

### 1 为什么要使用两个 `VirtualHost`

需要使用一个 `VirtualHost` 处理正常域名的访问请求，另一个 `VirtualHost` 拦截其他域名（`Host` 字段非 Apache 指定的 `ServerName`）的访问请求。


### 2 为什么要将用于拦截的 `VirtualHost` 放在第一个位置

Apache 配置示例的官方文档 [在此](https://httpd.apache.org/docs/current/vhosts/examples.html#purename)，其中关于 `VirtualHost` 处理顺序的描述如下：

> Due to the fact that the virtual host with ServerName www.example.com is first in the configuration file, it has the highest priority and can be seen as the default or primary server. That means that if a request is received that does not match one of the specified ServerName directives, it will be served by this first <VirtualHost>.

也就是说，Apache 对于 `VirtualHost` 的处理存在一个 fallback 机制：Apache 按先后顺序处理 `VirtualHost`，若所有的 `ServerName` 都没被匹配，则使用第一个 `VirtualHost` 中的处理逻辑响应请求（会忽略其中的匹配条件）。因此将 `Deny` 的 `VirtualHost` 放在第一个位置可以实现 Apache 中的域名白名单机制。


### 3 为什么第一个 `VirtualHost` 的 `ServerName` 为 `default`

由上一小节可知，当所有的 `ServerName` 都没被匹配时，会使用第一个 `VirtualHost` 中的处理逻辑响应请求，并忽略其中的匹配条件。因此当发生 fallback 时，无论第一个 `VirtualHost` 中的 `ServerName` 为何值都会被忽略，Apache 只关注其中的处理逻辑。

另外，第一个 `VirtualHost` 的 `ServerName` 也是必须要指定的，否则 Apache 会直接匹配到第一个 `VirtualHost` 并拦截所有域名的访问请求。


## 参考资料

1. [大东东东. 防止独立IP被其它恶意域名恶意解析[EB/OL]. https://www.cnblogs.com/dadonggg/p/8398112.html](https://www.cnblogs.com/dadonggg/p/8398112.html)
2. [VirtualHost Examples - Apache HTTP Server Version 2.4[EB/OL]. https://httpd.apache.org/docs/current/vhosts/examples.html](https://httpd.apache.org/docs/current/vhosts/examples.html)
