---
layout:     post
title:      "使用 VirtualBox + VHD 管理多系统"
subtitle:   "实体机多系统新姿势"
date:       2017-03-30 21:19 +0800
author:     "gsfish"
header-img: "img/post-bg-07.jpg"
tags:
    - VHD
---


经常用 VMware 运行虚拟机，一方面是因为有对于其他环境的需求，另一方面是因为虚拟机相对来说较为安全（虽然也有利用漏洞从虚拟机中逃逸的案例，例如前不久的 Pwn2Own 2017），并且快照的功能十分方便。然而，虚拟机在性能、硬件支持等方面也有其弊端。

最近了解到一个在 Windows 中潜藏已久的功能，即创建 VHD 虚拟硬盘。同时 VirtualBox 也支持创建管理 VHD 硬盘，因此便萌生了使用 VirtualBox 在 VHD 中安装系统，并在实体机的启动引导中添加 VHD 的启动项。如此一来便可以用 Windows 原生的启动程序来选择进入的系统了，并且该系统仅存在于 VHD 中，添加方便，删除不留痕迹，还能用 VirtualBox 的快照随时还原，岂不美哉。


# 0x00 VHD 概述

VHD 是 Microsoft Virtual Hard Disk format（微软虚拟磁盘文件）的简称。就是能够把一个 VHD 文件虚拟成一个硬盘的技术，在其中可以如在真实硬盘中一样操作：读取、写入、创建分区、格式化。实际测量之后会发现，虚拟硬盘的读取速度和实体硬盘的一模一样。


# 0x01 安装过程

主要有两种方式：

1. 通过 VirtualBox 创建使用 VHD 硬盘的虚拟机并安装系统
2. 手动创建 VHD 硬盘并使用 ImageX 将系统从镜像中写入 VHD（仅支持 Windows）

使用第一种方式的好处在于可以利用 VirtualBox 的快照功能进行还原。

## 使用 VirtualBox

打开 VirtualBox。点击 `新建 > 专家模式`，设置好名称、内存等选项。

在虚拟硬盘处选择 `现在创建虚拟硬盘 > 创建`，设置合适的文件大小：

* `动态分配` VHD 所占空间会根据所写入的数据量动态伸缩，有利于节省空间
* `固定大小` 创建 VHD 文件时将分配所有容量，使用效率相对来说会高一些

![01.png](/img/vhd-multi-os/01.png)

创建完毕后，点击 `设置 > 存储`，在光驱处选择系统镜像文件并保存，然后点击 `启动`。

在 VirtualBox 安装系统，待系统安装完毕将其关闭。

右键 `此电脑 > 管理 > 存储`，右键 `磁盘管理 > 附加 VHD`，选择刚才在 VirtualBox 中创建的 VHD 文件：

![02.png](/img/vhd-multi-os/02.png)

确定后下方会出现我们刚才附加的 VHD 虚拟硬盘，此处的 F 盘即为系统的所在位置：

![03.png](/img/vhd-multi-os/03.png)

接下来以管理员身份运行 `cmd`，输入以下命令将系统添加至启动项即可：

```
bcdboot F:\windows

说明：
F 为 VHD 中系统所在分区的驱动器号
```

最后，在 `msconfig` 中可以确认该系统已成功添加为启动项：

![04.png](/img/vhd-multi-os/04.png)

## 使用 ImageX

右键 `此电脑 > 管理 > 存储` ，右键 `磁盘管理 > 创建 VHD`：

![05.png](/img/vhd-multi-os/05.png)

此时会显示创建的磁盘还未初始化：

![06.png](/img/vhd-multi-os/06.png)

右键该磁盘，点击 `初始化该磁盘` ，确定即可（分区形式取决于所装系统是否支持）：

![07.png](/img/vhd-multi-os/07.png)

接下来需要对该磁盘进行分区。右键单击空白处，点击 `新建简单卷` ，然后一直 `下一步` 即可。

挂载或解压系统镜像。可以用 DAEMON Tools 挂载镜像，也可以用 Winrar 将镜像中的文件解压出来（比较耗时），以下采用挂载镜像的方式。

使用 ImageX 将 `sources\install.wim` 映像文件中的系统提取至所创建的分区中（该工具非系统自带，需自行下载）。

先使用以下命令查看系统映像里所包含的系统版本。

```
imagex /info F:\sources\install.wim

说明：
F 为挂载光驱的驱动器号  
IMAGE INDEX 为系统版本的索引号  
```

再通过以下命令把安装文件释放到 VHD 磁盘（需以管理员身份运行）。

```
imagex /apply F:\sources\install.wim 1 E:\

说明：
F:\sources\install.wim 为映像文件所在路径
1 为所选系统的索引号
E:\ 为 VHD 中系统所在分区的驱动器号
```

写入完成后，以管理员身份运行 `cmd`，输入以下命令将系统添加至启动项即可。

```
bcdboot E:\Windows
```


# 0x02 卸载方法

十分方便，两步即可完成：

1. 打开 `msconfig`，在 `引导` 中将系统的启动项删除
2. 删除 VHD 文件


# 0x03 进阶玩法

参考此帖：[http://tieba.baidu.com/p/3328948367](http://tieba.baidu.com/p/3328948367)

可以总结为以下两种用法：

1. 使用 `diskpart` 创建差分硬盘，实现秒备份、秒恢复
2. 实现基于差分硬盘的多系统，彼此隔离的同时节省空间
