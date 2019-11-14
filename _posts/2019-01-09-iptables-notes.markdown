---
typora-root-url: ../img
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

# 0x00 关于 iptables

iptables 是一个配置 Linux 内核防火墙的命令行工具，可以检测、修改、转发、重定向和丢弃网络数据包。iptables 只是用户空间的一个客户端代理，负责实际数据包处理的则是内核空间的 Netfilter 模块。

# 0x01 表链关系

表由一系列的链组成，链由一组按顺序排列的规则组成。每一条规则包含一个匹配条件和相应的动作，若数据包符合匹配条件，则该动作会被执行。一个数据包从链路到达本地网络接口（若 MAC 地址符合则会由内核中相应的驱动程序接收），再经路由判断（IP 是否符合）送达相应的应用程序，最后通过网络接口返回到链路的过程中，数据包的流向如下：

![](/img/iptables-notes/tables_traverse.jpg)

如上图所示，一个表中可以包含几个链，同时一个链中也可以出现几个表。可以分别从表和链的角度，来看待二者的对应关系。

## 以表分类

从功能的角度来说，iptables 的规则可以按表进行分类。每个表保存负责同一功能的一系列规则。所有的表如下：

1. `raw`：用于配置数据包（关闭连接追踪）
2. `mangle`：用于修改数据包（TOS、TTL、MARK）
3. `nat`：用于网络地址转换（SNAT、DNAT、端口转发）
4. `filter`：用于过滤数据包（实现软件防火墙）
5. `security`：用于强制访问控制（依赖 SELinux）

## 以链分类

从时间顺序的角度来说，iptables 的规则可以按链进行分类。当数据包抵达一个链时，Netfilter 按照 `raw -> mangle -> nat -> filter -> security` 的顺序来匹配表（无该表则忽略），并遍历匹配表中的每一条规则。所有的链如下：

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

## 常用表的功能

`mangle` 表：

1. TOS：设置或改变数据包的服务类型域（网络上的数据包如何被路由等策略）
2. TTL：改变数据包的生存时间域
3. MARK：给包设置特殊的标记

`nat` 表：

1. DNAT：改变包的目的地址，使包能重路由到某台主机
2. SNAT：改变包的源地址，使多个主机公用一个公网 IP
3. MASQUERADE：// TODO

`filter` 表：

用来过滤数据包，可根据包的内容对包做 DROP 或 ACCEPT。

# 0x02 状态机制

本地产生的包的跟踪由 `OUTPUT` 链处理，其他所有连接跟踪都由 `PREROUTING` 链处理（iptables 会在 `PREROUTING` 链里重新计算所有状态）：

1. 若会话的第一个数据包由本地应用程序发出，状态就会在 `OUTPUT` 链里被设置为 `NEW`；当我们收到回应的包时，状态就会在 `PREROUTING` 链里被设置为 `ESTABLISHED`。
2. 若会话的第一个数据包不是由本地应用程序发出，状态就会在 `PREROUTING` 链里被设置为 `NEW`。

## 数据包在用户空间的状态

1. `NEW`：还未建立好的连接（刚发出第一个包）
2. `ESTABLISHED`：已建立的连接（已在内核里注册过 `NEW` 状态）
3. `RELATED`：已经存在的、处于已建立状态的连接生成的新连接
4. `INVALID`：说明数据包不能被识别属于哪个连接或没有任何状态

TCP 连接在建立过程中的状态变化：

![](/img/iptables-notes/state-tcp-connecting.jpg)

TCP 连接在关闭过程中的状态变化：

![](/img/iptables-notes/state-tcp-closing.jpg)

## 数据包在内核空间的状态

可通过 `/proc/net/nf_conntrack` 查看内核保存在内存中的状态信息。

1. `NONE`
2. `ESTABLISHED`
3. `SYN_SENT`
4. `SYN_RECV`
5. `FIN_WAIT`
6. `TIME_WAIT`
7. `CLOSE`
8. `CLOSE_WAIT`
9. `LAST_ACK`
10. `LISTEN`

# 0x03 规则

规则是决定如何处理一个数据包的语句。如果一个包符合规则中的 **匹配条件**，我们就运行 **target/jump** 指令。

## 匹配条件

匹配条件可分为以下 3 种，不同类型的匹配条件包含了不同的参数：

1. 通用匹配：可直接使用而不需要前提条件
2. 隐含匹配：跟随匹配协议自动装载入内核（自动匹配该协议的包的一些特点）
3. 显式匹配：必须用 `-m/--match` 装载（扩展匹配）

## Targets/Jumps

target/jump 决定符合条件的包到何处去。

### jump 的目标是一个在同一个表内的链

我们在 `filter` 表中建一个名为 `tcp_packets` 的链，然后再把它作为 jump 的目标：

```
iptables -N tcp_packets
iptables -A INPUT -p tcp -j tcp_packets
```

这样当数据包到达 `INPUT` 链后会跳入 `tcp_packets` 链，开始在子链中遍历。如果未被子链中的任何规则匹配（到达了链的结尾），则会退到 `INPUT` 链的下一条规则继续向后匹配；如果在子链中被 `ACCEPT` 了，也就相当于在父链中被 `ACCEPT` 了，那么它不会再经过父链中的其他规则。

### target 的目标是具体的操作

target 可分为两类，一类操作完后会停止匹配（`ACCEPT`、`DROP`、`REJECT`），另一类对包操作完后可继续对链中的其他规则进行匹配（`LOG`、`ULOG`、`TOS`）。

target 有以下几种：

1. `ACCEPT`
2. `DNAT`
3. `DROP`
4. `LOG`
5. `MARK`
6. `MASQUERADE`
7. `MIRROR`
8. `QUEUE`
9. `REDIRECT`
10. `REJECT`
11. `RETURN`
12. `SNAT`
13. `TOS`
14. `TTL`
15. `ULOG`


# 参考资料

1. [Oskar Andreasson. Iptables 指南 1.1.19[EB/OL]. https://www.frozentux.net/iptables-tutorial/cn/iptables-tutorial-cn-1.1.19.html](https://www.frozentux.net/iptables-tutorial/cn/iptables-tutorial-cn-1.1.19.html)
2. [朱双印. iptables详解：图文并茂理解iptables[EB/OL]. http://www.zsythink.net/archives/1199](http://www.zsythink.net/archives/1199)
3. [Iptables (简体中文) - ArchWiki[EB/OL]. https://wiki.archlinux.org/index.php/Iptables_(简体中文)](https://wiki.archlinux.org/index.php/Iptables_(简体中文))
