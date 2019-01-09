---
layout:     post
title:      "iptables 学习笔记"
subtitle:   ""
date:       2019-01-09 22:00 +0800
author:     "gsfish"
header-img: "img/post-bg-07.jpg"
tags:
    - Linux
---


以往面对 iptables 的场景比较少，对其的了解仅存在于命令使用的层面。现在重新开始学习 iptables，终于弄懂了以前很模糊的一些概念，总结了以下的笔记。

## 0x00 关于 iptables

iptables 是一个配置 Linux 内核防火墙的命令行工具，可以检测、修改、转发、重定向和丢弃网络数据包。iptables 只是用户空间的一个客户端代理，负责实际数据包处理的则是内核空间的 Netfilter 模块。

## 0x01 表链关系

表由一系列的链组成，链由一组按顺序排列的规则组成。每一条规则包含一个匹配条件和相应的动作，若数据包符合匹配条件，则该动作会被执行。一个数据包从链路到达本地网络接口，再到相应的应用程序，最后通过网络接口返回到链路的过程中，数据包的流向如下：

![](/img/iptables-notes/tables_traverse.jpg)

从功能的角度来说，iptables 的规则可以按表进行分类。每个表保存负责同一功能的一系列规则。

1. `raw`：用于配置数据包（关闭连接追踪）
2. `mangle`：用于修改数据包（TOS、TTL、MARK）
3. `nat`：用于网络地址转换（SNAT、DNAT、端口转发）
4. `filter`：用于过滤数据包（实现软件防火墙）
5. `security`：用于强制访问控制（依赖 SELinux）

从时间顺序的角度来说，iptables 的规则可以按链进行分类。当数据包抵达一个链时，Netfilter 按照 `raw -> mangle -> nat -> filter -> security` 的顺序来匹配表（无该表则忽略），并遍历匹配表中的每一条规则。

根据数据包的目的地址，若是发往本地的包：

1. `PREROUTING`
2. `INPUT`
3. 本地应用程序
4. `OUTPUT`
5. `POSTROUTING`

若不是发往本地的包：

1. `PREROUTING`
2. `FORWARD`
3. `POSTROUTING`

# 参考资料

1. [[EB/OL]. ](https://www.frozentux.net/iptables-tutorial/cn/iptables-tutorial-cn-1.1.19.html)
2. [[EB/OL]. ](http://www.zsythink.net/archives/1199)
3. [[EB/OL]. ](https://wiki.archlinux.org/index.php/Iptables_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6)

