---
layout:     post
title:      "在 Linux 下使用 QQ"
subtitle:   "最近为了在 Linux 用上 QQ 折腾了好一阵子，特此记录"
date:       2016-11-04 22:53 +0800
author:     "gsfish"
header-img: "img/post-bg-wine.jpg"
tags:
    - Linux
    - 不务正业
---


## 0x00 前言

由于目前腾讯已经停止了对 Linux 版 QQ 的支持，那么我们只好自己动手把小企鹅搬到 Linux 下了。

目前能想到的解决方法有以下两种：

1. 在 VirtualBox 下安装 Windows 和安装增强扩展，再安装 QQ 并使用无缝模式来在 Ubuntu 下使用 QQ
2. 使用 Wine 安装并运行 QQ

个人比较偏向于第二种解决方案，因为为了使用 QQ 而开一个虚拟机太浪费资源了。


## 0x01 使用 VirtualBox

要开启的无缝模式，首先需要在 VirtualBox 里加载一个叫 [Oracle VM VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads) 的增强包（版本要和本机的 VirtualBox 一致，否则会安装失败），然后在 Windows 虚拟机里安装增强功能。

以下是使用效果，出现了迷之边框，无论怎么看都很不顺眼啊：

![01.png](/img/linux-use-qq/01.png)

而且每次点击 QQ 会进入到 Windows 的桌面环境，需要把 Windows 的任务栏缩下去，看来这样实在不是长久之计。


## 0x02 使用 Wine

Wine 是个好东西，根据百度百科，“Wine 不是 Windows 模拟器，而是运用 API 转换技术做出 Linux 对应到 Windows 相对应的函数来调用 DLL 以运行 Windows 程序”。用 Wine 运行 QQ 与原系统共用硬件资源，所以相比虚拟机来说肯定是个更好的选择。

Wine 的使用很简单，比如你下了一个 Windows 应用的安装包，直接在 Linux 下 `wine Setup.exe` 就可以进行安装了。Wine 会在本地创建一个容器，搭建了一个基本的运行环境（一些常用的 DLL），然而默认的 Wine 环境缺少很多 Windows 平台的库，所以很多情况下程序会出一些莫名其妙的问题，这就需要我们自行下载一些 DLL 放到运行环境里了。

这里有一个利器，就是 Winetricks。它内置了一个常用应用列表，Winetricks 会为每一个应用创建一个单独的容器，安装的同时会自动把所需的依赖文件装到 Wine 环境里，大大简化了我们用 Wine 装应用所遇到的麻烦。

不过默认的 Winetricks 里的中文软件太少了，不太符合国情，Github 上有个 [winetricks-zh](https://github.com/hillwoodroc/winetricks-zh) 项目，将 Winetricks 针对国内的环境进行了一些优化。上面有 QQ 的各种版本：

![02.png](/img/linux-use-qq/02.png)

有个比较坑的地方就是，安装的时候发现有一些 `archive.org` 的链接失效了，需要自己翻墙在 Archive 上找更早的副本下载，放到 `~/.cache/winetricks` 对应的文件夹里。

这里有个小插曲。由于我目前用的 Ubuntu 16.04 的源里 Wine 最新的版本号是 1.6，而现在 Wine 现在已经更新到 1.9 了，用 `winetricks-zh` 装 QQ 的时候遇到了一些 BUG 导致安装失败。最终添加了 Wine 官方的 PPA 仓库后把 Wine 卸载并安装了另一个版本才得以解决。

![03.png](/img/linux-use-qq/03.png)
*变相秀一把桌面~*

目前运行在 Wine 下的 QQ 还没崩过，Wine 环境下的 `我的文档` 能与家目录下的 `Documents` 形成映射，而且消息还能收进通知栏，使用体验良好。
